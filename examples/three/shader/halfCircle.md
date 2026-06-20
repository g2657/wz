---
title: "Three.js 半圆教程"
description: "详解 Three.js 半圆：基于 WebGL 实现「半圆」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,半圆,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 半圆 · *Half Circle* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=halfCircle)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![半圆](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/halfCircle.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **半圆** 效果：基于 WebGL 实现「半圆」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import Stats from 'three/examples/jsm/libs/stats.module.js'

var scene, camera, renderer, clock, controller, stats
var shader_material, rayMarchingFireMaterial, shaderMaterial
const vs = `
varying vec2 vUv;
varying vec3 vPosition;

void main(){
    vec4 modelPosition=modelMatrix*vec4(position,1.);
    vec4 viewPosition=viewMatrix*modelPosition;
    vec4 projectedPosition=projectionMatrix*viewPosition;
    gl_Position=projectedPosition;
    
    vUv=uv;
    vPosition=position;
}
`;

const fs = `
      #define PI 3.1415926535897932384626433832795
      varying vec2 vUv;

      uniform float uTime;

      vec2 rotatePoint(vec2 center,float angle,vec2 p) {
        float s = sin(angle);
        float c = cos(angle);

        // translate point back to origin:
        p.x -= center.x;
        p.y -= center.y;

        // rotate point
        float xNew = p.x * c - p.y * s;
        float yNew = p.x * s + p.y * c;

        // translate point back:
        p.x = xNew + center.x;
        p.y = yNew + center.y;
        return p;
      }

      float angleVec(vec2 a_, vec2 b_) {
          vec3 a = vec3(a_, 0);
          vec3 b = vec3(b_, 0);
          float dotProd = dot(a,b); 
          vec3 crossprod = cross(a,b);
          float crossprod_l = length(crossprod);
          float lenProd = length(a)*length(b);
          float cosa = dotProd/lenProd;
          float sina = crossprod_l/lenProd;
          float angle = atan(sina, cosa);
          
          if(dot(vec3(0,0,1), crossprod) < 0.0) 
              angle=90.0;
          return (angle * (180.0 / PI));
      }

      void main(){
        vec2 center = vec2(0.5, 0.5);
        float angleStela = 180.0;
        float radius = 0.5;
        float startAlpha = 0.0;
        float endAlpha = 0.5;
        vec3 color = vec3(0.0, 1.0, 0.0);
        float alpha = 0.0;
        
        float angle = (-uTime * 2.0);

        vec2 lineEnd =  vec2(center.x, center.y + radius);
        float distanceToCenter = distance(center, vUv.xy);	
        lineEnd = rotatePoint(center, angle, lineEnd);
        float angleStelaToApply = angleVec(normalize(lineEnd - center), normalize(vUv - center));
        if (angleStelaToApply < angleStela && distanceToCenter < radius) {
          float factorAngle = 1.0 - angleStelaToApply/angleStela;
          float finalFactorAngle = factorAngle * endAlpha;
          if (finalFactorAngle > startAlpha) {
            alpha = finalFactorAngle;
          }
        }

        gl_FragColor=vec4(color, alpha);
      }
      `

init();
animate();

// - Functions -
function init() {
  scene = new THREE.Scene();
  clock = new THREE.Clock();
  camera = new THREE.PerspectiveCamera(
    45,
    window.innerWidth / window.innerHeight,
    0.1,
    1000
  );
  camera.position.set(10, 10, 10)
  renderer = new THREE.WebGLRenderer({
    antialias: true, // 开启抗锯齿处理
    alpha: true,
  });

  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(window.devicePixelRatio)

  createRayMarchingFireMaterial();

  var axisHelper = new THREE.AxesHelper(10);
  scene.add(axisHelper);



  stats = new Stats()
  document.body.appendChild(stats.dom);

  // --------
  var geometry = new THREE.PlaneGeometry(5, 5);
  var cube = new THREE.Mesh(geometry, rayMarchingFireMaterial);
  scene.add(cube);
  // --------

  controller = new OrbitControls(camera, renderer.domElement);
  document.body.appendChild(renderer.domElement);
  window.onresize = onResize;
}

function createRayMarchingFireMaterial() {
  rayMarchingFireMaterial = new THREE.ShaderMaterial({
    transparent: true,
    vertexShader: vs,
    fragmentShader: fs,
    side: THREE.DoubleSide,
    uniforms: {
      uTime: {
        value: 0,
      },
      uMouse: {
        value: new THREE.Vector2(0, 0),
      },
      uResolution: {
        value: new THREE.Vector2(512, 512)
      },
      uVelocity: {
        value: 3,
      },
      uColor1: {
        value: new THREE.Color('#ff801a'),
      },
      uColor2: {
        value: new THREE.Color('#ff5718'),
      },
    },
  });
  shaderMaterial = rayMarchingFireMaterial;
}

function animate() {
  requestAnimationFrame(animate);
  renderer.render(scene, camera);
  stats.update();
  controller.update(clock.getDelta());
  if (rayMarchingFireMaterial) {
    rayMarchingFireMaterial.uniforms.uTime.value += 0.01
  }
}

function onResize() {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/halfCircle.js)

## 小结

- 本文提供 **半圆** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

