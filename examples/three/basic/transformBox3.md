---
title: "Three.js 变换 Box3教程"
description: "详解 Three.js 变换 Box3：基于 WebGL 实现「变换 Box3」可视化效果，附完整可运行源码，涵盖 OrbitControls、TransformControls、glTF/Draco 等关键实现，附完整源码与在线 Demo，适合 Three.js 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,变换 Box3,WebGL,源码,教程,在线案例,OrbitControls,相机控制,TransformControls,glTF,模型加载"
outline: deep
---

### 变换 Box3 · *Box3 Helper* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=basic&id=transformBox3)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![变换Box3](https://z2586300277.github.io/three-cesium-examples/threeExamples/basic/transformBox3.jpg)

## 你将学到什么

- **Box3.setFromObject** 计算物体世界空间 AABB
- **Box3Helper** 可视化黄色包围盒
- TransformControls `change` 事件驱动包围盒刷新

## 效果说明

Fox 模型挂载 TransformControls，拖拽时 **黄色线框包围盒** 跟随更新，直观看到物体占用空间。

## 核心概念

```js
const box3 = new THREE.Box3();
const box3Helper = new THREE.Box3Helper(box3, 0xffff00);
scene.add(box3Helper);

transformControls.addEventListener('change', () => {
    box3Helper.box = box3.setFromObject(transformControls.object);
});
```

**AABB**（轴对齐包围盒）不随物体旋转而旋转，始终与世界轴平行，适合碰撞粗测、视图 fit。

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js"
import { TransformControls } from 'three/examples/jsm/controls/TransformControls.js'
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js'
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

folder.add(transformControls, 'mode', ['translate', 'rotate', 'scale']).name('模式')

const transformControlsRoot = transformControls.getHelper()

transformControls.addEventListener('dragging-changed', event => controls.enabled = !event.value)

const box3 = new THREE.Box3()

const box3Helper = new THREE.Box3Helper(box3, 0xffff00)

scene.add(box3Helper)

transformControls.addEventListener('change', () => box3Helper.box = box3.setFromObject(transformControls.object))

scene.add(transformControlsRoot)

// 模型
const texture = new RGBELoader().load(FILE_HOST + '/files/hdr/1k.hdr', t => t.mapping = THREE.EquirectangularReflectionMapping)

new GLTFLoader().load(GLOBAL_CONFIG.getFileUrl('files/model/Fox.glb'), (gltf) => {

    const model = gltf.scene

    model.scale.set(0.01, 0.01, 0.01)

    model.traverse((child) => child.isMesh && (child.material.envMap = texture))

    scene.add(model)

    transformControls.attach(model)

})

animate()

function animate() {

    requestAnimationFrame(animate)

    controls.update()

    renderer.render(scene, camera)

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/basic/transformBox3.js)

## 小结

- 本文提供 **变换 Box3** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

