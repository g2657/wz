---
title: "Three.js UV图像变换教程"
description: "详解 Three.js UV图像变换：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期，涵盖 EffectComposer、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 后期处理 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,UV图像变换,WebGL,源码,教程,在线案例,EffectComposer,后期处理,OrbitControls,相机控制"
outline: deep
---
### UV图像变换 · *UV Transform* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=effectComposer&id=uvTransformation)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![UV图像变换](https://z2586300277.github.io/3d-file-server/images/four/uvTransformation.png)

## 你将学到什么

- EffectComposer 后期处理管线
- 相机交互控制器
- 轮廓高亮 OutlinePass
- requestAnimationFrame 渲染循环

## 效果说明

本案例演示 **UV图像变换** 效果：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期；核心用到 EffectComposer、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **EffectComposer** 多 Pass 链式渲染：RenderPass → 特效 Pass → 输出屏幕。`composer.render()` 替代 `renderer.render()`。

- **OrbitControls** 轨道旋转缩放；开 `enableDamping` 时每帧需 `controls.update()`。

- 选中物体外轮廓发光，常用于编辑器选中态。

## 实现步骤

1. 搭建 Scene / Camera / Renderer 与 OrbitControls
2. EffectComposer 组装 Pass 链并 render

## 代码要点

```js
import * as THREE from "three";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
import { RenderPass, EffectPass, EffectComposer, GodRaysEffect } from 'postprocessing'
// Max Muselmann https://unsplash.com/photos/oIVvGqqwVJw
const image_url =
  FILE_HOST + "images/four/photo-1583766395091-2eb9994ed094.avif";
const image_ratio = 687 / 1031;
const image_tex = new THREE.TextureLoader().load(image_url);
image_tex.repeat.set(1, 1);

// Daniil Silantev https://unsplash.com/photos/dxGTQArsC3M
const alpha_url =
  FILE_HOST + "images/four/photo-1510942752400-ebce99a8a2c0.avif";
const alpha_tex = new THREE.TextureLoader().load(alpha_url);
alpha_tex.repeat.set(0.6, 2);
alpha_tex.offset.x = (1 - alpha_tex.repeat.x) / 2;
alpha_tex.wrapT = THREE.RepeatWrapping;

// ----
// main
// ----

const renderer = new THREE.WebGLRenderer();
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, 2, 0.1, 100);
const controls = new OrbitControls(camera, renderer.domElement);

camera.position.set(0, 0, 1);
controls.enableDamping = true;

const light = new THREE.DirectionalLight();
light.position.set(0, 0, 1);
scene.add(light);

const geom = new THREE.PlaneGeometry(image_ratio * 2, 2);
const mat = new THREE.MeshLambertMaterial({
  alphaMap: alpha_tex,
  alphaTest: 0.15,
  map: image_tex,
});
const mesh = new THREE.Mesh(geom, mat);
scene.add(mesh);

const wall = new THREE.Mesh(
  geom,
  new THREE.MeshBasicMaterial({
    alphaMap: alpha_tex,
    alphaTest: 0.15,
    map: image_tex,
  })
);
wall.scale.setScalar(1.2);
wall.position.z = -0.1;
scene.add(wall);

// ----
// render
// ----

const composer = new EffectComposer(renderer);
const renderPass = new RenderPass(scene, camera);
const effect = new GodRaysEffect(camera, wall, {
  density: 1,
  decay: 0.96,
  weight: 1,
});
const effectPass = new EffectPass(camera, effect);
composer.addPass(renderPass);
composer.addPass(effectPass);

renderer.setAnimationLoop((t) => {
  composer.render();
  controls.update();
  alpha_tex.offset.y = t * -0.001;
});

// ----
// view
// ----

function resize(w, h, dpr = devicePixelRatio) {
  renderer.setPixelRatio(dpr);
  // renderer.setSize(w, h, false)
  composer.setSize(w, h, false);
  camera.aspect = w / h;
  camera.updateProjectionMatrix();
}
addEventListener("resize", () => resize(innerWidth, innerHeight));
dispatchEvent(new Event("resize"));
document.body.prepend(renderer.domElement);
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/effectComposer/uvTransformation.js)

## 小结

- 本文提供 **UV图像变换** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

