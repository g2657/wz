---
title: "Three.js 拖拽控制教程"
description: "详解 Three.js 拖拽控制：基于 WebGL 实现「拖拽控制」可视化效果，附完整可运行源码，涵盖 OrbitControls、TransformControls、glTF/Draco 等关键实现，附完整源码与在线 Demo，适合 Three.js 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,拖拽控制,WebGL,源码,教程,在线案例,OrbitControls,相机控制,TransformControls,glTF,模型加载"
outline: deep
---

### 拖拽控制 · *Transform Controls* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=basic&id=transformObject)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![拖拽控制](https://z2586300277.github.io/three-cesium-examples/threeExamples/basic/transformObject.jpg)

## 你将学到什么

- **TransformControls** 三种模式 translate / rotate / scale
- `attach(object)` 绑定要操控的 Object3D
- 拖拽时 **禁用 OrbitControls** 避免冲突

## 效果说明

GUI 可切换模式，并将控制器 **attach** 到 Fox 模型、平行光或地面平面，拖拽 gizmo 实时变换对象。

## 核心概念

```js
const transformControls = new TransformControls(camera, renderer.domElement);
scene.add(transformControls.getHelper());

transformControls.addEventListener('dragging-changed', event => {
    controls.enabled = !event.value;  // 拖拽中关掉轨道控制
});

folder.add(transformControls, 'mode', ['translate', 'rotate', 'scale']);
transformControls.attach(model);
```

| 模式 | 快捷键（默认） | 作用 |
|------|---------------|------|
| translate | W | 平移 |
| rotate | E | 旋转 |
| scale | R | 缩放 |

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js"
import { TransformControls } from 'three/examples/jsm/controls/TransformControls.js'
import { GUI } from "three/addons/libs/lil-gui.module.min.js";

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(50, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 3, 6)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

renderer.shadowMap.needsUpdate = true

renderer.shadowMap.enabled = true

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

const folder = new GUI()

// 变换控制器
const transformControls = new TransformControls(camera, renderer.domElement)

// 模式 'translate' | 'rotate' | 'scale'
folder.add(transformControls, 'mode', ['translate', 'rotate', 'scale']).name('模式')

const transformControlsRoot = transformControls.getHelper()

scene.add(transformControlsRoot)

transformControls.addEventListener('dragging-changed', event => {

    controls.enabled = !event.value

})

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

}

new GLTFLoader().load(GLOBAL_CONFIG.getFileUrl('files/model/Fox.glb'), (gltf) => {

    const model = gltf.scene

    model.scale.set(0.01, 0.01, 0.01)

    model.traverse((child) => {

        if (child.isMesh) child.castShadow = true

    })

    scene.add(model)

    folder.add({ '控制模型': () => transformControls.attach(model) }, '控制模型')

})

const pointLight = new THREE.DirectionalLight(0xffffff, 1)

pointLight.position.set(1, 2, 0)

pointLight.castShadow = true

scene.add(pointLight)

folder.add({ '控制光源': () => transformControls.attach(pointLight) }, '控制光源')

const plane = new THREE.Mesh(new THREE.PlaneGeometry(100, 100), new THREE.MeshStandardMaterial({ color: 0xffffff }))

plane.position.y -= 0.5

plane.rotation.x = -Math.PI / 2

plane.receiveShadow = true

scene.add(plane)

folder.add({ '控制平面': () => transformControls.attach(plane) }, '控制平面')

animate()

function animate() {

    requestAnimationFrame(animate)

    controls.update()

    renderer.render(scene, camera)

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/basic/transformObject.js)

## 小结

- 本文提供 **拖拽控制** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

