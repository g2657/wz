---
title: "Three.js 多轮廓光教程"
description: "详解 Three.js 多轮廓光：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期，涵盖 EffectComposer、OutlinePass、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,多轮廓光,WebGL,源码,教程,在线案例,EffectComposer,后期处理,OutlinePass,轮廓高亮,OrbitControls,相机控制"
outline: deep
---

### 多轮廓光 · *Multi Outline* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=basic&id=multOutlinePass)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

## 你将学到什么

- 一条 composer 链上 **多个 OutlinePass**
- 每个 Pass 独立 **selectedObjects** 与 **visibleEdgeColor**
- 锥体红边、立方体默认、球体蓝边

## 效果说明

三个 primitive 各自由一个 OutlinePass 负责描边，最后 **OutputPass** 输出。

```js
outlinePass.selectedObjects = [cone];      // 红
outputPass2.selectedObjects = [cube];
outlinePass3.selectedObjects = [sphere]; // 蓝
```

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
import { OutlinePass } from 'three/examples/jsm/postprocessing/OutlinePass.js'
import { OutputPass } from 'three/examples/jsm/postprocessing/OutputPass.js'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 0, 5)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

// 后期处理
const composer = new EffectComposer(renderer);

const renderPass = new RenderPass(scene, camera);

composer.addPass(renderPass);

// 轮廓
const outlinePass = new OutlinePass(new THREE.Vector2(box.clientWidth, box.clientHeight), scene, camera);

outlinePass.visibleEdgeColor.set('red'); // 设置可见边缘颜色

composer.addPass(outlinePass);

const outputPass2 = new OutlinePass(new THREE.Vector2(box.clientWidth, box.clientHeight), scene, camera);

composer.addPass(outputPass2);

const outlinePass3 = new OutlinePass(new THREE.Vector2(box.clientWidth, box.clientHeight), scene, camera);

outlinePass3.visibleEdgeColor.set('blue');

composer.addPass(outlinePass3);

// 色彩校正
const outputPass = new OutputPass();

composer.addPass(outputPass);

const cone = new THREE.Mesh(new THREE.ConeGeometry(1, 2, 32), new THREE.MeshBasicMaterial())

cone.position.set(2, 0, 0)

scene.add(cone);

outlinePass.selectedObjects = [cone]

const cube = new THREE.Mesh(new THREE.BoxGeometry(1, 1, 1), new THREE.MeshBasicMaterial({ color: 0x00ff00 }))

scene.add(cube);

outputPass2.selectedObjects = [cube];

const sphere = new THREE.Mesh(new THREE.SphereGeometry(1, 32, 32), new THREE.MeshBasicMaterial({ color: 'yellow' }))

sphere.position.set(-2, 0, 0)

scene.add(sphere);

outlinePass3.selectedObjects = [sphere];

// 渲染
animate()

function animate() {

    requestAnimationFrame(animate)

    controls.update()

    composer.render()

}

// 适配
window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/basic/multOutlinePass.js)

## 小结

- 本文提供 **多轮廓光** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

