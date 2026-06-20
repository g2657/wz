---
title: "Three.js GSAP 动画教程"
description: "详解 Three.js GSAP 动画：基于 WebGL 实现「GSAP 动画」可视化效果，附完整可运行源码，涵盖 OrbitControls、GSAP 等关键实现，附完整源码与在线 Demo，适合 Three.js 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,GSAP 动画,WebGL,源码,教程,在线案例,OrbitControls,相机控制,GSAP,动画"
outline: deep
---

### GSAP 动画 · *GSAP Camera* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=basic&id=gsapAnimate)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![GSAP动画](https://z2586300277.github.io/three-cesium-examples/threeExamples/basic/gsapAnimate.jpg)

## 你将学到什么

- **gsap.to** 对 Vector3 的 x/y/z 插值
- 同时动画 **camera.position** 与 **controls.target**
- 与 OrbitControls 配合的 **运镜** 思路

## 效果说明

点击 GUI「播放」，相机从 (0,30,30) **2 秒内** 飞到 (20,20,20)，观察中心 target 移到 (-5,2,1)，形成平滑运镜。

## 核心概念

```js
function createGsapAnimation(position, targetPos) {
    return gsap.to(position, {
        ...targetPos,
        duration: 2,
        ease: 'none',
    });
}

// 同时动相机与轨道中心
createGsapAnimation(camera.position, { x: 20, y: 20, z: 20 });
createGsapAnimation(controls.target, { x: -5, y: 2, z: 1 });
```

rAF 里仍需 `controls.update()`（尤其 enableDamping 时）。

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import dat from 'dat.gui'
import gsap from 'gsap'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 30, 30)

const renderer = new THREE.WebGLRenderer({ antialias: true , alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

scene.add(new THREE.AxesHelper(1000))

scene.add(new THREE.GridHelper(100, 20))

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

// 环境贴图
const boxGeometry = new THREE.BoxGeometry(10, 10, 10);

const boxMaterial = new THREE.MeshBasicMaterial({ color: 'blue' });

const boxMesh = new THREE.Mesh(boxGeometry, boxMaterial);

scene.add(boxMesh);

new dat.GUI().add({fn: () => {

    // 创建一个相机动画
    createGsapAnimation(camera.position, { x: 20, y: 20, z: 20 })

    // 创建一个目标运动动画
    createGsapAnimation(controls.target, { x: -5, y: 2, z: 1 })

}}, 'fn').name('播放');

/* 视角动画 */
function createGsapAnimation(position, position_, gsapQuery = null) {

    //设置动画 x轴运动 持续时间
    return gsap.to(

        position,

        {

            ...position_,

            //间隔时间
            duration: 2,

            //动画参数名
            ease: 'none',

            //重复次数
            repeat: 0,

            //往返移动
            yoyo: false,

            yoyoEase: true,

            ...gsapQuery,

        }

    )

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/basic/gsapAnimate.js)

## 小结

- 本文提供 **GSAP 动画** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

