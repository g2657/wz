---
title: "Three.js 视频地板教程"
description: "详解 Three.js 视频地板：基于 WebGL 实现「视频地板」可视化效果，附完整可运行源码，涵盖 OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,视频地板,WebGL,源码,教程,在线案例,OrbitControls,相机控制"
outline: deep
---
### 视频地板 · *Video Floor* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=videoFloor)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![视频地板](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/videoFloor.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **视频地板** 效果：基于 WebGL 实现「视频地板」可视化效果，附完整可运行源码；核心用到 OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **OrbitControls** 轨道旋转缩放；开 `enableDamping` 时每帧需 `controls.update()`。

## 实现步骤

1. 搭建 Scene / Camera / Renderer 与 OrbitControls
2. rAF 循环中 update 并 render

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 100000)

camera.position.set(2, 2, 2)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setPixelRatio(window.devicePixelRatio * 1.5)

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)
controls.enableDamping = true

scene.add(new THREE.AxesHelper(100), new THREE.GridHelper(100, 10))

const directionalLight = new THREE.DirectionalLight(0xffffff, 3)
directionalLight.position.set(5, 5, 5)
scene.add(directionalLight)

const ambientLight = new THREE.AmbientLight(0xffffff, 2)
scene.add(ambientLight)

animate()

function animate() {

    controls.update()

    requestAnimationFrame(animate)

    renderer.render(scene, camera)

}

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

}

async function createVideoPlane(url, width, height, positionY) {

    const video = document.createElement('video')
    video.crossOrigin = 'anonymous'
    video.src = url
    video.loop = true
    video.muted = true
    video.play()
    const texture = new THREE.VideoTexture(video)
    const geometry = new THREE.PlaneGeometry(width, height)

    const material = new THREE.MeshStandardMaterial({
        color: 0xffffff * Math.random(), // 随机颜色
        alphaMap: texture,
        opecity: 0.5, // 透明度，可调整
        transparent: true,
        side: THREE.DoubleSide,
    })

    const mesh = new THREE.Mesh(geometry, material)
    mesh.rotation.x = -Math.PI / 2
    mesh.position.set(0, positionY, 0) // 设置位置
    scene.add(mesh)
}

createVideoPlane(FILE_HOST + 'files/video/c1.mp4', 3, 3 , 0.01)
createVideoPlane(FILE_HOST + 'files/video/c2.mp4', 4, 4, 0)
createVideoPlane(FILE_HOST + 'files/video/c3.mp4', 5, 5, -0.01)
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/videoFloor.js)

## 小结

- 本文提供 **视频地板** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

