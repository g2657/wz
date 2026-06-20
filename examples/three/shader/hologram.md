---
title: "Three.js 全息投影教程"
description: "详解 Three.js 全息投影：基于 WebGL 实现「全息投影」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls、glTF/Draco 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,全息投影,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制,glTF"
outline: deep
---

### 全息投影 · *Hologram* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=hologram)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![全息投影](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/hologram.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- glTF/Draco 模型加载与优化
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **全息投影** 效果：基于 WebGL 实现「全息投影」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls、glTF/Draco。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 异步加载模型 / 3D Tiles / GeoJSON 等资源并加入 scene 或 entities
3. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
4. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
5. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
/**
 * 全息投影
 */
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";

/**
 * Base
 */


// Canvas
const canvas = document.querySelector("canvas.webgl");

// Scene
const scene = new THREE.Scene();

/**
 * Loaders
 */
const gltfLoader = new GLTFLoader();
const vertexShader=`
#include <common>
precision mediump float;

uniform float uTime;

varying vec2 vUv;
varying vec3 vPosition;
varying vec3 vNormal;
float random (in vec2 _st) {
    return fract(sin(dot(_st.xy,
                         vec2(12.9898,78.233)))*
        43758.5453123);
}


void main(){
    /**
    * Position
    */
    vec4 modelPosition=modelMatrix*vec4(position,1.);

    // 实现
    float gltichTime = modelPosition.y-uTime;
    // 正弦波的叠加，非规律
    float gltichStrength=sin(gltichTime)+sin(gltichTime*3.45)-sin(gltichTime*8.5);
    gltichTime=gltichTime/3.0;
    // 平滑step
    gltichStrength=smoothstep(0.3,1.0 ,gltichStrength );
    // 随着时间变化的随机数种子，生成随机偏移量。 
    modelPosition.x+=(random(modelPosition.xz+uTime)-0.5)*gltichStrength*0.25;
    modelPosition.z+=(random(modelPosition.zx+uTime)-0.5)*gltichStrength*0.25;

       // 计算法线变换矩阵：逆矩阵并转置
     mat3 normalMatrix = transpose(inverse(mat3(modelMatrix)));
    // 使用法线变换矩阵变换法线
    vec3 transformedNormal = normalize(normalMatrix * normal);
    // vec3 transformedNormal = normal;



    
    vec4 viewPosition=viewMatrix*modelPosition;
    vec4 projectedPosition=projectionMatrix*viewPosition;
    gl_Position=projectedPosition;

    vPosition=modelPosition.xyz;
    vNormal=transformedNormal;
    
}
`
const fragmentShader=`

precision mediump float;
varying vec2 vUv;
uniform float uTime;
varying vec3 vPosition;
varying vec3 vNormal;

void main(){
  float stripes=vPosition.y-uTime*0.02;
  stripes=mod(stripes*20.0,1.0);
  stripes=pow(stripes,3.0 );



  // gl_FragColor=vec4(1.0,1.0,1.0,stripes);
  vec3 normal =normalize(vNormal);

  // 让背面的法向量也朝向观察者，保持与正面一致
  if(!gl_FrontFacing)
      normal*=-1.0;

  vec3 viewDirection=normalize(vPosition-cameraPosition);
  float fresnel=dot(viewDirection,normal);
  fresnel=fresnel+1.0;
  fresnel=pow(fresnel,2.0 );

  //falloff
  float falloff = smoothstep(0.8,0.0 ,fresnel );

  float holographic=fresnel*stripes;
  holographic+=fresnel*1.25;
  holographic*=falloff;

// 注意开启材质透明
  gl_FragColor=vec4(0.7,0.25,0.8,holographic);

  // 引入three.js的内置shader代码。开启toneMapping和colorSpace
  #include <tonemapping_fragment>
  #include <colorspace_fragment>
  
}
`
// Textures
const material = new THREE.ShaderMaterial({
  transparent:true,
  side:THREE.DoubleSide,
  vertexShader,
  fragmentShader,
  blending:THREE.AdditiveBlending,
  uniforms: {
    uTime: new THREE.Uniform(0),
  },
});
/**
 * Models
 */
let suzanne = null;
gltfLoader.load("https://coderfmc.github.io/three.js-demo/suzanne.glb", (gltf) => {
  // Model
  gltf.scene.traverse((item) => {
    if (item.isMesh) {
      item.material = material;
      suzanne = item;
      scene.add(item);
    }
  });
});
/**
 * Objects
 */
// Torus knot
const torusKnot = new THREE.Mesh(
  new THREE.TorusKnotGeometry(0.6, 0.25, 128, 32),
  material
);
torusKnot.position.x = 3;
scene.add(torusKnot);

// Sphere
const sphere = new THREE.Mesh(new THREE.SphereGeometry(), material);
sphere.position.x = -3;
scene.add(sphere);

/**
 * Sizes
 */
const sizes = {
  width: window.innerWidth,
  height: window.innerHeight,
};

window.addEventListener("resize", () => {
  // Update sizes
  sizes.width = window.innerWidth;
  sizes.height = window.innerHeight;

  // Update camera
  camera.aspect = sizes.width / sizes.height;
  camera.updateProjectionMatrix();

  // Update renderer
  renderer.setSize(sizes.width, sizes.height);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
});

/**
 * Camera
 */
// Base camera
const camera = new THREE.PerspectiveCamera(
  75,
  sizes.width / sizes.height,
  0.1,
  100
);
camera.position.set(4, 1, -4);
scene.add(camera);
// scene.add(new THREE.AxesHelper(1, 1, 1));

/**
 * Renderer
 */
const renderer = new THREE.WebGLRenderer({
  antialias: true,
});
renderer.setSize(sizes.width, sizes.height);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
document.getElementById("box").appendChild(renderer.domElement);
// Controls
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
/**
 * Animate
 */
const clock = new THREE.Clock();

const tick = () => {
  const elapsedTime = clock.getElapsedTime();
  // Update controls
  controls.update();
  torusKnot.rotation.y = elapsedTime * 0.5;
  sphere.rotation.y = elapsedTime * 0.5;
  if (suzanne) {
    suzanne.rotation.y = elapsedTime * 0.5;
  }
  material.uniforms.uTime.value = elapsedTime;
  // Render
  renderer.render(scene, camera);

  // Call tick again on the next frame
  window.requestAnimationFrame(tick);
};

tick();
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/hologram.js)

## 小结

- 本文提供 **全息投影** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

