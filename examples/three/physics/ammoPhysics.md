---
title: "Three.js 物理ammo使用教程"
description: "详解 Three.js 物理ammo使用：基于 WebGL 实现「物理ammo使用」可视化效果，附完整可运行源码，涵盖 OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 物理 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,物理ammo使用,WebGL,源码,教程,在线案例,OrbitControls,相机控制"
outline: deep
---

### 物理ammo使用 · *Ammo Physics* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=physics&id=ammoPhysics)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![物理ammo使用](https://z2586300277.github.io/three-cesium-examples/threeExamples/physics/ammoPhysics.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **物理ammo使用** 效果：基于 WebGL 实现「物理ammo使用」可视化效果，附完整可运行源码；核心用到 OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
import { OrbitControls } from 'three/addons/controls/OrbitControls.js'
import { AmmoPhysics } from 'three/addons/physics/AmmoPhysics.js'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(60, box.clientWidth / box.clientHeight, 1, 10000)

camera.position.set(15, 15, 15)

const renderer = new THREE.WebGLRenderer({ antialias: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

scene.add(new THREE.DirectionalLight(0xffffff, 3))

// 安装
const physics = await AmmoPhysics()

const floor = new THREE.Mesh(new THREE.BoxGeometry(60, 5, 60), new THREE.MeshLambertMaterial({ color: 0x444444 }))

floor.position.y -= 20

floor.userData.physics = { mass: 0 }

scene.add(floor)

for (let i = 0; i < 100; i++) {

    const geometry = Math.random() > 0.5 ? new THREE.IcosahedronGeometry() : new THREE.SphereGeometry()

    const material = new THREE.MeshLambertMaterial({ color: 0xffffff * Math.random() })

    const mesh = new THREE.Mesh(geometry, material)

    mesh.position.set(Math.random() - 0.5, Math.random() * 2, Math.random() - 0.5)

    mesh.userData.physics = { mass: 1 }

    scene.add(mesh)

}

physics.addScene(scene) // 启动物理引擎

animate()

function animate() {

    renderer.render(scene, camera)

    requestAnimationFrame(animate)
    
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/physics/ammoPhysics.js)

## 小结

- 本文提供 **物理ammo使用** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

