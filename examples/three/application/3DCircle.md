---
title: "Three.js 扩散半球教程"
description: "详解 Three.js 扩散半球：基于 WebGL 实现「扩散半球」可视化效果，附完整可运行源码，涵盖 OrbitControls、GSAP 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,扩散半球,WebGL,源码,教程,在线案例,OrbitControls,相机控制,GSAP,动画"
outline: deep
---
### 扩散半球 · *3D Circle* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=3DCircle)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![扩散半球](https://z2586300277.github.io/3d-file-server/images/four/3DCircle.png)

## 你将学到什么

- OrbitControls 相机轨道交互
- GSAP 时间轴与补间动画
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **扩散半球** 效果：基于 WebGL 实现「扩散半球」可视化效果，附完整可运行源码；核心用到 OrbitControls、GSAP。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **OrbitControls** 轨道旋转缩放；开 `enableDamping` 时每帧需 `controls.update()`。

- 属性插值动画，适合相机动效、UI 过渡。

## 实现步骤

1. 搭建 Scene / Camera / Renderer 与 OrbitControls
2. rAF 循环中 update 并 render

## 代码要点

```js
import * as THREE from "three";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
import TWEEN from "three/addons/libs/tween.module.js";

const scene = new THREE.Scene(); // 创建场景
const geometry = new THREE.BoxGeometry(10, 60, 100); //几何体
const material = new THREE.MeshLambertMaterial(); //材质
const mesh = new THREE.Mesh(geometry, material); //网格模型
mesh.position.set(0, 10, 0); //网格模型位置
// scene.add(mesh); //场景添加网格模型

// 3d半圆扩散 + 扩散波
let circle3D = scatter3DCircle(50);
circle3D.layers.enable(1);
circle3D.position.set(0, 10, 0);
scene.add(circle3D);

// AxesHelper
const axesHelper = new THREE.AxesHelper(150);
scene.add(axesHelper);

// 光源
const pointLight = new THREE.DirectionalLight(0xff00ff, 1.0); //颜色、强度
pointLight.position.set(200, 300, 400); //位置
scene.add(pointLight); //点光源添加到场景中

// 光源参考线
const dirLightHelper = new THREE.DirectionalLightHelper(
  pointLight,
  5,
  0xff0000
);
scene.add(dirLightHelper);

// 相机
const camera = new THREE.PerspectiveCamera(); //相机
camera.position.set(200, 200, 200); //相机位置
camera.lookAt(0, 10, 0); //相机观察位置

// 渲染器
const renderer = new THREE.WebGLRenderer(); // 创建渲染器

renderer.setSize(window.innerWidth, window.innerHeight); //渲染区域
renderer.render(scene, camera); //执行渲染
document.body.appendChild(renderer.domElement);

// 设置相机控件轨道控制器OrbitControls
new OrbitControls(camera, renderer.domElement);

// 圆扩散
function scatter3DCircle(r) {
  const geometry = new THREE.SphereGeometry(
    r,
    120,
    120,
    0,
    Math.PI * 2,
    0,
    Math.PI / 2
  );

  let circle = new THREE.Mesh(
    geometry,
    new THREE.MeshBasicMaterial({
      // side: THREE.DoubleSide,
      // transparent: true,
      // color:new THREE.Color("#ff0000")
      map: new THREE.TextureLoader().load(
        FILE_HOST + "images/four/gradual_red_01.png"
      ),
    })
  );

  return circle;
}
let s = 0,
  p = 0;
animate();

function animate() {
  requestAnimationFrame(animate);

  // animation
  if (s > 160) {
    (s = 0), (p = 160);
  }
  circle3D.scale.set(1 + s / 60, 1 + s / 80, 1 + s / 60);
  circle3D.material.opacity = p / 160;
  s++;
  p--;

  TWEEN.update();
  renderer.render(scene, camera);
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/3DCircle.js)

## 小结

- 本文提供 **扩散半球** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

