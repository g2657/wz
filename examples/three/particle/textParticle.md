---
title: "Three.js 文字采集成粒子教程"
description: "详解 Three.js 文字采集成粒子：基于 WebGL 实现「文字采集成粒子」可视化效果，附完整可运行源码，涵盖 OrbitControls、THREE.Points、BufferGeometry 等关键实现，附完整源码与在线 Demo，适合 Three.js 粒子 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,文字采集成粒子,WebGL,源码,教程,在线案例,OrbitControls,相机控制,粒子特效,Points,BufferGeometry"
outline: deep
---

### 文字采集成粒子 · *Text Particle* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=particle&id=textParticle)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![文字采集成粒子](https://z2586300277.github.io/three-cesium-examples/threeExamples/particle/textParticle.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- THREE.Points 粒子点渲染
- BufferGeometry 自定义顶点/索引数据
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **文字采集成粒子** 效果：基于 WebGL 实现「文字采集成粒子」可视化效果，附完整可运行源码；核心用到 OrbitControls、THREE.Points、BufferGeometry。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- **THREE.Points** 将每个顶点渲染为可控大小的粒子；可用自定义 attribute（如 `u_index`）驱动片元/顶点动画。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
3. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { FontLoader } from 'three/examples/jsm/loaders/FontLoader.js';
import { TextGeometry } from 'three/examples/jsm/geometries/TextGeometry.js';
import { MeshSurfaceSampler } from 'three/examples/jsm/math/MeshSurfaceSampler.js'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(50, box.clientWidth / box.clientHeight, 0.1, 1000)
camera.position.set(0, 0, 10)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })
renderer.setSize(box.clientWidth, box.clientHeight)
box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)
controls.enableDamping = true

const loader = new FontLoader()
loader.load(FILE_HOST + 'files/json/font.json', font => {

    const textGeometry = new TextGeometry(`
        Three.js
        Cesium.js
        Examples
          - star -
    `, {
        font,
        size: 1,
        depth: 0.5,
        curveSegments: 10,
        bevelEnabled: false,
        bevelThickness: 0.1,
        bevelSize: 0.1,
        bevelSegments: 5
    }).center()

    const mesh = new THREE.Mesh(textGeometry, new THREE.MeshBasicMaterial({ color: 0xffffff }));
    const sampler = new MeshSurfaceSampler(mesh).build();
    const positions = new Float32Array(30000);
    const colors = new Float32Array(30000);
    const samplePoint = new THREE.Vector3();
    const color = new THREE.Color();
    
    for (let i = 0; i < 10000; i++) {
        sampler.sample(samplePoint);
        positions.set([samplePoint.x, samplePoint.y, samplePoint.z], i * 3);
    
        // 随机生成颜色
        color.setHSL(Math.random(), 1.0, 0.5);
        colors.set([color.r, color.g, color.b], i * 3);
    }
    
    const pointsGeometry = new THREE.BufferGeometry();
    pointsGeometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    pointsGeometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));
    
    const pointsMaterial = new THREE.PointsMaterial({ size: 0.04, vertexColors: true });
    scene.add(new THREE.Points(pointsGeometry, pointsMaterial));

})

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
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/particle/textParticle.js)

## 小结

- 本文提供 **文字采集成粒子** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

