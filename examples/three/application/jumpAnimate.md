---
title: "Three.js 跳跃动画教程"
description: "详解 Three.js 跳跃动画：基于 WebGL 实现「跳跃动画」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,跳跃动画,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 跳跃动画 · *Jump Animate* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=jumpAnimate)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![跳跃动画](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/jumpAnimate.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **跳跃动画** 效果：基于 WebGL 实现「跳跃动画」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
import { GUI } from 'dat.gui'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(50, box.clientWidth / box.clientHeight, 0.1, 2000)

camera.position.set(10, 10, 10)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

scene.add(new THREE.GridHelper(10, 10))

// uniforms
const uniforms = {
    points: {
        value: [
            new THREE.Vector3(-30, 2.0, 0),
            new THREE.Vector3(0, 5.0, 0),
            new THREE.Vector3(30, 10.0, 0)
        ]
    },
    // 目标点（用于插值过渡）
    targetPoints: {
        value: [
            new THREE.Vector3(-30, 2.0, 0),
            new THREE.Vector3(0, 5.0, 0),
            new THREE.Vector3(30, 10.0, 0)
        ]
    },
    baseInfluenceRadius: { value: 5.0 },
    heightToRadiusRatio: { value: 2.0 },
    falloffPower: { value: 2.0 },
    baseHeight: { value: 0.0 },
    maxHeight: { value: 10.0 },
    color1: { value: new THREE.Color(0x0066ff) },
    color2: { value: new THREE.Color(0xff3300) },
}

// 顶点着色器
const vertexShader = `
uniform vec3 points[3];
uniform float baseInfluenceRadius;
uniform float heightToRadiusRatio;
uniform float falloffPower;
uniform float baseHeight;
uniform float maxHeight;

varying float vHeightRatio;

float calculateInfluence(vec3 point, vec3 vertex) {
    float distance2D = length(point.xz - vertex.xz);
    float dynamicRadius = baseInfluenceRadius + point.y * heightToRadiusRatio;
    dynamicRadius = max(dynamicRadius, baseInfluenceRadius);
    if (distance2D > dynamicRadius) return 0.0;
    float normalizedDistance = distance2D / dynamicRadius;
    float attenuation = pow(1.0 - normalizedDistance, falloffPower);
    return attenuation * point.y;
}

void main() {
    vec4 worldPosition = modelMatrix * vec4(position, 1.0);
    float heightOffset = 0.0;
    heightOffset += calculateInfluence(points[0], worldPosition.xyz);
    heightOffset += calculateInfluence(points[1], worldPosition.xyz);
    heightOffset += calculateInfluence(points[2], worldPosition.xyz);
    vHeightRatio = clamp(heightOffset / maxHeight, 0.0, 1.0);
    vec3 newPosition = position;
    newPosition.y += heightOffset + baseHeight;
    gl_Position = projectionMatrix * modelViewMatrix * vec4(newPosition, 1.0);
}
`

// 片段着色器
const fragmentShader = `
uniform vec3 color1;
uniform vec3 color2;
varying float vHeightRatio;

void main() {
    vec3 color = mix(color1, color2, vHeightRatio);
    gl_FragColor = vec4(color, 1.0);
}
`

// 创建网格
const geometry = new THREE.BoxGeometry(200, 10, 1, 200, 1, 1).rotateX(-Math.PI / 2)
const material = new THREE.ShaderMaterial({
    uniforms,
    vertexShader,
    fragmentShader,
    side: THREE.DoubleSide,
    wireframe: true
})
scene.add(new THREE.Mesh(geometry, material))

// GUI 参数
const params = {
    baseInfluenceRadius: 5.0,
    heightToRadiusRatio: 2.0,
    falloffPower: 2.0,
    baseHeight: 0.0,
    maxHeight: 10.0,
    autoUpdate: true,
    interval: 1.0,
    wireframe: true,
    color1: '#0066ff',
    color2: '#ff3300',
}

// GUI 控制
const gui = new GUI()
gui.add(params, 'baseInfluenceRadius', 1, 50).name('影响半径').onChange(v => uniforms.baseInfluenceRadius.value = v)
gui.add(params, 'heightToRadiusRatio', 0, 10).name('高度放大系数').onChange(v => uniforms.heightToRadiusRatio.value = v)
gui.add(params, 'falloffPower', 0.1, 10).name('衰减指数').onChange(v => uniforms.falloffPower.value = v)
gui.add(params, 'baseHeight', -5, 5).name('基础高度').onChange(v => uniforms.baseHeight.value = v)
gui.add(params, 'maxHeight', 1, 30).name('最大高度').onChange(v => uniforms.maxHeight.value = v)
gui.add(params, 'autoUpdate').name('自动更新')
gui.add(params, 'interval', 0.3, 5).name('更新间隔(秒)')
gui.add(params, 'wireframe').name('线框模式').onChange(v => material.wireframe = v)
gui.addColor(params, 'color1').name('低点颜色').onChange(v => uniforms.color1.value.set(v))
gui.addColor(params, 'color2').name('高点颜色').onChange(v => uniforms.color2.value.set(v))

// 随机生成目标点
function randomizeTargets() {
    for (let i = 0; i < 3; i++) {
        uniforms.targetPoints.value[i].set(
            Math.random() * 200 - 100,
            Math.random() * params.maxHeight,
            0
        )
    }
}

// 定时更新目标
let timer = 0
const clock = new THREE.Clock()

function animate() {
    requestAnimationFrame(animate)
    const delta = clock.getDelta()

    // 定时更新目标点
    if (params.autoUpdate) {
        timer += delta
        if (timer >= params.interval) {
            timer = 0
            randomizeTargets()
        }
    }

    // 平滑插值到目标点
    for (let i = 0; i < 3; i++) {
        uniforms.points.value[i].lerp(uniforms.targetPoints.value[i], delta * 3)
    }

    renderer.render(scene, camera)
}
animate()

window.onresize = () => {
    renderer.setSize(box.clientWidth, box.clientHeight)
    camera.aspect = box.clientWidth / box.clientHeight
    camera.updateProjectionMatrix()
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/jumpAnimate.js)

## 小结

- 本文提供 **跳跃动画** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

