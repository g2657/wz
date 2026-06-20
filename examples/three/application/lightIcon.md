---
title: "Three.js 亮光标记教程"
description: "详解 Three.js 亮光标记：基于 WebGL 实现「亮光标记」可视化效果，附完整可运行源码，涵盖 OrbitControls、GSAP 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,亮光标记,WebGL,源码,教程,在线案例,OrbitControls,相机控制,GSAP,动画"
outline: deep
---

### 亮光标记 · *Light Icon* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=lightIcon)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![亮光标记](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/lightIcon.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- GSAP 时间轴与补间动画
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **亮光标记** 效果：基于 WebGL 实现「亮光标记」可视化效果，附完整可运行源码；核心用到 OrbitControls、GSAP。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- 阅读下方完整源码时，建议从 `init` / `load` / `animate` 三条主线入手，再深入 shader 与工具函数。

## 实现步骤

1. 创建 OrbitControls 并处理 resize
2. mixer.update(delta) 或 gsap.to 驱动属性
3. 搭建灯光与环境（如有）
4. requestAnimationFrame 循环 update + render

## 代码要点

```js
import { Mesh, MeshBasicMaterial, PerspectiveCamera, PlaneGeometry, Scene, TextureLoader, WebGLRenderer } from 'three'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import gsap from 'gsap'

const size = { width: window.innerWidth, height: window.innerHeight }
const scene = new Scene()

const camera = new PerspectiveCamera(45, size.width / size.height, 0.1, 1000)
camera.position.set(30, 30, 30)

const renderer = new WebGLRenderer({ antialias: true })
renderer.setSize(size.width, size.height)
renderer.setPixelRatio(window.devicePixelRatio)
document.body.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

const circlePlane = new PlaneGeometry(6, 6)
const circleTexture = new TextureLoader().load(FILE_HOST + 'images/channels/label.png')
const circleMaterial = new MeshBasicMaterial({
    color: 0xffffff,
    map: circleTexture,
    transparent: true,
    blending: THREE.AdditiveBlending,
    depthWrite: false,
    side: THREE.DoubleSide
})
const circleMesh = new Mesh(circlePlane, circleMaterial)
circleMesh.rotation.x = -Math.PI / 2
gsap.to(circleMesh.scale, {
    duration: 1 + Math.random() * 0.5,
    x: 1.5,
    y: 1.5,
    repeat: -1
})

const lightPillarTexture = new THREE.TextureLoader().load(FILE_HOST + 'images/channels/light_column.png')
const lightPillarGeometry = new THREE.PlaneGeometry(3, 20)
const lightPillarMaterial = new THREE.MeshBasicMaterial({
    color: 0xffffff,
    map: lightPillarTexture,
    alphaMap: lightPillarTexture,
    transparent: true,
    blending: THREE.AdditiveBlending,
    side: THREE.DoubleSide,
    depthWrite: false
})
const lightPillar = new THREE.Mesh(lightPillarGeometry, lightPillarMaterial)
lightPillar.add(lightPillar.clone().rotateY(Math.PI / 2))
circleMesh.position.set(0, -10, 0)

lightPillar.add(circleMesh)
scene.add(lightPillar)

animate()
function animate() {
    requestAnimationFrame(animate)
    renderer.render(scene, camera)
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/lightIcon.js)

## 小结

- 本文提供 **亮光标记** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

