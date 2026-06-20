---
title: "Three.js 时间轴动画教程"
description: "详解 Three.js 时间轴动画：基于 WebGL 实现「时间轴动画」可视化效果，附完整可运行源码，涵盖 OrbitControls、GSAP 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,时间轴动画,WebGL,源码,教程,在线案例,OrbitControls,相机控制,GSAP,动画"
outline: deep
---

### 时间轴动画 · *Gsap TimeLine* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=gsapTimeLine)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![时间轴动画](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/gsapTimeLine.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- GSAP 时间轴与补间动画
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **时间轴动画** 效果：基于 WebGL 实现「时间轴动画」可视化效果，附完整可运行源码；核心用到 OrbitControls、GSAP。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- 阅读下方完整源码时，建议从 `init` / `load` / `animate` 三条主线入手，再深入 shader 与工具函数。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
3. 在定时器或 GSAP 时间轴中更新 uniform / 变换，驱动特效播放
4. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import gsap from 'gsap'

const box = document.getElementById('box')
const scene = new THREE.Scene()
const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 100000)
camera.position.set(0, 10, 20)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true })
renderer.setSize(box.clientWidth, box.clientHeight)
renderer.shadowMap.enabled = true
box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

// 场景设置
scene.background = new THREE.Color(0x87ceeb)
scene.add(new THREE.AmbientLight(0xffffff, 0.6))
const light = new THREE.DirectionalLight(0xffffff, 2)
light.position.set(10, 20, 5)
light.castShadow = true
scene.add(light)

const ground = new THREE.Mesh( new THREE.PlaneGeometry(30, 30),new THREE.MeshLambertMaterial({ color: 0x228b22 }))
ground.rotation.x = -Math.PI / 2
ground.receiveShadow = true
scene.add(ground)

// 花朵
const flower = new THREE.Group()
const stem = new THREE.Mesh(
    new THREE.CylinderGeometry(0.1, 0.1, 3),
    new THREE.MeshPhongMaterial({ color: 0x228b22 })
)
stem.position.y = 1.5
stem.castShadow = true

const petals = new THREE.Mesh(
    new THREE.SphereGeometry(1.2, 12, 12),
    new THREE.MeshPhongMaterial({ color: 0xff69b4, shininess: 100 })
)
petals.position.y = 3
petals.scale.set(1, 0.5, 1)
petals.castShadow = true

flower.add(stem, petals)
scene.add(flower)

// 蝴蝶
const butterfly = new THREE.Group()
const body = new THREE.Mesh(
    new THREE.CylinderGeometry(0.05, 0.05, 1),
    new THREE.MeshPhongMaterial({ color: 0x8b4513 })
)
const wing1 = new THREE.Mesh(
    new THREE.SphereGeometry(0.4, 8, 8),
    new THREE.MeshPhongMaterial({ color: 0xff6347, transparent: true, opacity: 0.8 })
)
wing1.scale.set(1.5, 0.1, 1)
wing1.position.set(0.3, 0, 0)

const wing2 = wing1.clone()
wing2.position.set(-0.3, 0, 0)

butterfly.add(body, wing1, wing2)
butterfly.position.set(5, 4, 0)
scene.add(butterfly)

// 简洁花园动画
const tl = gsap.timeline({ repeat: -1, yoyo: true }) // repeat: -1 无限循环, yoyo: true 往返动画

tl.from(flower.scale, { x: 0, y: 0, z: 0, duration: 2, ease: "back.out(1.7)" })
  .to(petals.material.color, { r: 1, g: 0.8, b: 0.2, duration: 1 }, "-=0.5") // -=0.5 表示与前一个动画同时开始
  .to(butterfly.position, { x: 0, y: 5, z: 2, duration: 3, ease: "power1.inOut" })
  .to(butterfly.rotation, { z: Math.PI / 4, duration: 0.5 }, "-=1") // -=1 表示与前一个动画同时开始
  .to(petals.scale, { x: 1.3, y: 0.6, z: 1.3, duration: 1 })
  .to(butterfly.position, { x: -3, y: 3, z: -2, duration: 2 })

animate()

function animate() {
    requestAnimationFrame(animate)
    
    // 蝴蝶翅膀扇动
    wing1.rotation.z = Math.sin(Date.now() * 0.02) * 0.3
    wing2.rotation.z = -Math.sin(Date.now() * 0.02) * 0.3
    
    // 花朵轻微摆动
    petals.rotation.y += 0.005
    
    renderer.render(scene, camera)
}

window.onresize = () => {
    renderer.setSize(box.clientWidth, box.clientHeight)
    camera.aspect = box.clientWidth / box.clientHeight
    camera.updateProjectionMatrix()
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/gsapTimeLine.js)

## 小结

- 本文提供 **时间轴动画** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

