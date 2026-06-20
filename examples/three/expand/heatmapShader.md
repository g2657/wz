---
title: "Three.js 热力图教程"
description: "详解 Three.js 热力图：基于 WebGL 实现「热力图」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 扩展 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,热力图,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 热力图 · *Heatmap Shader* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=expand&id=heatmapShader)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![热力图](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/heatmapShader.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **热力图** 效果：基于 WebGL 实现「热力图」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。

## 实现步骤

1. 搭建灯光与环境（如有）
2. requestAnimationFrame 循环 update + render

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GUI } from "three/addons/libs/lil-gui.module.min.js";

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 0, 20)

const renderer = new THREE.WebGLRenderer()

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

animate()

function animate() {

    requestAnimationFrame(animate)

    renderer.render(scene, camera)

}

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

}

scene.add(new THREE.AmbientLight(0xffffff, 2), new THREE.AxesHelper(1000))

/* 热力图实现 */
const arr = [[0., 0., 10.], [.2, .6, 5.], [.25, .7, 8.], [.33, .9, 5.], [.35, .8, 6.], [0.017, 5.311, 6.000], [-.45, .8, 4.], [-.2, -.6, 5.], [-.25, -.7, 8.], [-.33, -.9, 8.], [.35, -.45, 10.], [-.1, -.8, 10.], [.33, -.3, 5.], [-.35, .75, 6.], [.6, .4, 10.], [-.4, -.8, 4.], [.7, -.3, 6.], [.3, -.8, 8.]].map(i => new THREE.Vector3(...i))

const uniforms1 = {

    HEAT_MAX: { value: 10, type: 'number', unit: 'float' },

    PointRadius: { value: 0.42, type: 'number', unit: 'float' },

    PointsCount: { value: arr.length, type: 'number-array', unit: 'int' }, // 数量

    c1: { value: new THREE.Color(0x000000), type: 'color', unit: 'vec3' },

    c2: { value: new THREE.Color(0x000000), type: 'color', unit: 'vec3' },

    uvY: { value: 1, type: 'number', unit: 'float' },

    uvX: { value: 1, type: 'number', unit: 'float' },

    opacity: { value: 1, type: 'number', unit: 'float' }

}

const gui = new GUI()

gui.add(uniforms1.HEAT_MAX, 'value', 0, 10).name('HEAT_MAX')

gui.add(uniforms1.PointRadius, 'value', 0, 1).name('PointRadius')

gui.add(uniforms1.uvY, 'value', 0, 1).name('uvY')

gui.add(uniforms1.uvX, 'value', 0, 1).name('uvX')

gui.add(uniforms1.opacity, 'value', 0, 1).name('opacity')

gui.addColor(uniforms1.c1, 'value').name('c1')

gui.addColor(uniforms1.c2, 'value').name('c2')

const uniforms2 = {

    Points: { value: arr, type: 'vec3-array', unit: 'vec3' }

}

const uniforms = {

    ...uniforms1,

    ...uniforms2

}

const vertexShader = `
varying vec2 vUv;
void main() {
    vUv = uv;
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}
`

const getFragmentShader = () => 'precision highp float;\n' + 'varying vec2 vUv; \n' +

    Object.keys(uniforms1).map(i => 'uniform ' + uniforms1[i].unit + ' ' + i + ';')
        .join('\n')

    + '\nuniform vec3 Points['
    + uniforms1.PointsCount.value + '];'
    +
    `
vec3 gradient(float w, vec2 uv) {
    w = pow(clamp(w, 0., 1.) * 3.14159 * .5, .9);
    return vec3(sin(w), sin(w * 2.), cos(w))* 1.1 + mix(c1, c2, w) * 1.1; 
}
void main()
{
    vec2 uv = vUv;
    uv.xy *= vec2(uvX, uvY);
    float d = 0.;
    for (int i = 0; i < PointsCount; i++) {
        vec3 v = Points[i];
        float intensity = v.z / HEAT_MAX;
        float pd = (1. - length(uv - v.xy) / PointRadius) * intensity;
        d += pow(max(0., pd), 2.);
    }
    gl_FragColor = vec4(gradient(d, uv), opacity);
} `

const shaderMaterial = new THREE.ShaderMaterial({

    uniforms,

    vertexShader,

    fragmentShader: getFragmentShader(),

    transparent: true


})

const plane = new THREE.Mesh(new THREE.PlaneGeometry(20, 20), shaderMaterial)

scene.add(plane)

/* 循环 */
setInterval(() => {

    arr.pop()

    arr.unshift(new THREE.Vector3(Math.random() * 2 - 1, Math.random() * 2 - 1, Math.random() * 10))

    uniforms.PointsCount.value = arr.length

    shaderMaterial.fragmentShader = getFragmentShader()

    shaderMaterial.needsUpdate = true

}, 200)
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/heatmapShader.js)

## 小结

- 本文提供 **热力图** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

