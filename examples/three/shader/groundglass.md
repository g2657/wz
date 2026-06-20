---
title: "Three.js 毛玻璃教程"
description: "详解 Three.js 毛玻璃：基于 WebGL 实现「毛玻璃」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,毛玻璃,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 毛玻璃 · *Groundglass* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=groundglass)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![毛玻璃](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/groundglass.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **毛玻璃** 效果：基于 WebGL 实现「毛玻璃」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
import * as dat from 'dat.gui'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 0, 1)

const renderer = new THREE.WebGLRenderer()

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

animate()

function animate() {

    requestAnimationFrame(animate)

    renderer.render(scene, camera)

}

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

}

const vertexShader = ` 
    varying vec2 vUv;
    void main(){
    vUv = uv;
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}`

const fragmentShader = `
    varying vec2 vUv;
    uniform sampler2D tDiffuse;
    uniform vec2 vScreenSize;
    uniform float fRadius;

    float rand(vec2 co) {
    float a = fract(dot(co, vec2(2.067390879775102, 12.451168662908249))) - 0.5;
    float s = a * (6.182785114200511 + a * a * (-38.026512460676566 + a * a * 53.392573080032137));
    float t = fract(s * 43758.5453);
    return t;
    }

    void main() {
    float x = (vUv.x * vScreenSize.x) + rand(vUv) * fRadius * 2.0 - fRadius;
    float y = (vUv.y * vScreenSize.y) + rand(vec2(vUv.y, vUv.x)) * fRadius * 2.0 - fRadius;
    vec4 textureColor = texture2D(tDiffuse, vec2(x, y) / vScreenSize);
    gl_FragColor = textureColor;
}`

const uniforms = {

    tDiffuse: { type: 't', value: new THREE.TextureLoader().load(HOST + 'files/author/flowers-10.jpg') },

    vScreenSize: { type: "v2", value: new THREE.Vector2(window.innerWidth, window.innerHeight) },

    fRadius: { type: "f", value: 20.0 },

}

const material = new THREE.ShaderMaterial({ uniforms, vertexShader, fragmentShader });

const geometry = new THREE.PlaneGeometry();

const mesh = new THREE.Mesh(geometry, material);

scene.add(mesh);

var gui = new dat.GUI();
gui.add(uniforms['fRadius'], 'value', 1.0, 100.0).step(1.0).name('mosaicScale');
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/groundglass.js)

## 小结

- 本文提供 **毛玻璃** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

