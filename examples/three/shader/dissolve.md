---
title: "Three.js 溶解教程"
description: "详解 Three.js 溶解：Loader，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,溶解,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 溶解 · *Dissolve* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=dissolve)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![溶解](https://z2586300277.github.io/3d-file-server/images/dissolve/dissolve.png)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **溶解** 效果：基于 WebGL 实现「溶解」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js"
import * as dat from 'dat.gui'


/* GUI */

const gui = new dat.GUI()

// Container
const box = document.getElementById("box")


// Scene
const scene = new THREE.Scene()

/**
 * Loader
 */
const textureLoader = new THREE.TextureLoader()


/* Tex */
const dissolveTex = textureLoader.load(FILE_HOST + 'images/dissolve/dissolveTex.png')
dissolveTex.colorSpace = THREE.SRGBColorSpace
const dissolveRampTex = textureLoader.load(FILE_HOST + 'images/dissolve/dissolveRamp.png')
dissolveRampTex.colorSpace = THREE.SRGBColorSpace
const diffuseTex = textureLoader.load(FILE_HOST + 'images/dissolve/diffuse.png')
diffuseTex.colorSpace = THREE.SRGBColorSpace

/**
 * Test mesh
 */
// Geometry
const geometry = new THREE.PlaneGeometry(4, 3, 32, 32)

// Material
const shaderMaterial = new THREE.ShaderMaterial({
  side: THREE.DoubleSide,
  vertexShader:/* glsl */`
    varying vec2 vUv;
    void main() {
      vUv = uv;
      vec4 modelPosition = modelMatrix * vec4(position, 1.);
      vec4 viewPosition = viewMatrix * modelPosition;
      vec4 projectedPosition = projectionMatrix * viewPosition;
      gl_Position = projectedPosition;
    }
    `,
  fragmentShader:/* glsl */`
    uniform sampler2D uDissloveTex;
    uniform sampler2D uRamTex;
    uniform sampler2D uDiffuseTex;
    uniform float uClip;
    varying vec2 vUv;
    
    float customSmoothstep(float min, float max, float x) {
      return (x - min) / (max - min);
    }

    vec4 map(in vec4 value, in vec4 inMin, in vec4 inMax, in vec4 outMin, in vec4 outMax) {
      return outMin + (outMax - outMin) * (value - inMin) / (inMax - inMin);
    }
    
    void main() {
  
      vec4 DissloveTex = texture2D(uDissloveTex, vUv);
      DissloveTex = map(DissloveTex, vec4(0.), vec4(1.), vec4(0.1), vec4(1.));

      if((DissloveTex.r - uClip) < 0.) {
        discard;
      }
     
      float dissloveValue = clamp(customSmoothstep(uClip, uClip+.1, DissloveTex.r), 0., 1.);
      vec4 RamTex = texture2D(uRamTex, vec2(dissloveValue));
      vec4 diffuse = texture2D(uDiffuseTex, vUv);

      vec3 color = vec3(clamp( diffuse.rgb  + RamTex.rgb, 0., 1.));

      gl_FragColor = vec4(color, 1.0);

      #include <tonemapping_fragment>
	    #include <colorspace_fragment>
    }
    `,
  uniforms: {
    uDissloveTex: new THREE.Uniform(dissolveTex),
    uRamTex: new THREE.Uniform(dissolveRampTex),
    uDiffuseTex: new THREE.Uniform(diffuseTex),
    uClip: new THREE.Uniform(0)
  }
})

gui.add(shaderMaterial.uniforms.uClip, 'value').min(0).max(1).step(0.01).name('Clip')

// Mesh
const mesh = new THREE.Mesh(geometry, shaderMaterial)
scene.add(mesh)

/**
 * Sizes
 */
const sizes = {
  width: window.innerWidth,
  height: window.innerHeight
}

window.addEventListener('resize', () => {
  // Update sizes
  sizes.width = window.innerWidth
  sizes.height = window.innerHeight

  // Update camera
  camera.aspect = sizes.width / sizes.height
  camera.updateProjectionMatrix()

  // Update renderer
  renderer.setSize(sizes.width, sizes.height)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
})

/**
 * Camera
 */
// Base camera
const camera = new THREE.PerspectiveCamera(75, sizes.width / sizes.height, 0.1, 100)
camera.position.set(0.25, - 0.25, 3.5)
scene.add(camera)

/**
 * Renderer
 */
const renderer = new THREE.WebGLRenderer()
renderer.antialias = true
renderer.setSize(sizes.width, sizes.height)
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
renderer.outputColorSpace = THREE.SRGBColorSpace
box.appendChild(renderer.domElement)

// Controls
const controls = new OrbitControls(camera, renderer.domElement)
controls.enableDamping = true

/**
 * Animate
 */
const clock = new THREE.Clock()

const tick = () => {
  const elapsedTime = clock.getElapsedTime()

  // Update controls
  controls.update()

  // Render
  renderer.render(scene, camera)

  // Call tick again on the next frame
  window.requestAnimationFrame(tick)
}

tick()
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/dissolve.js)

## 小结

- 本文提供 **溶解** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

