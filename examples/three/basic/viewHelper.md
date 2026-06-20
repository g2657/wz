---
title: "Three.js 视图辅助教程"
description: "详解 Three.js 视图辅助：基于 WebGL 实现「视图辅助」可视化效果，附完整可运行源码，涵盖 OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,视图辅助,WebGL,源码,教程,在线案例,OrbitControls,相机控制"
outline: deep
---

### 视图辅助 · *View Helper* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=basic&id=viewHelper)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

## 你将学到什么

- **ViewHelper** 右下角 orientation gizmo
- `renderer.autoClear = false` 叠加绘制
- 主场景 render 后再 `viewHelper.render(renderer)`

## 效果说明

网格场景右下角显示 **XYZ 方向小控件**，点击可快速切换标准视角（Three.js r152+ ViewHelper 内置交互）。

## 核心概念

```js
renderer.autoClear = false;
const viewHelper = new ViewHelper(camera, renderer.domElement);

function animate() {
    controls.update();
    renderer.render(scene, camera);
    viewHelper.render(renderer);  // 叠在小角落
    requestAnimationFrame(animate);
}
```

编辑器、Blender 式视口方向指示同款思路。

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { ViewHelper } from 'three/examples/jsm/helpers/ViewHelper.js'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(50, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 10, 10)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

scene.add(new THREE.AxesHelper(5), new THREE.GridHelper(10, 10))

const viewHelper = new ViewHelper(camera, renderer.domElement)

renderer.autoClear = false // 需要将自动清除关闭

animate()

function animate() {

    controls.update()

    // renderer.clear() // 可能需要的清除操作

    renderer.render(scene, camera)

    viewHelper.render(renderer)

    requestAnimationFrame(animate)

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/basic/viewHelper.js)

## 小结
