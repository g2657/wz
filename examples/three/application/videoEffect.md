---
title: "Three.js 视频碎片教程"
description: "详解 Three.js 视频碎片：基于 WebGL 实现「视频碎片」可视化效果，附完整可运行源码，涵盖 OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,视频碎片,WebGL,源码,教程,在线案例,OrbitControls,相机控制"
outline: deep
---

### 视频碎片 · *Video Effect* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=videoEffect)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![视频碎片](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/videoEffect.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **视频碎片** 效果：基于 WebGL 实现「视频碎片」可视化效果，附完整可运行源码；核心用到 OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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

camera.position.set(0, 0, 20)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

const video = document.createElement('video')

video.crossOrigin = 'anonymous' // 跨域

video.src = 'https://z2586300277.github.io/3d-file-server/video/test.mp4'

video.loop = true // 循环播放

video.muted = true // 静音

video.play()

const texture = await new Promise(r => video.onloadeddata = () => r(new THREE.VideoTexture(video))) // 创建视频纹理

const group = new THREE.Group()
const config = {
    width: 16,
    height: 9,
    xGrid: 4,
    yGrid: 3,
    offset: 0.1
}
const ux = 1 / config.xGrid
const uy = 1 / config.yGrid
const planeWidth = config.width / config.xGrid - config.offset
const planeHeight = config.height / config.yGrid - config.offset
for (let i = 0; i < config.xGrid; i++) {
    for (let j = 0; j < config.yGrid; j++) {
        // 创建 4 * 3 子平面实现整体效果
        const geometry = new THREE.PlaneGeometry(planeWidth, planeHeight)
        const material = new THREE.MeshBasicMaterial({
            map: texture,
            side: THREE.DoubleSide
        })

        // 切割uv来实现纹理映射到全部平面
        const uvs = geometry.attributes.uv.array
        for (let index = 0; index < uvs.length; index += 2) {
            uvs[index] = (uvs[index] + i) * ux
            uvs[index + 1] = (uvs[index + 1] + j) * uy
        }

        const mesh = new THREE.Mesh(geometry, material)

        mesh.dx = 0.004 * (0.5 - Math.random())
        mesh.dy = 0.004 * (0.5 - Math.random())

        const x = (i - config.xGrid / 2) * planeWidth + planeWidth * 0.5 + (i * config.offset) / 2
        const y = (j - config.yGrid / 2) * planeHeight + planeHeight * 0.5 + (j * config.offset) / 2
        mesh.position.set(x, y, 0)
        group.add(mesh)

    }

}
scene.add(group)

const clock = new THREE.Clock()
animate()

function animate() {

    const elapsedTime = clock.getElapsedTime()

    for (const mesh of group.children) {
        mesh.rotation.x += Math.sin(elapsedTime * 0.1) * mesh.dx;
        mesh.rotation.y += Math.sin(elapsedTime * 0.2) * mesh.dy;

        mesh.position.x -= Math.sin(elapsedTime * 0.1) * mesh.dx;
        mesh.position.y += Math.sin(elapsedTime * 0.3) * mesh.dy;
        mesh.position.z += Math.cos(elapsedTime * 0.2) * mesh.dx;
    }

    requestAnimationFrame(animate)

    controls.update()

    renderer.render(scene, camera)

}

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/videoEffect.js)

## 小结

- 本文提供 **视频碎片** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

