---
title: "Three.js 等值线教程"
description: "详解 Three.js 等值线：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理，涵盖 OrbitControls、Canvas 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,等值线,WebGL,源码,教程,在线案例,OrbitControls,相机控制,Canvas纹理,动态贴图"
outline: deep
---
### 等值线 · *Isoline* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=isoline)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![等值线](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/isoline.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- Canvas 动态纹理贴图
- 监听窗口 `resize` 同步更新 camera 与 renderer

## 效果说明

本案例演示 **等值线** 效果：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理；核心用到 OrbitControls、Canvas。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **OrbitControls** 轨道旋转缩放；开 `enableDamping` 时每帧需 `controls.update()`。

- **CubeTexture** 六面贴图作 `scene.background`；`scene.environment` 供 PBR 材质反射。

## 实现步骤

1. 搭建 Scene / Camera / Renderer 与 OrbitControls
2. rAF 循环中 update 并 render

## 代码要点

```js
import * as THREE from 'three';
import { OrbitControls } from "three/examples/jsm/Addons.js";
import { SimplexNoise } from 'three/examples/jsm/math/SimplexNoise.js';
const DOM = document.getElementById('box')

var scene = new THREE.Scene();
scene.background = new THREE.Color('gainsboro');

var camera = new THREE.PerspectiveCamera(30, innerWidth / innerHeight);
camera.position.set(0, 4, 4);
camera.lookAt(scene.position);

var renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setAnimationLoop(animationLoop);
DOM.appendChild(renderer.domElement);

window.addEventListener("resize", () => {
    camera.aspect = innerWidth / innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(innerWidth, innerHeight);
});

var controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.autoRotate = true;

var light = new THREE.DirectionalLight('white', 3);
light.position.set(1, 1, 1);
scene.add(light);


// next comment

// a texture with isolines
var canvas = document.createElement('CANVAS');
canvas.width = 16;
canvas.height = 128;

var context = canvas.getContext('2d');
context.fillStyle = 'royalblue';
context.fillRect(0, 0, 16, 128);
context.fillStyle = 'white';
context.fillRect(0, 0, 16, 6);

var isoTexture = new THREE.CanvasTexture(canvas);
isoTexture.repeat.set(1, 10);
isoTexture.wrapS = THREE.RepeatWrapping;
isoTexture.wrapT = THREE.RepeatWrapping;


// some terrain with simlex noise
// reference https://codepen.io/boytchev/full/gOQQRLd
var geometry = new THREE.PlaneGeometry(6, 4, 150, 100),
    pos = geometry.getAttribute('position'),
    uv = geometry.getAttribute('uv'),
    simplex = new SimplexNoise();

for (var i = 0; i < pos.count; i++) {
    var x = pos.getX(i),
        y = pos.getY(i),
        z = 0.4 * simplex.noise(x, y);

    pos.setZ(i, z);
    uv.setXY(i, 0, z);
}

geometry.computeVertexNormals();

var terrain = new THREE.Mesh(
    geometry,
    new THREE.MeshPhysicalMaterial({
        roughness: 0.5,
        metalness: 0.2,
        side: THREE.DoubleSide,
        map: isoTexture,
    })
);
terrain.rotation.x = -Math.PI / 2;
scene.add(terrain);
function animationLoop() {
    controls.update();
    light.position.copy(camera.position);
    renderer.render(scene, camera);
}
// test robot test robot test
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/isoline.js)

## 小结

- 本文提供 **等值线** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

