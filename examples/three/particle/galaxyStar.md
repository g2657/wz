---
title: "Three.js 星系教程"
description: "详解 Three.js 星系：基于 WebGL 实现「星系」可视化效果，附完整可运行源码，涵盖 OrbitControls、THREE.Points、BufferGeometry 等关键实现，附完整源码与在线 Demo，适合 Three.js 粒子 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,星系,WebGL,源码,教程,在线案例,OrbitControls,相机控制,粒子特效,Points,BufferGeometry"
outline: deep
---

### 星系 · *Galaxy Star* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=particle&id=galaxyStar)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![星系](https://z2586300277.github.io/three-cesium-examples/threeExamples/particle/galaxyStar.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- THREE.Points 粒子点渲染
- BufferGeometry 自定义顶点/索引数据
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **星系** 效果：基于 WebGL 实现「星系」可视化效果，附完整可运行源码；核心用到 OrbitControls、THREE.Points、BufferGeometry。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
import * as THREE from 'three';
import { GUI } from "three/addons/libs/lil-gui.module.min.js";
import Stats from 'three/examples/jsm/libs/stats.module.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

const initializeScene = ({ root, antialias = true } = {}) => {
  // Create scene
  const scene = new THREE.Scene();

  // Create camera
  const camera = new THREE.PerspectiveCamera(
    35,
    window.innerWidth / window.innerHeight,
    0.1,
    1000,
  );
  camera.position.z = 110;

  // Create renderer
  const renderer = new THREE.WebGLRenderer({ antialias });
  renderer.setSize(window.innerWidth, window.innerHeight);
  // renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  root.appendChild(renderer.domElement);

  const onWindowResize = () => {
    // Adjust camera and renderer on window resize
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    controls.update();
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.render(scene, camera);
  };
  onWindowResize();
  window.addEventListener('resize', onWindowResize, false);

  // Create GUI
  const gui = new GUI({ container: root });

  const stats = new Stats();
  stats.showPanel(0);
  root.appendChild(stats.domElement);

  return {
    scene,
    renderer,
    camera,
    controls,
    gui,
    stats,
  };
}
const getRandomPolarCoordinate = (radius) => {
  const theta = Math.random() * Math.PI * 2;
  const phi = Math.random() * Math.PI * 2;
  const x = radius * Math.sin(theta) * Math.cos(phi);
  const y = radius * Math.sin(theta) * Math.sin(phi);
  const z = radius * Math.cos(theta);
  return { x, y, z };
}

const init = (root) => {
  const params = {
    particleCount: 250000,
    particleSize: 0.02,
    branches: 6,
    branchRadius: 5,
    spin: 0.2,
    radialRandomness: 0.5,
    innerColor: '#ff812e',
    outerColor: '#a668ff',
  };

  const { scene, renderer, camera, gui, stats, controls } = initializeScene({
    root,
  });

  camera.position.set(7, 4, 7);
  controls.update();

  let spinDirection = 1;
  let material = null;
  let geometry = null;
  let points = null;

  const particleTexture = new THREE.TextureLoader().load(FILE_HOST + 'threeExamples/shader/star.png');

  const generateGalaxy = () => {
    // Remove old particles
    if (points) {
      geometry.dispose();
      material.dispose();
      scene.remove(points);
    }

    // Create new particles
    const positions = new Float32Array(params.particleCount * 3);
    const colors = new Float32Array(params.particleCount * 3);
    const innerColor = new THREE.Color(params.innerColor);
    const outerColor = new THREE.Color(params.outerColor);
    for (let i = 0; i < params.particleCount; i++) {
      const i3 = i * 3;

      const radius = params.branchRadius * Math.random();
      const branchAngle =
        ((i % params.branches) / params.branches) * Math.PI * 2;
      const spinAngle = params.spin * radius * Math.PI * 2;

      const randRadius = Math.random() * params.radialRandomness * radius;
      const {
        x: randX,
        y: randY,
        z: randZ,
      } = getRandomPolarCoordinate(randRadius);

      positions[i3] = radius * Math.cos(branchAngle + spinAngle) + randX;
      positions[i3 + 1] = randY;
      positions[i3 + 2] = radius * Math.sin(branchAngle + spinAngle) + randZ;

      const mixedColor = innerColor
        .clone()
        .lerp(outerColor, radius / params.branchRadius);
      colors[i3] = mixedColor.r;
      colors[i3 + 1] = mixedColor.g;
      colors[i3 + 2] = mixedColor.b;
    }

    material = new THREE.PointsMaterial({
      size: params.particleSize,
      sizeAttenuation: true,
      depthWrite: false,
      blending: THREE.AdditiveBlending,
      vertexColors: true,
      transparent: true,
      alphaMap: particleTexture,
    });
    geometry = new THREE.BufferGeometry();
    geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));

    points = new THREE.Points(geometry, material);
    scene.add(points);

    spinDirection = params.spin > 0 ? 1 : -1;
  };

  generateGalaxy();

  // Create GUI
  gui.width = 360;
  gui
    .add(params, 'particleCount', 5000, 500000, 100)
    .onFinishChange(generateGalaxy);
  gui.add(params, 'particleSize', 0.005, 0.15).onFinishChange(generateGalaxy);
  gui.add(params, 'branches', 2, 15, 1).onFinishChange(generateGalaxy);
  gui.add(params, 'branchRadius', 1, 10).onFinishChange(generateGalaxy);
  gui.add(params, 'spin', -1, 1).onFinishChange(generateGalaxy);
  gui.add(params, 'radialRandomness', 0, 1).onFinishChange(generateGalaxy);
  gui.addColor(params, 'innerColor').onFinishChange(generateGalaxy);
  gui.addColor(params, 'outerColor').onFinishChange(generateGalaxy);

  const tick = () => {
    requestAnimationFrame(tick);
    stats.begin();

    controls.update();

    geometry.rotateY(0.001 * spinDirection);

    stats.end();
    renderer.render(scene, camera);
  };

  tick();
};

init(document.getElementById('box'));
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/particle/galaxyStar.js)

## 小结

- 本文提供 **星系** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

