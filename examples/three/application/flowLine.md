---
title: "Three.js 贴图飞线教程"
description: "详解 Three.js 贴图飞线：基于 WebGL 实现「贴图飞线」可视化效果，附完整可运行源码，涵盖 OrbitControls、CatmullRomCurve3 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,贴图飞线,WebGL,源码,教程,在线案例,OrbitControls,相机控制,样条曲线,路径动画"
outline: deep
---

### 贴图飞线 · *Flow Line* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=flowLine)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![贴图飞线](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/flowLine.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- CatmullRomCurve3 样条曲线路径
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **贴图飞线** 效果：基于 WebGL 实现「贴图飞线」可视化效果，附完整可运行源码；核心用到 OrbitControls、CatmullRomCurve3。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- 曲线类 `getPoints(n)` 将贝塞尔/样条离散为路径点，再写入 BufferGeometry 驱动飞线或路径动画。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 用曲线离散点构建 BufferGeometry，写入自定义 attribute 驱动动画
3. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
4. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(50, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(7, 7, 7)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

const gridHelper = new THREE.GridHelper(10, 10)

scene.add(gridHelper)

// 生成一个管道
const curve = new THREE.CatmullRomCurve3([
    new THREE.Vector3(-5, 0, 0),
    new THREE.Vector3(0, 2.5, 0),
    new THREE.Vector3(5, 0, 0)
])

const map = new THREE.TextureLoader().load(FILE_HOST + '/images/texture/flyLine1.png')

map.wrapS = THREE.RepeatWrapping

map.repeat.set(5, 2)

const tube = new THREE.TubeGeometry(curve, 200, 0.2, 2, false);

const material = new THREE.MeshBasicMaterial({ map, transparent: true })

const mesh = new THREE.Mesh(tube, material)

scene.add(mesh)

// 生成一个飞线
const curve1 = new THREE.CatmullRomCurve3([
    new THREE.Vector3(-3, 0, 3),
    new THREE.Vector3(0, 2, 0),
    new THREE.Vector3(3, 0, -4),
    
])

const map1 = new THREE.TextureLoader().load(FILE_HOST + '/images/texture/flyLine4.png')

map1.wrapS = THREE.RepeatWrapping

map1.wrapT = THREE.RepeatWrapping

map1.repeat.set(3, 1)

const tube1 = new THREE.TubeGeometry(curve1, 200, 0.03, 20, false);

const material1 = new THREE.MeshBasicMaterial({map:map1, transparent: true , side: THREE.DoubleSide})

const mesh1 = new THREE.Mesh(tube1, material1)

scene.add(mesh1)

const mesh2 = mesh.clone()

mesh2.rotation.y = Math.PI / 2

scene.add(mesh2)

animate()

function animate() {

    map.offset.x -= 0.01

    map1.offset.x -= 0.01

    controls.update()

    renderer.render(scene, camera)

    requestAnimationFrame(animate)

}

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/flowLine.js)

## 小结

- 本文提供 **贴图飞线** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

