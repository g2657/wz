---
title: "Three.js 围栏着色器教程"
description: "详解 Three.js 围栏着色器：基于 WebGL 实现「围栏着色器」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls、BufferGeometry 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,围栏着色器,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制,BufferGeometry"
outline: deep
---

### 围栏着色器 · *Fence Shader* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=fenceShader)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![围栏着色器](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/fenceShader.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- BufferGeometry 自定义顶点/索引数据
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **围栏着色器** 效果：基于 WebGL 实现「围栏着色器」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls、BufferGeometry。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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

camera.position.set(0, 40, 40)

const renderer = new THREE.WebGLRenderer()

renderer.setSize(box.clientWidth, box.clientHeight)

new OrbitControls(camera, renderer.domElement)

window.onresize = () => {

  renderer.setSize(box.clientWidth, box.clientHeight)

  camera.aspect = box.clientWidth / box.clientHeight

  camera.updateProjectionMatrix()
  
}

box.appendChild(renderer.domElement)

/* 顶点着色器 */
const vertexs = `varying vec2 vUv;
void main() {
  vec4 mvPosition = modelViewMatrix * vec4( position, 1.0 );
  vUv = uv;
  gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}`

/* 片元着色器 */
const fragments = `
uniform float time;
uniform float opacity;
uniform vec3 color;
uniform float num;
uniform float hiz;
varying vec2 vUv;
void main() {
  vec4 fragColor = vec4(0.);
  float sin = sin((vUv.y - time * hiz) * 10. * num);
  float high = 0.92;
  float medium = 0.4;
  if (sin > high) {
    fragColor = vec4(mix(vec3(.8, 1., 1.), color, (1. - sin) / (1. - high)), 1.);
  } else if(sin > medium) {
    fragColor = vec4(color, mix(1., 0., 1.-(sin - medium) / (high - medium)));
  } else {
    fragColor = vec4(color, 0.);
  }
  vec3 fade = mix(color, vec3(0., 0., 0.), vUv.y);
  fragColor = mix(fragColor, vec4(fade, 1.), 0.85);
  gl_FragColor = vec4(fragColor.rgb, fragColor.a * opacity * (1. - vUv.y));
}`

// 自定义材质
const material = new THREE.ShaderMaterial({
    uniforms: {
      time: {
        type: "pv2",
        value: 0,
      },
      color: {
        type: "uvs",
        value: new THREE.Color("#FF4127"),
      },
      opacity: {
        type: "pv2",
        value: 1.0,
      },
      num: {
        type: "pv2",
        value: 10,
      },
      hiz: {
        type: "pv2",
        value: 0.03,
      },
    },
    vertexShader: vertexs,
    fragmentShader: fragments,
    blending: THREE.AdditiveBlending,
    transparent: !0,
    depthWrite: !1,
    depthTest: !0,
    side: THREE.DoubleSide,
});

let c = [0,0, 10, 0, 10, 10, 0, 10, 0, 0]

let posArr = [];

let uvrr = [];

let h = 10; //围墙拉伸高度

for (let i = 0; i < c.length - 2; i += 2) {

  // 矩形的三角形1
  posArr.push(c[i], c[i + 1], 0, c[i + 2], c[i + 3], 0, c[i + 2], c[i + 3], h);

  // 矩形的三角形2
  posArr.push(c[i], c[i + 1], 0, c[i + 2], c[i + 3], h, c[i], c[i + 1], h);

  // 注意顺序问题，和顶点位置坐标对应
  uvrr.push(0, 0, 1, 0, 1, 1);

  uvrr.push(0, 0, 1, 1, 0, 1);

}

const geometry = new THREE.BufferGeometry(); //声明一个空几何体对象

// 设置几何体attributes属性的位置position属性
geometry.attributes.position = new THREE.BufferAttribute(new Float32Array(posArr), 3);

// 设置几何体attributes属性的位置uv属性
geometry.attributes.uv = new THREE.BufferAttribute(new Float32Array(uvrr), 2);

geometry.computeVertexNormals()

const mesh = new THREE.Mesh(geometry, material) //网格模型对象Mesh

scene.add(mesh)

function animate() {

    requestAnimationFrame(animate)

    mesh.rotation.x += 0.01

    mesh.rotation.y += 0.01

    material.uniforms.time.value += 0.03

    renderer.render(scene, camera)

}

animate()
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/fenceShader.js)

## 小结

- 本文提供 **围栏着色器** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

