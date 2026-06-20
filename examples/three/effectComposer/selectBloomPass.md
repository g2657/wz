---
title: "Three.js 辉光-postprocessing教程"
description: "详解 Three.js 辉光-postprocessing：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期，涵盖 EffectComposer、OrbitControls、Raycaster 等关键实现，附完整源码与在线 Demo，适合 Three.js 后期处理 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,辉光-postprocessing,WebGL,源码,教程,在线案例,EffectComposer,后期处理,OrbitControls,相机控制,Raycaster,射线检测"
outline: deep
---

### 辉光-postprocessing · *Select Bloom* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=effectComposer&id=selectBloomPass)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![辉光-postprocessing](https://z2586300277.github.io/three-cesium-examples/threeExamples/effectComposer/selectBloomPass.jpg)

## 你将学到什么

- EffectComposer 多 Pass 后期处理管线
- OrbitControls 相机轨道交互
- Raycaster 鼠标拾取与交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **辉光-postprocessing** 效果：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期，支持鼠标拾取、绘制或拖拽交互；核心用到 EffectComposer、OrbitControls、Raycaster。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **EffectComposer** 以多 Pass 链式渲染：RenderPass → 特效 Pass → 输出屏幕，替代直接 `renderer.render`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- **Raycaster** 将屏幕坐标转为射线，与场景求交得到世界坐标，常用于绘制/拾取。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 组装 EffectComposer Pass 链，在 animate 中调用 `composer.render()`
3. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
4. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { BlendFunction, SelectiveBloomEffect, EffectComposer, EffectPass, RenderPass } from "postprocessing";

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 5, 5)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

}

// 附加后处理库
const composer = new EffectComposer(renderer);

const bloomEffect = new SelectiveBloomEffect(scene, camera, {

    blendFunction: BlendFunction.ADD,

    mipmapBlur: true,

    luminanceThreshold: 0.4,

    luminanceSmoothing: 0.2,

    intensity: 3.0

})

composer.addPass(new RenderPass(scene, camera))

composer.addPass(new EffectPass(camera, bloomEffect))

// 添加10个立方体
const boxGeometry = new THREE.BoxGeometry(1, 1, 1)

const boxMaterial = new THREE.MeshBasicMaterial({ color: 0xa0dee2 })

const boxMaterial2 = new THREE.MeshBasicMaterial({ color: 'yellow' })

const box1 = new THREE.Mesh(boxGeometry, boxMaterial)

const box2 = new THREE.Mesh(boxGeometry, boxMaterial2)

box1.position.set(-1.5, 0, 0)

scene.add(box1, box2)

bloomEffect.selection.set([box1], true)

// 点击立方体时，高亮立方体
box.addEventListener('click', e => {

    const raycaster = new THREE.Raycaster()

    const mouse = new THREE.Vector2(

        (e.offsetX / e.target.clientWidth) * 2 - 1,

        -(e.offsetY / e.target.clientHeight) * 2 + 1

    )

    raycaster.setFromCamera(mouse, camera)

    const intersects = raycaster.intersectObjects(scene.children)

    if (intersects.length > 0) {

        const object = intersects[0].object

        bloomEffect.selection.toggle(object);

    }

})

// 渲染
function animate() {

    requestAnimationFrame(animate)

    composer.render()

}

animate()
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/effectComposer/selectBloomPass.js)

## 小结

- 本文提供 **辉光-postprocessing** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

