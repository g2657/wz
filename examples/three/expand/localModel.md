---
title: "Three.js 本地模型加载教程"
description: "详解 Three.js 本地模型加载：基于 WebGL 实现「本地模型加载」可视化效果，附完整可运行源码，涵盖 OrbitControls、glTF/Draco 等关键实现，附完整源码与在线 Demo，适合 Three.js 扩展 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,本地模型加载,WebGL,源码,教程,在线案例,OrbitControls,相机控制,glTF,模型加载"
outline: deep
---

### 本地模型加载 · *Local Model* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=expand&id=localModel)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![本地模型加载](https://z2586300277.github.io/three-cesium-examples/threeExamples/basic/localModel.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- glTF/Draco 模型加载与优化
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **本地模型加载** 效果：基于 WebGL 实现「本地模型加载」可视化效果，附完整可运行源码；核心用到 OrbitControls、glTF/Draco。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(50, box.clientWidth / box.clientHeight, 0.1, 10000000)

camera.position.set(10, 10, 10)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

scene.add(new THREE.AxesHelper(500), new THREE.AmbientLight(0xffffff, 2))

animate()

function animate() {

  requestAnimationFrame(animate)

  controls.update()

  renderer.render(scene, camera)

}

window.onresize = () => {

  renderer.setSize(box.clientWidth, box.clientHeight)

  camera.aspect = box.clientWidth / box.clientHeight

  camera.updateProjectionMatrix()

}

// 文件地址
const urls = [0, 1, 2, 3, 4, 5].map(k => (FILE_HOST + 'files/sky/skyBox0/' + (k + 1) + '.png'));

const textureCube = new THREE.CubeTextureLoader().load(urls);

scene.background = textureCube;

// 创建一个文件上传的输入框
const input = document.createElement('input')
input.type = 'file'
input.accept = '.glb'
Object.assign(input.style, {
  position: 'absolute',
  top: '30px',
  left: '100px',
  zIndex: 9999
})
input.onchange = (e) => {
  const file = e.target.files[0]
  const url = URL.createObjectURL(file)
  new GLTFLoader().setDRACOLoader(new DRACOLoader().setDecoderPath(FILE_HOST + 'js/three/draco/')).load(url, (gltf) => {
    gltf.scene.traverse((child) => {
      if (child?.material) child.material.envMap = textureCube
    })
    scene.add(gltf.scene)

    // 主视图
    const box3 = new THREE.Box3().setFromObject(gltf.scene)
    const center = box3.getCenter(new THREE.Vector3())
    const size = box3.getSize(new THREE.Vector3())
    const maxDim = Math.max(size.x, size.y, size.z)
    const fov = camera.fov * (Math.PI / 180)
    const distance = (maxDim / 2) / Math.tan(fov / 2)
    camera.position.set(center.x, center.y, center.z + distance + size.z / 2)
    camera.lookAt(center)
    controls.target.copy(center)
    controls.update()

  })
  URL.revokeObjectURL(url)
}
document.body.appendChild(input)
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/basic/localModel.js)

## 小结

- 本文提供 **本地模型加载** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

