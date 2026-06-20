---
title: "Three.js 波扫描教程"
description: "详解 Three.js 波扫描：基于 WebGL 实现「波扫描」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,波扫描,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 波扫描 · *Wave Scan* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=waveScan)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![波扫描](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/waveScan.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **波扫描** 效果：基于 WebGL 实现「波扫描」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
  uniform float iTime;
  const float PI = 3.14159265359;


  float random(float p){
      return fract(sin(p) * 10000.0);
  } 
  
  float noise(vec2 p){
      float t = iTime / 2000.0;
      if(t > 1.0) t -= floor(t);
      return random(p.x * 14. + p.y * sin(t) * 0.5);
  }

  vec2 sw(vec2 p){
      return vec2(floor(p.x), floor(p.y));
  }
  
  vec2 se(vec2 p){
      return vec2(ceil(p.x), floor(p.y));
  }
  
  vec2 nw(vec2 p){
      return vec2(floor(p.x), ceil(p.y));
  }
  
  vec2 ne(vec2 p){
      return vec2(ceil(p.x), ceil(p.y));
  }

  float smoothNoise(vec2 p){
      vec2 inter = smoothstep(0.0, 1.0, fract(p));
      float s = mix(noise(sw(p)), noise(se(p)), inter.x);
      float n = mix(noise(nw(p)), noise(ne(p)), inter.x);
      return mix(s, n, inter.y);
  }

  mat2 rotate (in float theta){
      float c = cos(theta);
      float s = sin(theta);
      return mat2(c, -s, s, c);
  }

  float circ(vec2 p){
      float r = length(p);
      r = log(sqrt(r));
      return abs(mod(4.0 * r, PI * 2.0) - PI) * 3.0 + 0.2;
  }

  float fbm(in vec2 p){
      float z = 2.0;
      float rz = 0.0;
      vec2 bp = p;
      for(float i = 1.0; i < 6.0; i++){
          rz += abs((smoothNoise(p) - 0.5)* 2.0) / z;
          z *= 2.0;
          p *= 2.0;
      }
      return rz;
  }
  float distanceTo(vec2 src, vec2 dst) {
      float dx = src.x - dst.x;
      float dy = src.y - dst.y;
      float dv = dx * dx + dy * dy;
      return sqrt(dv);
  }
  varying vec2 vUv; 
  uniform vec2 iResolution; 
  void main() { 
      float len = distanceTo(vec2(0.5, 0.5), vec2(vUv.x, vUv.y)) * 2.0; 

      vec2 p = vUv - 0.5;
      p.x *= iResolution.x / iResolution.y;
      p *= 8.0;
      float rz = fbm(p);
      p /= exp(mod(iTime * 2.0, PI));
      rz *= pow(abs(0.1 - circ(p)), 0.9);
      vec3 col = vec3(0.2, 0.1, 0.643); 
      
      gl_FragColor = vec4(col / rz,  1.0 - pow(len, 3.0))  ;
      
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

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/waveScan.js)

## 小结

- 本文提供 **波扫描** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

