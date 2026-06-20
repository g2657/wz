---
title: "Three.js 地球粒子教程"
description: "详解 Three.js 地球粒子：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期，涵盖 ShaderMaterial、EffectComposer、UnrealBloomPass 等关键实现，附完整源码与在线 Demo，适合 Three.js 粒子 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,地球粒子,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,EffectComposer,后期处理,Bloom"
outline: deep
---

### 地球粒子 · *Globe Particle* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=particle&id=globeParticle)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![地球粒子](https://z2586300277.github.io/three-cesium-examples/threeExamples/particle/globeParticle.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- EffectComposer 多 Pass 后期处理管线
- UnrealBloomPass 辉光 Bloom 效果
- OrbitControls 相机轨道交互
- THREE.Points 粒子点渲染
- BufferGeometry 自定义顶点/索引数据
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **地球粒子** 效果：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期；核心用到 ShaderMaterial、EffectComposer、UnrealBloomPass。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **EffectComposer** 以多 Pass 链式渲染：RenderPass → 特效 Pass → 输出屏幕，替代直接 `renderer.render`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
3. 组装 EffectComposer Pass 链，在 animate 中调用 `composer.render()`
4. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
5. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from "three";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { RenderPass } from "three/addons/postprocessing/RenderPass.js";
import { ShaderPass } from "three/addons/postprocessing/ShaderPass.js";
import { UnrealBloomPass } from "three/addons/postprocessing/UnrealBloomPass.js";
import { GammaCorrectionShader } from "three/addons/shaders/GammaCorrectionShader.js";

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x020209);
scene.fog = new THREE.Fog(0x020209, 15, 60);

const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.z = 6;

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.2;
document.getElementById("box").appendChild(renderer.domElement);

const ambientLight = new THREE.AmbientLight(0x333366, 0.8);
scene.add(ambientLight);

const directionalLight = new THREE.DirectionalLight(0xffffcc, 1.2);
directionalLight.position.set(1, 3, 2);
scene.add(directionalLight);

const pointLight = new THREE.PointLight(0x3366ff, 2, 10);
pointLight.position.set(0, 0, 0);
scene.add(pointLight);

const composer = new EffectComposer(renderer);
const renderPass = new RenderPass(scene, camera);
composer.addPass(renderPass);

const bloomPass = new UnrealBloomPass(
  new THREE.Vector2(window.innerWidth, window.innerHeight),
  1.2,
  0.7,
  0.5
);
composer.addPass(bloomPass);

const gammaCorrectionPass = new ShaderPass(GammaCorrectionShader);
composer.addPass(gammaCorrectionPass);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.dampingFactor = 0.1;
controls.rotateSpeed = 0.5;
controls.minDistance = 3;
controls.maxDistance = 15;
controls.autoRotate = true;
controls.autoRotateSpeed = 0.5;

controls.addEventListener("start", () => {
  document.body.style.cursor = "grabbing";
  controls.autoRotate = false;
});

controls.addEventListener("end", () => {
  document.body.style.cursor = "grab";
  setTimeout(() => { controls.autoRotate = true; }, 3000);
});

const numParticles = 30000;
const geometry = new THREE.BufferGeometry();
const positions = new Float32Array(numParticles * 3);
const colors = new Float32Array(numParticles * 3);
const sizes = new Float32Array(numParticles);
const speeds = new Float32Array(numParticles);

for (let i = 0; i < numParticles; i++) {
  if (i < numParticles * 0.8) {
    const phi = Math.acos(-1 + (2 * i) / (numParticles * 0.8));
    const theta = Math.sqrt(numParticles * Math.PI) * phi;
    const radius = 1.8 + Math.random() * 0.4;
    const x = Math.sin(phi) * Math.cos(theta) * radius;
    const y = Math.sin(phi) * Math.sin(theta) * radius;
    const z = Math.cos(phi) * radius;
    positions[i * 3] = x;
    positions[i * 3 + 1] = y;
    positions[i * 3 + 2] = z;
  } else {
    const angle = Math.random() * Math.PI * 2;
    const radius = 2 + Math.random() * 3;
    const height = (Math.random() - 0.5) * 2;
    positions[i * 3] = Math.cos(angle) * radius;
    positions[i * 3 + 1] = height;
    positions[i * 3 + 2] = Math.sin(angle) * radius;
  }

  const distance = Math.sqrt(
    positions[i * 3] ** 2 +
    positions[i * 3 + 1] ** 2 +
    positions[i * 3 + 2] ** 2
  );

  let color;
  if (distance < 1.5) {
    color = new THREE.Color(0x4466ff);
  } else if (distance < 2.5) {
    color = new THREE.Color(0x9944ff);
  } else {
    color = new THREE.Color(0xff44aa);
  }

  color.offsetHSL(0, (Math.random() - 0.5) * 0.3, (Math.random() - 0.5) * 0.2);
  colors[i * 3] = color.r;
  colors[i * 3 + 1] = color.g;
  colors[i * 3 + 2] = color.b;

  sizes[i] = distance < 2 ? 0.05 + Math.random() * 0.04 : 0.03 + Math.random() * 0.03;
  speeds[i] = 0.4 + Math.random() * 0.6;
}

geometry.setAttribute("position", new THREE.BufferAttribute(positions, 3));
geometry.setAttribute("color", new THREE.BufferAttribute(colors, 3));
geometry.setAttribute("size", new THREE.BufferAttribute(sizes, 1));

const particleMaterial = new THREE.ShaderMaterial({
  uniforms: {
    time: { value: 0 },
    pixelRatio: { value: window.devicePixelRatio }
  },
  vertexShader: `
    attribute float size;
    uniform float time;
    uniform float pixelRatio;
    varying vec3 vColor;
    void main() {
      vColor = color;
      float pulse = 1.0 + 0.2 * sin(time + position.x + position.z);
      vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
      gl_PointSize = size * pulse * pixelRatio * (300.0 / -mvPosition.z);
      gl_Position = projectionMatrix * mvPosition;
    }
  `,
  fragmentShader: `
    varying vec3 vColor;
    void main() {
      float distanceToCenter = length(gl_PointCoord - vec2(0.5));
      if (distanceToCenter > 0.5) discard;
      float alpha = 1.0 - smoothstep(0.4, 0.5, distanceToCenter);
      gl_FragColor = vec4(vColor, alpha);
    }
  `,
  transparent: true,
  vertexColors: true,
  depthWrite: false,
  blending: THREE.AdditiveBlending
});

const particleSystem = new THREE.Points(geometry, particleMaterial);
scene.add(particleSystem);

window.addEventListener("resize", () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
  composer.setSize(window.innerWidth, window.innerHeight);
  particleMaterial.uniforms.pixelRatio.value = window.devicePixelRatio;
}, false);

renderer.domElement.addEventListener("dblclick", () => {
  camera.position.set(0, 0, 6);
  camera.lookAt(0, 0, 0);
  controls.reset();
});

const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  const delta = clock.getDelta();
  const elapsedTime = clock.getElapsedTime();
  particleSystem.rotation.y += delta * 0.05;
  particleMaterial.uniforms.time.value = elapsedTime;
  particleSystem.scale.x = 1 + Math.sin(elapsedTime * 0.3) * 0.05;
  particleSystem.scale.y = 1 + Math.sin(elapsedTime * 0.4) * 0.05;
  particleSystem.scale.z = 1 + Math.sin(elapsedTime * 0.5) * 0.05;
  controls.update();
  composer.render();
}

animate();
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/particle/globeParticle.js)

## 小结

- 本文提供 **地球粒子** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

