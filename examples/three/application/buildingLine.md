---
title: "Three.js 建筑线条教程"
description: "详解 Three.js 建筑线条：基于 WebGL 实现「建筑线条」可视化效果，附完整可运行源码，涵盖 OrbitControls、glTF/Draco、EdgesGeometry 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,建筑线条,WebGL,源码,教程,在线案例,OrbitControls,相机控制,glTF,模型加载,EdgesGeometry,边线"
outline: deep
---

### 建筑线条 · *Building Line* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=buildingLine)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![建筑线条](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/buildingLine.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- glTF/Draco 模型加载与优化
- EdgesGeometry 模型边线提取
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **建筑线条** 效果：基于 WebGL 实现「建筑线条」可视化效果，附完整可运行源码；核心用到 OrbitControls、glTF/Draco、EdgesGeometry。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- 阅读下方完整源码时，建议从 `init` / `load` / `animate` 三条主线入手，再深入 shader 与工具函数。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 异步加载模型 / 3D Tiles / GeoJSON 等资源并加入 scene 或 entities
3. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
4. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { LineSegments2 } from 'three/examples/jsm/lines/LineSegments2.js';
import { LineMaterial } from 'three/examples/jsm/lines/LineMaterial.js';
import { LineSegmentsGeometry } from 'three/examples/jsm/lines/LineSegmentsGeometry.js';

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 100000)

camera.position.set(1, 1, 1)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true })

renderer.setPixelRatio(window.devicePixelRatio * 1.3)

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

scene.add(new THREE.AxesHelper(100))

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

const urls = [0, 1, 2, 3, 4, 5].map(k => (FILE_HOST + 'files/sky/skyBox0/' + (k + 1) + '.png'));

const textureCube = new THREE.CubeTextureLoader().load(urls);

new GLTFLoader().load(FILE_HOST + 'models/whitebuild.glb', (gltf) => {

    const model = gltf.scene

    model.traverse((child) => {

        if (child.isMesh) {

            child.material.envMap = textureCube

            const { geometry } = child
            const edges = new THREE.EdgesGeometry(geometry)
            let lineGeometry = new LineSegmentsGeometry().fromEdgesGeometry(edges)
            const material = new LineMaterial({
                color: 0xfcde8c,
                linewidth: 2.5,
                envMap: textureCube,
                resolution: new THREE.Vector2(box.clientWidth, box.clientHeight)
            })

            const lines = new LineSegments2(lineGeometry, material)
            lines.computeLineDistances()
            lines.applyMatrix4(child.matrixWorld)

            scene.add(lines)
        }

    })

    scene.add(model)
})
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/buildingLine.js)

## 小结

- 本文提供 **建筑线条** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

