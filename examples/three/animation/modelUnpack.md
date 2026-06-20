---
title: "Three.js 模型拆解动画教程"
description: "详解 Three.js 模型拆解动画：基于 WebGL 实现「模型拆解动画」可视化效果，附完整可运行源码，涵盖 OrbitControls、glTF/Draco、GSAP 等关键实现，附完整源码与在线 Demo，适合 Three.js 动画 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,模型拆解动画,WebGL,源码,教程,在线案例,OrbitControls,相机控制,glTF,模型加载,GSAP,动画"
outline: deep
---

### 模型拆解动画 · *Model Unpack* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=animation&id=modelUnpack)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![模型拆解动画](https://z2586300277.github.io/three-cesium-examples/threeExamples/basic/modelUnpack.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- glTF/Draco 模型加载与优化
- GSAP 时间轴与补间动画
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **模型拆解动画** 效果：基于 WebGL 实现「模型拆解动画」可视化效果，附完整可运行源码；核心用到 OrbitControls、glTF/Draco、GSAP。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js'
import * as dat from 'dat.gui'
import gsap from 'gsap'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(5, 5, 5)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

}

animate()

function animate() {

    requestAnimationFrame(animate)

    renderer.render(scene, camera)

}

scene.add(new THREE.AmbientLight(0xffffff, 1))

const pointLight = new THREE.PointLight(0xffffff, 1.5, 0, 2)

pointLight.position.set(5, 5, 5)

scene.add(pointLight)

const directionalLight = new THREE.DirectionalLight(0xffffff, 2)

directionalLight.position.set(-5, 5, -5)

scene.add(directionalLight)

scene.add(new THREE.AxesHelper(1000))

let group

// 加载模型 gltf/ glb  draco解码器
const loader = new GLTFLoader()

loader.setDRACOLoader(new DRACOLoader().setDecoderPath(FILE_HOST + 'js/three/draco/'))

loader.load(

    HOST + '/files/model/car.glb',

    gltf => {

        group = gltf.scene

        const box = new THREE.Box3().setFromObject(group);

        const center = new THREE.Vector3();

        box.getCenter(center);

        group.center = center

        group.traverse((child) => {

            if (child.isMesh) {

                child.localToWorld(child.position)   //转为世界坐标

                child.startPoisition = child.position.clone()

            }

        })

        scene.add(group)

    }

)

// 拆解
const mergeHandle = (model) => {

    const distance = () => Math.random() > 0.5 ? 1.5 : -1.5

    model.traverse((child) => {

        if (child.startPoisition) {

            const v1 = distance()

            const v2 = distance()

            const v3 = distance()

            gsap.to(child.position, { x: child.startPoisition.x + v1, y: child.startPoisition.y + v2, z: child.startPoisition.z + v3, duration: 1, ease: 'power2.inOut' })

        }

    });
};

const gui = new dat.GUI()

gui.add({

    '拆解动画': () => {

        mergeHandle(group)
    }

}, '拆解动画')

gui.add({

    '还原动画': () => {

        group.traverse((child) => {

            if (child.startPoisition) {

                gsap.to(child.position, { x: child.startPoisition.x, y: child.startPoisition.y, z: child.startPoisition.z, duration: 1, ease: 'power2.inOut' })

            }

        });

    }

}, '还原动画')
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/basic/modelUnpack.js)

## 小结

- 本文提供 **模型拆解动画** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

