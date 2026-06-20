---
title: "Three.js 第一人称漫游控制教程"
description: "详解 Three.js 第一人称漫游控制：基于 WebGL 实现「第一人称漫游控制」可视化效果，附完整可运行源码，涵盖 OrbitControls、FBXLoader 等关键实现，附完整源码与在线 Demo，适合 Three.js 动画 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,第一人称漫游控制,WebGL,源码,教程,在线案例,OrbitControls,相机控制,FBX,模型加载"
outline: deep
---

### 第一人称漫游控制 · *Person Move* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=animation&id=pointLockControls)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![第一人称漫游控制](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/pointLockControls.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- FBXLoader 加载 FBX 城市/角色模型
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **第一人称漫游控制** 效果：基于 WebGL 实现「第一人称漫游控制」可视化效果，附完整可运行源码；核心用到 OrbitControls、FBXLoader。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- 阅读下方完整源码时，建议从 `init` / `load` / `animate` 三条主线入手，再深入 shader 与工具函数。

## 实现步骤

1. 搭建灯光与环境（如有）
2. requestAnimationFrame 循环 update + render

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { FBXLoader } from 'three/examples/jsm/loaders/FBXLoader.js'
import { PointerLockControls } from 'three/examples/jsm/controls/PointerLockControls.js'
import { GUI } from 'dat.gui'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(50, box.clientWidth / box.clientHeight, 0.1, 100000)
camera.position.set(0, 100, 400)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })
renderer.setSize(box.clientWidth, box.clientHeight)
box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)
controls.enableDamping = true

scene.add(new THREE.AmbientLight(0xffffff, 1))

// 天空
const urls = [0, 1, 2, 3, 4, 5].map(k => (FILE_HOST + 'files/sky/skyBox0/' + (k + 1) + '.png'));
const textureCube = new THREE.CubeTextureLoader().load(urls);
scene.background = textureCube;
scene.backgroundBlurriness = 0.5;

// 地面
const groundGeometry = new THREE.PlaneGeometry(10000, 10000);
const map = new THREE.TextureLoader().load(FILE_HOST + 'images/channels/black.png')
map.wrapS = map.wrapT = THREE.RepeatWrapping;
map.repeat.set(100, 100);
const groundMaterial = new THREE.MeshStandardMaterial({ map, envMap: textureCube });
const groundMesh = new THREE.Mesh(groundGeometry, groundMaterial);
groundMesh.rotation.x = -Math.PI / 2;
scene.add(groundMesh);

// 加载模型 fbx
new FBXLoader().load(HOST + '/files/model/city.FBX', (object3d) => {
    scene.add(object3d)
    object3d.scale.set(0.06, 0.06, 0.06)
})

// 创建第一人称漫游控制器
const pointLockControls = new PointerLockControls(camera, box);
pointLockControls.speed = 2

// 处理键盘事件以实现相机的移动
const keyboard = {};
const moveForward = () => pointLockControls.moveForward(pointLockControls.speed)
const moveBackward = () => pointLockControls.moveForward(-pointLockControls.speed)
const moveLeft = () => pointLockControls.moveRight(-pointLockControls.speed)
const moveRight = () => pointLockControls.moveRight(pointLockControls.speed)
const moveUp = () => pointLockControls.getObject().position.y += pointLockControls.speed
const moveDown = () => pointLockControls.getObject().position.y -= pointLockControls.speed

// 第一人称
pointLockControls.pointLockRender = () => {
    if (keyboard['KeyW']) moveForward();
    else if (keyboard['KeyA']) moveLeft();
    else if (keyboard['KeyS']) moveBackward();
    else if (keyboard['KeyD']) moveRight();
    else if (keyboard['ArrowUp']) moveUp();
    else if (keyboard['ArrowDown']) moveDown();
}

// 键盘事件 
const onKeyDown = function (event) {
    event.preventDefault();
    keyboard[event.code] = true
}

// 键盘事件
const onKeyUp = function (event) {
    event.preventDefault();
    keyboard[event.code] = false
}

// 锁定事件
pointLockControls.addEventListener('lock', () => {
    window.addEventListener('keydown', onKeyDown)
    window.addEventListener('keyup', onKeyUp)
})

// 解锁事件
pointLockControls.addEventListener('unlock', () => {
    window.removeEventListener('keydown', onKeyDown)
    window.removeEventListener('keyup', onKeyUp)
})

const gui = new GUI()
gui.add(pointLockControls, 'isLocked').onChange((v) => v ? pointLockControls.lock() : pointLockControls.unlock()).name('漫游').listen()
gui.add(pointLockControls, 'speed').name('漫游速度')

animate()
function animate() {
    requestAnimationFrame(animate)
    pointLockControls.isLocked ? pointLockControls.pointLockRender() : controls.update()
    renderer.render(scene, camera)
}

window.onresize = () => {
    renderer.setSize(box.clientWidth, box.clientHeight)
    camera.aspect = box.clientWidth / box.clientHeight
    camera.updateProjectionMatrix()
}

GLOBAL_CONFIG.ElMessage('键盘事件：WASD移动, esc退出')
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/pointLockControls.js)

## 小结

- 本文提供 **第一人称漫游控制** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

