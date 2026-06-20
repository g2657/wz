---
title: "Three.js 动画合集教程"
description: "详解 Three.js 动画合集：基于 WebGL 实现「动画合集」可视化效果，附完整可运行源码，涵盖 GSAP、场景雾效增强纵深 等关键实现，附完整源码与在线 Demo，适合 Three.js 动画 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,动画合集,WebGL,源码,教程,在线案例,GSAP,动画,雾效"
outline: deep
---

### 动画合集 · *GSAP* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=animation&id=gsapCollection)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![动画合集](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/gsapCollection.jpg)

## 你将学到什么

- GSAP 时间轴与补间动画
- 场景雾效增强纵深
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **动画合集** 效果：基于 WebGL 实现「动画合集」可视化效果，附完整可运行源码；核心用到 GSAP、场景雾效增强纵深。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- 阅读下方完整源码时，建议从 `init` / `load` / `animate` 三条主线入手，再深入 shader 与工具函数。

## 实现步骤

1. 搭建灯光与环境（如有）
2. requestAnimationFrame 循环 update + render

## 代码要点

```js
import * as THREE from "three";
import gsap from 'gsap'

const scene = new THREE.Scene();
scene.fog = new THREE.FogExp2(0x1E2630, 0.002)

const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 1, 1000)
camera.animAngle = 0
camera.position.set(Math.cos(camera.animAngle) * 400, 180, Math.sin(camera.animAngle) * 400)

const renderer = new THREE.WebGLRenderer({ antialias: true })
renderer.setPixelRatio(window.devicePixelRatio)
renderer.setSize(window.innerWidth, window.innerHeight)
document.body.appendChild(renderer.domElement)
renderer.setClearColor(scene.fog.color)

scene.add(new THREE.GridHelper(600, 10))

const alight = new THREE.AmbientLight(0xffffff, 0.2)
scene.add(alight)

const light = new THREE.DirectionalLight(0xffffff, 1)
light.position.set(1, 1, 1)
scene.add(light)

const light2 = new THREE.DirectionalLight(0x002288, 1)
light2.position.set(-1, -1, -1)
scene.add(light2)

const dome = new THREE.Mesh(new THREE.IcosahedronGeometry(700, 1), new THREE.MeshPhongMaterial({
    color: 0xfb3550,
    side: THREE.BackSide
}))
scene.add(dome)

const planeGeometry = new THREE.PlaneGeometry(600, 600)
const planeMaterial = new THREE.MeshPhongMaterial({
    color: 0x222A38,
    transparent: true,
    opacity: 0.8,
    side: THREE.DoubleSide
})
const plane = new THREE.Mesh(planeGeometry, planeMaterial)
plane.rotation.x = Math.PI / 2
scene.add(plane)

const geometry = new THREE.BoxGeometry(10, 10, 10)
const getMaterial = () => new THREE.MeshPhongMaterial({ color: 0xffffff * Math.random() })
const boxes = new Array(100).fill('').map(() => {
    const mesh = new THREE.Mesh(geometry, getMaterial())
    scene.add(mesh)
    return mesh
})

loop()
function loop() {

    const t = 1.5; //时间

    boxes.forEach(box => {

        gsap.to(box.scale, t, {
            x: 1 + Math.random() * 3,
            y: 1 + Math.random() * 20 + (Math.random() < 0.1 ? 15 : 0),
            z: 1 + Math.random() * 3
        })
        gsap.to(box.position, t, {
            x: -200 + Math.random() * 400,
            z: -200 + Math.random() * 400
        })

    })

    // 视角
    gsap.to(camera, t, {
        animAngle: camera.animAngle + (2 * Math.random() - 1) * Math.PI,
        onUpdate: function () {
            camera.position.x = Math.cos(camera.animAngle) * 440;
            camera.position.z = Math.sin(camera.animAngle) * 440;
            camera.updateProjectionMatrix();
            camera.lookAt(scene.position);
        }
    })

    const intens =  Math.random() 
    
    // 灯光强度
    gsap.to(light, t * 1.5, {
        intensity: intens * 3,
    })

    gsap.to(light2, t * 1.5, {
        intensity: intens * 3
    })

    gsap.to(alight, t * 1.5, {
        intensity: intens
    })

    gsap.to(window, 2.5, {
        onComplete: loop
    })
}

animate()

window.onresize = () => {
    camera.aspect = window.innerWidth / window.innerHeight
    camera.updateProjectionMatrix()
    renderer.setSize(window.innerWidth, window.innerHeight)
}

animate()
function animate() {
    requestAnimationFrame(animate)
    renderer.render(scene, camera)
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/gsapCollection.js)

## 小结

- 本文提供 **动画合集** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

