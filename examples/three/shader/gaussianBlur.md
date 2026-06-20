---
title: "Three.js 高斯模糊教程"
description: "详解 Three.js 高斯模糊：基于 WebGL 实现「高斯模糊」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,高斯模糊,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 高斯模糊 · *Gaussian Blur* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=gaussianBlur)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![高斯模糊](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/gaussianBlur.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **高斯模糊** 效果：基于 WebGL 实现「高斯模糊」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
3. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
4. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from "three";
import { GUI } from "three/addons/libs/lil-gui.module.min.js";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
import Stats from "three/addons/libs/stats.module.js";

var container;
var scene, camera, renderer;
var controls;
var stats;

var cubeMaterial;

init();
update();
createGUI();

function init() {
  container = document.getElementById('box');

  // scene
  scene = new THREE.Scene();

  // camera
  camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 1, 10000);
  camera.position.set(1, 1, 1);
  camera.target = new THREE.Vector3(0, 0, 0);
  scene.add(camera);

  // renderer
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(window.devicePixelRatio);
  container.appendChild(renderer.domElement);

  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.minDistance = 2;
  controls.maxDistance = 10;
  stats = new Stats();
  document.body.appendChild(stats.dom);

  // light
  initLight();

  // model
  initModel();

  // event
  window.addEventListener('resize', onWindowResize, false);
}

function initLight() {
  var light = new THREE.DirectionalLight(0xffffff);
  light.position.set(0, 200, 100);
  scene.add(light);
}

// model
function initModel() {
  const cubeShader = {
    uniforms: {
      tDiffuse: { type: 't', value: new THREE.TextureLoader().load(FILE_HOST + 'threeExamples/shader/dlam.jpg') },
      vScreenSize: { type: "v2", value: new THREE.Vector2(window.innerWidth, window.innerHeight) },
      fScale: { type: "float", value: 10 },
    },
    vertexShader: `
    varying vec2 vUv;
    void main(){
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
    `,
    fragmentShader: `
      varying vec2 vUv;
      uniform sampler2D tDiffuse;
      uniform vec2 vScreenSize;
      uniform float fScale;

      void main() {
        vec4 sum = vec4(0.0);
        float h = fScale/vScreenSize.x;
        float v = fScale/vScreenSize.y;

        //纵向高斯模糊
        sum += texture2D(tDiffuse, vec2(vUv.x, vUv.y - 4.0 * v)) * (0.051/2.0);
        sum += texture2D(tDiffuse, vec2(vUv.x, vUv.y - 3.0 * v)) * (0.0918/2.0);
        sum += texture2D(tDiffuse, vec2(vUv.x, vUv.y - 2.0 * v)) * (0.12245/2.0);
        sum += texture2D(tDiffuse, vec2(vUv.x, vUv.y - 1.0 * v)) * (0.1531/2.0);
        sum += texture2D(tDiffuse, vec2(vUv.x, vUv.y)) * (0.1633/2.0);
        sum += texture2D(tDiffuse, vec2(vUv.x, vUv.y + 1.0 * v)) * (0.1531/2.0);
        sum += texture2D(tDiffuse, vec2(vUv.x, vUv.y + 2.0 * v)) * (0.12245/2.0);
        sum += texture2D(tDiffuse, vec2(vUv.x, vUv.y + 3.0 * v)) * (0.0918/2.0);
        sum += texture2D(tDiffuse, vec2(vUv.x, vUv.y + 4.0 * v)) * (0.051/2.0);

        //横向高斯模糊
        sum += texture2D(tDiffuse, vec2(vUv.x - 4.0 * h, vUv.y)) * (0.051/2.0);
        sum += texture2D(tDiffuse, vec2(vUv.x - 3.0 * h, vUv.y)) * (0.0918/2.0);
        sum += texture2D(tDiffuse, vec2(vUv.x - 2.0 * h, vUv.y)) * (0.12245/2.0);
        sum += texture2D(tDiffuse, vec2(vUv.x - 1.0 * h, vUv.y)) * (0.1531/2.0);
        sum += texture2D(tDiffuse, vec2(vUv.x, vUv.y)) * (0.1633/2.0);
        sum += texture2D(tDiffuse, vec2(vUv.x + 1.0 * h, vUv.y)) * (0.1531/2.0);
        sum += texture2D(tDiffuse, vec2(vUv.x + 2.0 * h, vUv.y)) * (0.12245/2.0);
        sum += texture2D(tDiffuse, vec2(vUv.x + 3.0 * h, vUv.y)) * (0.0918/2.0);
        sum += texture2D(tDiffuse, vec2(vUv.x + 4.0 * h, vUv.y)) * (0.051/2.0);

        gl_FragColor = sum;
      }
    `,
  }

  cubeMaterial = new THREE.ShaderMaterial({
    uniforms: cubeShader.uniforms,
    vertexShader: cubeShader.vertexShader,
    fragmentShader: cubeShader.fragmentShader,
    side: THREE.DoubleSide
  });

  var geometry = new THREE.PlaneGeometry();
  var cube = new THREE.Mesh(geometry, cubeMaterial);
  scene.add(cube);
}

function update() {
  requestAnimationFrame(update);
  renderer.render(scene, camera);

  controls.update();
  stats.update();
}

function onWindowResize() {
  var w = window.innerWidth;
  var h = window.innerHeight;
  camera.aspect = w / h;
  camera.updateProjectionMatrix();

  renderer.setSize(w, h);
}

function createGUI() {
  var gui = new GUI();
  gui.add(cubeMaterial.uniforms['fScale'], 'value', 0.0, 10.0).step(1.0).name('fScale');
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/gaussianBlur.js)

## 小结

- 本文提供 **高斯模糊** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

