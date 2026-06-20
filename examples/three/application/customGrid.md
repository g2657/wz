---
title: "Three.js 自定义网格教程"
description: "详解 Three.js 自定义网格：基于 WebGL 实现「自定义网格」可视化效果，附完整可运行源码，涵盖 OrbitControls、BufferGeometry 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,自定义网格,WebGL,源码,教程,在线案例,OrbitControls,相机控制,BufferGeometry"
outline: deep
---

### 自定义网格 · *Custom Grid* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=customGrid)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![自定义网格](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/customGrid.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- BufferGeometry 自定义顶点/索引数据
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **自定义网格** 效果：基于 WebGL 实现「自定义网格」可视化效果，附完整可运行源码；核心用到 OrbitControls、BufferGeometry。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- 阅读下方完整源码时，建议从 `init` / `load` / `animate` 三条主线入手，再深入 shader 与工具函数。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
3. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(50, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 0, 16)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setClearColor(0xffffff, 1)

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

animate()

function animate() {

    requestAnimationFrame(animate)

    controls.update()

    renderer.render(scene, camera)

}

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

}

const width = 10, height = 10 // 宽度和高度

const dx = 1, dy = 1 // 网格间隔

const colorA = new THREE.Color(0xaaaaaa), colorB = new THREE.Color(0xdfdfdf) // 网格颜色

const vertices = [], colors = [] // 顶点和颜色

// 生成网格顶点和颜色
for (let i = -width / 2; i <= width / 2; i += dx) {

    vertices.push(i, -height / 2, 0, i, height / 2, 0) // 水平线

    const color = (i === 0) ? colorA : colorB // 颜色

    colors.push(...Array(2).fill(color.toArray()).flat())

}

// 垂直线
for (let j = -height / 2; j <= height / 2; j += dy) {

    vertices.push(-width / 2, j, 0, width / 2, j, 0) // 垂直线

    const color = (j === 0) ? colorA : colorB // 颜色

    colors.push(...Array(2).fill(color.toArray()).flat())
    
}

// 创建 BufferGeometry 和材质
const geometry = new THREE.BufferGeometry().setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3))

geometry.setAttribute('color', new THREE.Float32BufferAttribute(colors, 3))

const material = new THREE.LineBasicMaterial({ vertexColors: true })

const grid = new THREE.LineSegments(geometry, material)

scene.add(grid)
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/customGrid.js)

## 小结

- 本文提供 **自定义网格** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

