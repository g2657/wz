---
title: "Three.js 扭曲教程"
description: "详解 Three.js 扭曲：基于 WebGL 实现「扭曲」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,扭曲,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 扭曲 · *Vortex Shader* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=vortexShader)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![扭曲](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/vortexShader.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **扭曲** 效果：基于 WebGL 实现「扭曲」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
      vCenter: { type: "v2", value: new THREE.Vector2(window.innerWidth / 2, window.innerHeight / 2) },
      fRadius: { type: "f", value: window.innerWidth / 2.0 },
      fUzuStrength: { type: "f", value: 2.0 },
    },
    vertexShader: `
    varying vec2 vUv;
    void main(){
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
    `,
    fragmentShader: `
    uniform sampler2D tDiffuse;
    varying vec2 vUv;
    uniform vec2 vScreenSize;
    uniform vec2 vCenter;
    uniform float fRadius;
    uniform float fUzuStrength;

    void main() {
      vec2 pos = (vUv * vScreenSize) - vCenter;
      float len = length(pos);
      if (len >= fRadius) {
        gl_FragColor = texture2D(tDiffuse, vUv);
        return;
      }

      float uzu = min(max(1.0 - (len / fRadius), 0.0), 1.0) * fUzuStrength;
      float x = pos.x * cos(uzu) - pos.y * sin(uzu);
      float y = pos.x * sin(uzu) + pos.y * cos(uzu);
      vec2 retPos = (vec2(x, y) + vCenter) / vScreenSize;
      vec4 color = texture2D(tDiffuse, retPos);
      gl_FragColor = color;
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
  gui.add(cubeMaterial.uniforms['fRadius'], 'value', 1.0, 1000.0).step(1.0).name('radius');
  gui.add(cubeMaterial.uniforms['fUzuStrength'], 'value', -4.0, 4.0).step(0.1).name('uzuStrength');
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/vortexShader.js)

## 小结

- 本文提供 **扭曲** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

