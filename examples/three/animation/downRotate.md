---
title: "Three.js 下钻动画教程"
description: "详解 Three.js 下钻动画：基于 WebGL 实现「下钻动画」可视化效果，附完整可运行源码，涵盖 OrbitControls、CatmullRomCurve3、glTF/Draco 等关键实现，附完整源码与在线 Demo，适合 Three.js 动画 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,下钻动画,WebGL,源码,教程,在线案例,OrbitControls,相机控制,样条曲线,路径动画,glTF,模型加载"
outline: deep
---

### 下钻动画 · *Down Rotate* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=animation&id=downRotate)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![下钻动画](https://z2586300277.github.io/three-cesium-examples/threeExamples/animation/downRotate.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- CatmullRomCurve3 样条曲线路径
- glTF/Draco 模型加载与优化
- BufferGeometry 自定义顶点/索引数据
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **下钻动画** 效果：基于 WebGL 实现「下钻动画」可视化效果，附完整可运行源码；核心用到 OrbitControls、CatmullRomCurve3、glTF/Draco。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- 曲线类 `getPoints(n)` 将贝塞尔/样条离散为路径点，再写入 BufferGeometry 驱动飞线或路径动画。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 异步加载模型 / 3D Tiles / GeoJSON 等资源并加入 scene 或 entities
3. 用曲线离散点构建 BufferGeometry，写入自定义 attribute 驱动动画
4. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
5. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(30, 30, 30)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

}

scene.add(new THREE.AmbientLight(0xffffff, 2))

// 创建一个曲线
const curve = new THREE.CatmullRomCurve3([

    new THREE.Vector3(0, 20, 2),

    new THREE.Vector3(0, 0, 0),

    new THREE.Vector3(10, 8, 20),

])

// 创建曲线几何
const geometry = new THREE.BufferGeometry().setFromPoints(curve.getPoints(500))

// 创建曲线材质
const material = new THREE.LineBasicMaterial({ color: 0xffffff })

// 创建曲线
const curveMesh = new THREE.Line(geometry, material)

// 添加曲线到场景
scene.add(curveMesh)

let obj = null

const loader = new GLTFLoader()

loader.load('https://z2586300277.github.io/3d-file-server/models/glb/daodan.glb', g => {

    scene.add(g.scene)

    g.scene.scale.multiplyScalar(0.1)

    obj = g.scene

})

animate()

const speed = 0.001 // 控制物体沿曲线移动的速度

const spin = 0.15 // 控制物体自旋的速度

let t = 0 // 当前物体在曲线上的位置（0~1之间）

let r = 0 // 当前自旋的角度

const baseDir = new THREE.Vector3(0, 0, 1) // 物体默认朝向（z轴正方向）

function animate() {

    requestAnimationFrame(animate)

    if (obj) {

        t = (t + speed) % 1 // 更新位置参数

        r += spin // 更新自旋角度

        obj.position.copy(curve.getPointAt(t)) // 设置物体位置

        const tangent = curve.getTangentAt(t).normalize() // 计算切线方向

        const lookQuat = new THREE.Quaternion().setFromUnitVectors(baseDir, tangent) // 计算朝向四元数

        const spinQuat = new THREE.Quaternion().setFromAxisAngle(tangent, r) // 计算自旋四元数

        obj.quaternion.copy(lookQuat).premultiply(spinQuat) // 应用旋转

    }

    renderer.render(scene, camera)
    
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/animation/downRotate.js)

## 小结

- 本文提供 **下钻动画** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

