---
title: "Three.js Mesh变换动画教程"
description: "详解 Three.js Mesh变换动画：基于 WebGL 实现「Mesh变换动画」可视化效果，附完整可运行源码，涵盖 OrbitControls、FBXLoader、GSAP 等关键实现，附完整源码与在线 Demo，适合 Three.js 动画 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,Mesh变换动画,WebGL,源码,教程,在线案例,OrbitControls,相机控制,FBX,模型加载,GSAP,动画"
outline: deep
---

### Mesh变换动画 · *Transform Gsap* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=animation&id=transformAnimate)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![Mesh变换动画](https://z2586300277.github.io/three-cesium-examples/threeExamples/basic/transformAnimate.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- FBXLoader 加载 FBX 城市/角色模型
- GSAP 时间轴与补间动画
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **Mesh变换动画** 效果：基于 WebGL 实现「Mesh变换动画」可视化效果，附完整可运行源码；核心用到 OrbitControls、FBXLoader、GSAP。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- 阅读下方完整源码时，建议从 `init` / `load` / `animate` 三条主线入手，再深入 shader 与工具函数。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 异步加载模型 / 3D Tiles / GeoJSON 等资源并加入 scene 或 entities
3. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
4. 在定时器或 GSAP 时间轴中更新 uniform / 变换，驱动特效播放
5. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { FBXLoader } from 'three/examples/jsm/loaders/FBXLoader.js'
import gsap from 'gsap'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(10, 10, 10)

const renderer = new THREE.WebGLRenderer()

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

animate()

function animate() {

    renderer.render(scene, camera)

    requestAnimationFrame(animate)

}

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

}

scene.add(new THREE.AxesHelper(100000), new THREE.AmbientLight(0xffffff, 6))

// 加载模型
new FBXLoader().load(HOST + '/files/model/city.FBX', (object3d) => {

    scene.add(object3d)

    object3d.scale.set(0.0005, 0.0005, 0.0005)

    setTransformAnimate(object3d)

})

// 变换动画
function setTransformAnimate(mesh) {

    const position = mesh.position.clone()
    
    position.y += 5 // 位置向上移动100

    const rotation = mesh.rotation.clone()

    rotation.y += Math.PI / 4 // 旋转45度

    const scale = mesh.scale.clone()
    
    scale.z *= 2 // 缩放z轴2倍

    scale.x *= 2 // 缩放x轴2倍

    // 组合参数
    const transformInfo_ = { position, rotation, scale } 

    // 执行
    const promises_gsap = ['position', 'rotation', 'scale'].map(i => {

        return new Promise(resolve => {
    
            gsap.to(mesh[i], {
    
                x: transformInfo_[i].x,
    
                y: transformInfo_[i].y,
    
                z: transformInfo_[i].z,
    
                //间隔时间
                duration: 3,
    
                //动画参数名
                ease: 'none',
    
                //重复次数
                repeat: 0,
    
                //往返移动
                yoyo: false,
    
                yoyoEase: true,
    
                onComplete: resolve
    
            })
    
        })
    
    })
    
    Promise.all(promises_gsap).then(() => console.log('动画完成'))

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/basic/transformAnimate.js)

## 小结

- 本文提供 **Mesh变换动画** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

