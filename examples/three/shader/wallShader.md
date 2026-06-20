---
title: "Three.js 扩散圆墙教程"
description: "详解 Three.js 扩散圆墙：基于 WebGL 实现「扩散圆墙」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,扩散圆墙,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 扩散圆墙 · *Wall Shader* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=wallShader)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![扩散圆墙](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/wallShader.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **扩散圆墙** 效果：基于 WebGL 实现「扩散圆墙」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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

const box = document.getElementById('box')
const scene = new THREE.Scene()
const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)
camera.position.set(0, 10, 10)
const renderer = new THREE.WebGLRenderer({ antialias: true })
renderer.setSize(box.clientWidth, box.clientHeight)
box.appendChild(renderer.domElement)
const controls = new OrbitControls(camera, renderer.domElement)
controls.enableDamping = true
controls.dampingFactor = 0.25

window.onresize = () => {
    renderer.setSize(box.clientWidth, box.clientHeight)
    camera.aspect = box.clientWidth / box.clientHeight
    camera.updateProjectionMatrix()
}

const curve = new THREE.LineCurve3(new THREE.Vector3(), new THREE.Vector3().setY(3))
const geometry = new THREE.TubeGeometry(curve, 20, 5, 300, false);

geometry.computeBoundingBox()
const { max, min } = geometry.boundingBox

// 创建材质
const material = new THREE.ShaderMaterial({
    transparent: true,
    side: THREE.DoubleSide,
    uniforms: {
        uMax: { value: max },
        uMin: { value: min },
        uColor: { value: new THREE.Color('#409eff') }
    },
    vertexShader: `
      varying vec4 vPosition;
      void main() {
        vPosition = modelMatrix * vec4(position,1.0);
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
      }
    `,
    fragmentShader: `
      uniform vec3 uColor; // 半径        
      uniform vec3 uMax; 
      uniform vec3 uMin;
      uniform mat4 modelMatrix; // 世界矩阵
      varying vec4 vPosition; // 接收顶点着色传递进来的位置数据
      void main() {
        vec4 uMax_world = modelMatrix * vec4(uMax,1.0);
        vec4 uMin_world = modelMatrix * vec4(uMin,1.0);
        float opacity =1.0 - (vPosition.y - uMin_world.y) / (uMax_world.y -uMin_world.y); 
        gl_FragColor = vec4( uColor, opacity);
      }
    `
})

const mesh = new THREE.Mesh(geometry, material)
scene.add(mesh)

let time = 0
animate()

function animate() {
    if (time >= 1) time = 0
    else {
        time += 0.01
        mesh.scale.set(time, 1, time)
    }

    requestAnimationFrame(animate)
    controls.update()
    renderer.render(scene, camera)
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/wallShader.js)

## 小结

- 本文提供 **扩散圆墙** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

