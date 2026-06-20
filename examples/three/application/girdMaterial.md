---
title: "Three.js 网格材质教程"
description: "详解 Three.js 网格材质：基于 WebGL 实现「网格材质」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,网格材质,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 网格材质 · *Gird Material* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=girdMaterial)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![网格材质](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/girdMaterial.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **网格材质** 效果：基于 WebGL 实现「网格材质」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
3. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
4. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GUI } from 'dat.gui'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 100000)

camera.position.set(50, 50, 50)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

scene.add(new THREE.AxesHelper(100), new THREE.GridHelper(100, 10))

// 通用网格材质
const gridMaterial = new THREE.ShaderMaterial({
    transparent: true,
    side: THREE.DoubleSide,
    uniforms: {
        color: { value: new THREE.Color(0x00caea) },
        gridX: { value: 40.0 },
        gridY: { value: 20.0 },
        lineWidth: { value: 0.6 }
    },
    vertexShader: `
        varying vec2 vUv;
        void main() {
            vUv = uv;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
    `,
    fragmentShader: `
        uniform vec3 color;
        uniform float gridX;
        uniform float gridY;
        uniform float lineWidth;
        varying vec2 vUv;
        
        void main() {
            vec2 grid = vec2(gridX, gridY);
            vec2 f = fract(vUv * grid);
            vec2 df = fwidth(vUv * grid);
            vec2 line = smoothstep(df * lineWidth, df * lineWidth * 2.0, f) * 
                        smoothstep(df * lineWidth, df * lineWidth * 2.0, 1.0 - f);
            float alpha = 1.0 - min(line.x, line.y);
            if (alpha < 0.1) discard;
            gl_FragColor = vec4(color, alpha);
        }
    `
})

// 几何体列表
const geometries = {
    'TorusKnot': new THREE.TorusKnotGeometry(15, 4, 100, 16),
    'Cylinder': new THREE.CylinderGeometry(15, 15, 40, 64),
    'Sphere': new THREE.SphereGeometry(20, 64, 32),
    'Torus': new THREE.TorusGeometry(15, 5, 32, 64),
    'Box': new THREE.BoxGeometry(30, 30, 30),
    'Cone': new THREE.ConeGeometry(15, 40, 64),
    'Capsule': new THREE.CapsuleGeometry(10, 20, 16, 32)
}

const mesh = new THREE.Mesh(geometries['TorusKnot'], gridMaterial)
scene.add(mesh)

// GUI 控制
const gui = new GUI()
const params = { geometry: 'TorusKnot', color: '#00ffff' }

gui.add(params, 'geometry', Object.keys(geometries)).name('几何体').onChange(v => mesh.geometry = geometries[v])
gui.add(gridMaterial.uniforms.gridX, 'value', 1, 100).name('横向网格')
gui.add(gridMaterial.uniforms.gridY, 'value', 1, 100).name('纵向网格')
gui.add(gridMaterial.uniforms.lineWidth, 'value', 0.1, 5).name('线条粗细')
gui.addColor(params, 'color').name('颜色').onChange(v => gridMaterial.uniforms.color.value.set(v))

function animate() {
    requestAnimationFrame(animate)
    renderer.render(scene, camera)
}

animate()

window.onresize = () => {
    renderer.setSize(box.clientWidth, box.clientHeight)
    camera.aspect = box.clientWidth / box.clientHeight
    camera.updateProjectionMatrix()
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/girdMaterial.js)

## 小结

- 本文提供 **网格材质** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

