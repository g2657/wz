---
title: "Three.js 内发光教程"
description: "详解 Three.js 内发光：需要注意法线向量值，菲涅尔反射对于法线向量差别比较大的模型效果明显，即不规则物体，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,内发光,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 内发光 · *Inner Glow* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=innerGlow)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![内发光](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/innerGlow.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **内发光** 效果：基于 WebGL 实现「内发光」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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

const camera = new THREE.PerspectiveCamera(50, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(5, 5, 10)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

scene.add(new THREE.AxesHelper(50))

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

animate()

function animate() {

    requestAnimationFrame(animate)

    controls.update()

    renderer.render(scene, camera)

}

const vertexShader = `
varying vec3 vNormal;
varying vec3 vPositionNormal;

void main() {
  // 视图空间下的单位法线向量
  vNormal = normalize(normalMatrix * normal);
  // 视图空间下的单位坐标向量
  vPositionNormal = normalize((modelViewMatrix * vec4(position, 1.0) ).xyz);
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}
`

/**
* 需要注意法线向量值，菲涅尔反射对于法线向量差别比较大的模型效果明显，即不规则物体
* 模型表面平整的话效果较差，不过可以使用法线贴图更改物体法线向量
*/
const fragmentShader = `
uniform vec3 uColor;
uniform float uBias;
uniform float uPower;
uniform float uScale;

varying vec3 vNormal;
varying vec3 vPositionNormal;

// 菲涅尔反射
float fresnelReflex() {
    return pow(uBias + uScale * abs(dot(vNormal, vPositionNormal)), uPower);
}

void main() {
    float opacity = fresnelReflex();
    gl_FragColor = vec4(uColor, opacity);
}`

const material = new THREE.ShaderMaterial({
    uniforms: {
        uColor: { value: new THREE.Color(0x00ffff) },
        uBias: { value: 1.0 },
        uScale: { value: -1.0 },
        uPower: { value: 2.0 }
    },
    vertexShader,
    fragmentShader,
    transparent: true
})


const sphere = new THREE.Mesh(new THREE.SphereGeometry(1, 32, 16), material)

scene.add(sphere)


const vertexShader1 = `
  varying vec2 vUV;

  void main() {
  vUV = uv;
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
  }
  `
const fragmentShader1 = `
  uniform vec3 uColor;

  varying vec2 vUV;

  float glow(vec2 coord, float radius, float intensity) {
    return pow(radius / length(coord), intensity);
  }

  void main() {
    float ratio = glow(vUV - vec2(0.5), 0.1, 3.0);
    gl_FragColor = vec4(uColor * ratio, ratio);
  }

  `
const material1 = new THREE.ShaderMaterial({
  uniforms: {
    uColor: { value: new THREE.Color(0x00ffff) }
  },
  vertexShader: vertexShader1,
  fragmentShader: fragmentShader1,
  depthWrite: false,
  transparent: true
})

const plane = new THREE.Mesh(new THREE.PlaneGeometry(2, 2), material1)

plane.position.set(1, 0, 1)

scene.add(plane)
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/innerGlow.js)

## 小结

- 本文提供 **内发光** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

