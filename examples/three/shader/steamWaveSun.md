---
title: "Three.js 蒸汽波太阳教程"
description: "详解 Three.js 蒸汽波太阳：基于 WebGL 实现「蒸汽波太阳」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,蒸汽波太阳,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 蒸汽波太阳 · *Steam Sun* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=steamWaveSun)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![蒸汽波太阳](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/steamWaveSun.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **蒸汽波太阳** 效果：基于 WebGL 实现「蒸汽波太阳」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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

camera.position.set(0, 0, 2)

const renderer = new THREE.WebGLRenderer()

renderer.setSize(box.clientWidth, box.clientHeight)


window.onresize = () => {

  renderer.setSize(box.clientWidth, box.clientHeight)

  camera.aspect = box.clientWidth / box.clientHeight

  camera.updateProjectionMatrix()
  
}

box.appendChild(renderer.domElement)

/* 顶点着色器 */
const vertexShader = `
varying vec2 vUv;
void main(){
	vec4 mPosition=modelMatrix*vec4(position,1.);
	gl_Position=projectionMatrix*viewMatrix*mPosition;
	vUv = uv;
}`

/* 片元着色器 */
const fragmentShader = `
varying vec2 vUv;
uniform float uTime;

void main(){
  vec2 uv=vUv;
  float circular=length(uv-vec2(.5));
  circular=smoothstep(.3,.29,circular);
  vec3 color=vec3(0.);
  vec3 sumCol=mix(vec3(4.,0.,.2),vec3(1.,1.1,0.),uv.y);
  // 递减值
  float decreasing=(-uv.y+.3);
  uv.y +=uTime/10.;
  float line=fract(uv.y*20.);
  line=abs(line-.5);
  line-=decreasing;
  line=step(line,.2);
  color+=circular*sumCol*(1.-line);
  gl_FragColor=vec4(color,1.);
}`

// 自定义材质
const material = new THREE.ShaderMaterial({
  fragmentShader: fragmentShader,
  vertexShader: vertexShader,
  uniforms: {
    uTime: { value: 0 },
  },
  side: THREE.DoubleSide,
})

const geometry = new THREE.PlaneGeometry(2, 2)
const mesh = new THREE.Mesh(geometry, material) //网格模型对象Mesh

scene.add(mesh)

function animate() {
    requestAnimationFrame(animate)
    material.uniforms.uTime.value += 0.03
    renderer.render(scene, camera)
}
animate()
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/steamWaveSun.js)

## 小结

- 本文提供 **蒸汽波太阳** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

