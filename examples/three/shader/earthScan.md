---
title: "Three.js 地球扫描教程"
description: "详解 Three.js 地球扫描：基于 WebGL 实现「地球扫描」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,地球扫描,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 地球扫描 · *Earth Scan* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=earthScan)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![地球扫描](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/earthScan.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **地球扫描** 效果：基于 WebGL 实现「地球扫描」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
import { GUI } from 'three/examples/jsm/libs/lil-gui.module.min.js'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(50, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 8, 8)

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

const earthGeometry = new THREE.SphereGeometry(2.5, 32, 16)

const earthMaterial = new THREE.MeshBasicMaterial({ map: new THREE.TextureLoader().load(FILE_HOST + 'threeExamples/shader/earth1.jpg') })

const earth = new THREE.Mesh(earthGeometry, earthMaterial)

scene.add(earth)

const geometry = new THREE.SphereGeometry(3, 32, 16)

const material = new THREE.ShaderMaterial({

  uniforms: {

    iTime: { value: 0.0 },

    pointNum: { value: new THREE.Vector2(64, 32) },

    uColor: { value: new THREE.Color('#bbd9ec') }

  },

  transparent: true,

  vertexShader: `
    varying vec2 vUv;
    void main(){
    vUv=uv;
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
  }`,

  fragmentShader: `
    float PI = acos(-1.0);
    uniform vec3 uColor;
    uniform vec2 pointNum;
    uniform float iTime;                        
    varying vec2 vUv;
    void main(){
    vec2 uv = vUv+ vec2(0.0, iTime);
      float current = abs(sin(uv.y * PI) );             
    if(current < 0.99) {      
      current=current*0.5;
    }
    float d = distance(fract(uv * pointNum), vec2(0.5, 0.5));
    if(d > current*0.2 ) {
       discard;
    } else {
       gl_FragColor =vec4(uColor,current);
    }
  }`

})

const folder = new GUI()

folder.addColor(material.uniforms.uColor, 'value')

folder.add(material.uniforms.pointNum.value, 'x', 1, 128).name('pointNumX')

folder.add(material.uniforms.pointNum.value, 'y', 1, 128).name('pointNumY')

const sphere = new THREE.Mesh(geometry, material)

scene.add(sphere)

animate()

function animate() {

  earth.rotation.y += 0.002

  material.uniforms.iTime.value += 0.002

  requestAnimationFrame(animate)

  controls.update()

  renderer.render(scene, camera)

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/earthScan.js)

## 小结

- 本文提供 **地球扫描** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

