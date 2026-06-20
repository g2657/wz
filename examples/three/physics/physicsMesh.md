---
title: "Three.js 物理cannon使用教程"
description: "详解 Three.js 物理cannon使用：基于 WebGL 实现「物理cannon使用」可视化效果，附完整可运行源码，涵盖 OrbitControls、物理引擎刚体碰撞 等关键实现，附完整源码与在线 Demo，适合 Three.js 物理 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,物理cannon使用,WebGL,源码,教程,在线案例,OrbitControls,相机控制,物理引擎,碰撞检测"
outline: deep
---
### 物理cannon使用 · *Physics Mesh* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=physics&id=physicsMesh)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![物理cannon使用](https://z2586300277.github.io/three-cesium-examples/threeExamples/physics/physicsMesh.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- 物理引擎刚体碰撞
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **物理cannon使用** 效果：基于 WebGL 实现「物理cannon使用」可视化效果，附完整可运行源码；核心用到 OrbitControls、物理引擎刚体碰撞。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **OrbitControls** 轨道旋转缩放；开 `enableDamping` 时每帧需 `controls.update()`。

## 实现步骤

1. 搭建 Scene / Camera / Renderer 与 OrbitControls
2. rAF 循环中 update 并 render

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import * as CANNON from 'cannon-es';

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 10, 10)

const renderer = new THREE.WebGLRenderer()

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

scene.add(new THREE.AxesHelper(5000), new THREE.AmbientLight(0xffffff, 0.5))

const pointLight = new THREE.PointLight(0xffffff, 3, 0)

pointLight.position.set(0, 2, 0)

scene.add(pointLight)

const world = new CANNON.World({ gravity: new CANNON.Vec3(0, -9.82, 0) })

// 渲染时间略有不同的时间显示物理模拟的状态
world.PhysicsRender = (deltaTime) => {

    world.step(1 / 60, deltaTime, 3)

    world.bodies.forEach(body => body.render?.())

}

// 创建一个立方体
const cubeGeometry = new THREE.BoxGeometry(1, 1, 1)

const cubeMaterial = new THREE.MeshStandardMaterial({ color: 0xffffff })

const cubeMesh = new THREE.Mesh(cubeGeometry, cubeMaterial)

cubeMesh.position.set(0, 5, 0)

createPhysicsBody(world, cubeMesh, 1)

scene.add(cubeMesh)

// 创建一个平面
const planeGeometry = new THREE.PlaneGeometry(10, 10)

const planeMaterial = new THREE.MeshStandardMaterial({ color: 0xffffff, side: THREE.DoubleSide })

const planeMesh = new THREE.Mesh(planeGeometry, planeMaterial)

planeMesh.rotation.x = Math.PI / 2

createPhysicsBody(world, planeMesh, 0)

scene.add(planeMesh)

animate()

function animate() {

    requestAnimationFrame(animate)

    world.PhysicsRender()

    renderer.render(scene, camera)

}

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

}

function createPhysicsBody(PhysicsWorld, model, mass) {

    const box = new THREE.Box3().setFromObject(model);

    const { max, min } = box

    const center = new THREE.Vector3();

    box.getCenter(center);

    const body = new CANNON.Body({

        mass: mass,

        shape: new CANNON.Box(new CANNON.Vec3((max.x - min.x) / 2, (max.y - min.y) / 2, (max.z - min.z) / 2)),

        position: center,

    })

    body.position.copy(model.position)

    PhysicsWorld.addBody(body)

    body.render = () => model.position.copy(body.position)

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/physics/physicsMesh.js)

## 小结

- 本文提供 **物理cannon使用** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

