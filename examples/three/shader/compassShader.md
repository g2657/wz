---
title: "Three.js 罗盘教程"
description: "详解 Three.js 罗盘：基于 WebGL 实现「罗盘」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,罗盘,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 罗盘 · *Compass Shader* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=compassShader)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![罗盘](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/compassShader.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **罗盘** 效果：基于 WebGL 实现「罗盘」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 0, 0.6)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

window.onresize = () => {

  renderer.setSize(box.clientWidth, box.clientHeight)

  camera.aspect = box.clientWidth / box.clientHeight

  camera.updateProjectionMatrix()

}

const uniforms = {

  iTime: {

    value: 0

  },

  iResolution: {

    value: new THREE.Vector2(box.clientWidth, box.clientHeight)

  }
  
}

const geometry = new THREE.PlaneGeometry(1, 1)

const material = new THREE.ShaderMaterial({

  uniforms,

  transparent: true,

  side: THREE.DoubleSide,

  vertexShader: `
      varying vec3 vPosition;
      varying vec2 vUv;
      void main() { 
          vUv = uv; 
          vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
          gl_Position = projectionMatrix * mvPosition;
      }
  `,
  fragmentShader: `
  uniform float ratio;

  float PI = 3.1415926;
  uniform float iTime;
  uniform vec2 iResolution; 
  varying vec2 vUv;
  
  vec2 rotate(vec2 p, float rad) {
      mat2 m = mat2(cos(rad), sin(rad), -sin(rad), cos(rad));
      return m * p;
  }
  
  vec2 translate(vec2 p, vec2 diff) {
      return p - diff;
  }
  
  vec2 scale(vec2 p, float r) {
      return p*r;
  }
  
  float circle(float pre, vec2 p, float r1, float r2, float power) {
      float leng = length(p);
      float d = min(abs(leng-r1), abs(leng-r2));
      if (r1<leng && leng<r2) pre /= exp(d)/r2;
      float res = power / d;
      return clamp(pre + res, 0.0, 1.0);
  }
  
  float rectangle(float pre, vec2 p, vec2 half1, vec2 half2, float power) {
      p = abs(p);
      if ((half1.x<p.x || half1.y<p.y) && (p.x<half2.x && p.y<half2.y)) {
          pre = max(0.01, pre);
      }
      float dx1 = (p.y < half1.y) ? abs(half1.x-p.x) : length(p-half1);
      float dx2 = (p.y < half2.y) ? abs(half2.x-p.x) : length(p-half2);
      float dy1 = (p.x < half1.x) ? abs(half1.y-p.y) : length(p-half1);
      float dy2 = (p.x < half2.x) ? abs(half2.y-p.y) : length(p-half2);
      float d = min(min(dx1, dx2), min(dy1, dy2));
      float res = power / d;
      return clamp(pre + res, 0.0, 1.0);
  }
  float radiation(float pre, vec2 p, float r1, float r2, int num, float power) {
      float angle = 2.0*PI/float(num);
      float d = 1e10;
      for(int i=0; i<360; i++) {
          if (i>=num) break;
          float _d = (r1<p.y && p.y<r2) ? 
              abs(p.x) : 
              min(length(p-vec2(0.0, r1)), length(p-vec2(0.0, r2)));
          d = min(d, _d);
          p = rotate(p, angle);
      }
      float res = power / d;
      return clamp(pre + res, 0.0, 1.0);
  }
  vec3 calc(vec2 p) {
      float dst = 0.0;
      p = scale(p, sin(PI*iTime/1.0)*0.02+1.1);
      {
          vec2 q = p;
          q = rotate(q, iTime * PI / 6.0);
          dst = circle(dst, q, 0.85, 0.9, 0.006);
          dst = radiation(dst, q, 0.87, 0.88, 36, 0.0008);
      }
      {
          vec2 q = p;
          q = rotate(q, iTime * PI / 6.0);
          const int n = 6;
          float angle = PI / float(n);
          q = rotate(q, floor(atan(q.x, q.y)/angle + 0.5) * angle);
          for(int i=0; i<n; i++) {
              dst = rectangle(dst, q, vec2(0.85/sqrt(2.0)), vec2(0.85/sqrt(2.0)), 0.0015);
              q = rotate(q, angle);
          }
      }
      {
          vec2 q = p;
          q = rotate(q, iTime * PI / 6.0);
          const int n = 12;
          q = rotate(q, 2.0*PI/float(n)/2.0);
          float angle = 2.0*PI / float(n);
          for(int i=0; i<n; i++) {
              dst = circle(dst, q-vec2(0.0, 0.875), 0.001, 0.05, 0.004);
              dst = circle(dst, q-vec2(0.0, 0.875), 0.001, 0.001, 0.008);
              q = rotate(q, angle);
          }
      }
      {
          vec2 q = p;
          dst = circle(dst, q, 0.5, 0.55, 0.002);
      }
      {
          vec2 q = p;
          q = rotate(q, -iTime * PI / 6.0);
          const int n = 3;
          float angle = PI / float(n);
          q = rotate(q, floor(atan(q.x, q.y)/angle + 0.5) * angle);
          for(int i=0; i<n; i++) {
              dst = rectangle(dst, q, vec2(0.36, 0.36), vec2(0.36, 0.36), 0.0015);
              q = rotate(q, angle);
          }
      }
      {
          vec2 q = p;
          q = rotate(q, -iTime * PI / 6.0);
          const int n = 12;
          q = rotate(q, 2.0*PI/float(n)/2.0);
          float angle = 2.0*PI / float(n);
          for(int i=0; i<n; i++) {
              dst = circle(dst, q-vec2(0.0, 0.53), 0.001, 0.035, 0.004);
              dst = circle(dst, q-vec2(0.0, 0.53), 0.001, 0.001, 0.001);
              q = rotate(q, angle);
          }
      }
      {
          vec2 q = p;
          q = rotate(q, iTime * PI / 6.0);
          dst = radiation(dst, q, 0.25, 0.3, 12, 0.005);
      }
      {
          vec2 q = p;
          q = scale(q, sin(PI*iTime/1.0)*0.04+1.1);
          q = rotate(q, -iTime * PI / 6.0);
          for(float i=0.0; i<6.0; i++) {
              float r = 0.13-i*0.01;
              q = translate(q, vec2(0.1, 0.0));
              dst = circle(dst, q, r, r, 0.002);
              q = translate(q, -vec2(0.1, 0.0));
              q = rotate(q, -iTime * PI / 12.0);
          }
          dst = circle(dst, q, 0.04, 0.04, 0.004);
      }
      return pow(dst, 2.5) * vec3(1.0, 0.95, 0.8);
  }
  void main() { 
      vec2 uv = (vUv - 0.5) * 2.0;
      gl_FragColor = vec4(calc(uv), 1.0);;
      
  }
    `
})

const mesh = new THREE.Mesh(geometry, material)

scene.add(mesh)

animate()

function animate() {

  uniforms.iTime.value += 0.01

  requestAnimationFrame(animate)

  controls.update()

  renderer.render(scene, camera)

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/compassShader.js)

## 小结

- 本文提供 **罗盘** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

