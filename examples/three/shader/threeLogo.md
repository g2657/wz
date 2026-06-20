---
title: "Three.js three.js Logo教程"
description: "详解 Three.js three.js Logo：基于 WebGL 实现「three.js Logo」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,three.js Logo,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### three.js Logo · *Three Logo* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=threeLogo)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![three.js Logo](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/threeLogo.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **three.js Logo** 效果：基于 WebGL 实现「three.js Logo」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
import { GUI } from "three/addons/libs/lil-gui.module.min.js"

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(50, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 0, 10)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

const vertexShader = `
varying vec2 vUv;
varying vec3 vPosition;
varying vec3 vNormal;

void main() {
  vUv = uv;
  vPosition = position;
  vNormal = normalize(normalMatrix * normal);
  gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}
`

const fragmentShader = `
uniform float time;
uniform vec3 color1;
uniform vec3 color2;
uniform vec3 color3;

varying vec2 vUv;
varying vec3 vPosition;
varying vec3 vNormal;

void main() {
  // 创建基于位置和时间的渐变效果
  float noise = sin(vPosition.x * 2.0 + time) * 0.25 +
               cos(vPosition.y * 2.0 + time * 0.5) * 0.25 +
               sin(vPosition.z * 2.0 + time * 0.8) * 0.5;
  
  // 边缘发光效果
  float fresnel = pow(1.0 - abs(dot(vNormal, vec3(0.0, 0.0, 1.0))), 2.0);
  
  // 混合颜色
  vec3 baseColor = mix(color1, color2, noise + 0.5);
  vec3 glowColor = mix(baseColor, color3, fresnel);
  
  // 添加脉冲效果
  float pulse = (sin(time * 2.0) + 1.0) * 0.15 + 0.85;
  
  gl_FragColor = vec4(glowColor * pulse, 1.0);
}
`

const material = new THREE.ShaderMaterial({
    uniforms: {
        time: { value: 0 },
        color1: { value: new THREE.Color(0xdbb8ff) },
        color2: { value: new THREE.Color(0x98d0fb) },
        color3: { value: new THREE.Color(0xfdebbf) }
    },
    vertexShader,
    fragmentShader,
    side: THREE.DoubleSide,
    transparent: true
})

const gui = new GUI()
gui.addColor(material.uniforms.color1, 'value').name('Color 1')
gui.addColor(material.uniforms.color2, 'value').name('Color 2')
gui.addColor(material.uniforms.color3, 'value').name('Color 3')

const geometry = createThreeJSLogoGeometry()
const mesh = new THREE.Mesh(geometry, material)
scene.add(mesh)

function createThreeJSLogoGeometry() {
    // 基础变量设置
    const v = new THREE.Vector2(0, 1);
    const c = new THREE.Vector2();
    const s = 0.85; // 缩放比例

    // 创建基本三角形
    const tri = [
        v.clone().multiplyScalar(s).add(new THREE.Vector2(0, -0.25)),
        v.clone().rotateAround(c, (-Math.PI * 2) / 3).multiplyScalar(s).add(new THREE.Vector2(0, -0.25)),
        v.clone().rotateAround(c, (Math.PI * 2) / 3).multiplyScalar(s).add(new THREE.Vector2(0, -0.25))
    ];

    // 创建翻转三角形
    const triFlip = [
        v.clone().rotateAround(c, Math.PI).multiplyScalar(s).sub(new THREE.Vector2(0, -0.25)),
        v.clone().rotateAround(c, Math.PI / 3).multiplyScalar(s).sub(new THREE.Vector2(0, -0.25)),
        v.clone().rotateAround(c, -Math.PI / 3).multiplyScalar(s).sub(new THREE.Vector2(0, -0.25))
    ];

    // 创建三角形孔洞
    const holes = [];
    const hA = 3 / Math.sqrt(3) * 0.5;

    // 生成图案中的三角形孔洞
    for (let row = 0; row < 4; row++) {
        const items = 1 + row * 2;
        const h = 1.5 * (1.5 - row);
        const w = -((items - 1) / 2) * hA;

        for (let i = 0; i < items; i++) {
            const offsetX = w + hA * i;
            const offsetY = h;
            const points = (i % 2 === 0 ? tri : triFlip).map(p => {
                const pt = p.clone();
                pt.x += offsetX;
                pt.y += offsetY;
                return pt;
            });
            holes.push(new THREE.Path(points));
        }
    }

    // 创建外部轮廓
    const contour = [
        v.clone().multiplyScalar(4.1).add(new THREE.Vector2(0, -1)),
        v.clone().rotateAround(c, (Math.PI * 2) / 3).multiplyScalar(4.1).add(new THREE.Vector2(0, -1)),
        v.clone().rotateAround(c, (-Math.PI * 2) / 3).multiplyScalar(4.1).add(new THREE.Vector2(0, -1))
    ];

    // 创建形状并添加孔洞
    const shape = new THREE.Shape(contour);
    shape.holes = holes;

    // 创建挤出几何体
    const geometry = new THREE.ExtrudeGeometry(shape, {
        depth: 0.1,
        bevelEnabled: false
    });

    // 旋转和居中
    geometry.rotateZ(Math.PI * 0.25);
    geometry.center();

    return geometry;
}

animate()

function animate() {

    material.uniforms.time.value += 0.01

    requestAnimationFrame(animate)

    controls.update()

    renderer.render(scene, camera)
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/threeLogo.js)

## 小结

- 本文提供 **three.js Logo** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

