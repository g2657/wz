---
title: "Three.js 平面扫描教程"
description: "详解 Three.js 平面扫描：基于 WebGL 实现「平面扫描」可视化效果，附完整可运行源码，涵盖 onBeforeCompile、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,平面扫描,WebGL,源码,教程,在线案例,onBeforeCompile,shader注入,OrbitControls,相机控制"
outline: deep
---

### 平面扫描 · *Plane Scan* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=planeScan)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![平面扫描](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/planeScan.jpg)

## 你将学到什么

- onBeforeCompile 注入 GLSL 改造内置材质
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **平面扫描** 效果：基于 WebGL 实现「平面扫描」可视化效果，附完整可运行源码；核心用到 onBeforeCompile、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **onBeforeCompile** 在 Three 拼好内置 shader 后替换 `#include <xxx>` 片段，适合在 PBR 材质上叠加大屏特效。
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

camera.position.set(0, 2, 2)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.outputColorSpace = THREE.LinearSRGBColorSpace 

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

const plane = new THREE.PlaneGeometry(5,5)

const map = new THREE.TextureLoader().load('https://z2586300277.github.io/three-editor/dist/files/channels/texture.jpg')

const material = new THREE.MeshBasicMaterial({ color: 0xffffff, side: THREE.DoubleSide, map , transparent: true })

const mesh = new THREE.Mesh(plane, material)

const uniforms = {

    innerCircleWidth: { value: 0.59, type: 'number', unit: 'float' },

    circleWidth: { value: 0.2, type: 'number', unit: 'float' },

    circleMax: { value: 0.7, type: 'number', unit: 'float' },

    opacityScale: { value: 0.8, type: 'number', unit: 'float' },

    reverseOpacity: { value: false, type: 'bool', unit: 'bool' },

    circleSpeed: { value: 0.003, type: 'number', unit: 'float' },

    diff: { value: new THREE.Color(0xfff25f), type: 'color', unit: 'vec3' },

    color3: { value: new THREE.Color(0xff), type: 'color', unit: 'vec3' },

    center: { value: new THREE.Vector3(0, 0, 0), type: 'position', unit: 'vec3' },

    intensity: { value: 3, type: 'number', unit: 'float' },

    isDisCard: { value: true, type: 'bool', unit: 'bool' },

}

material.onBeforeCompile = (shader) => {

    Object.keys(uniforms).forEach((key) => shader.uniforms[key] = uniforms[key])

    shader.vertexShader = shader.vertexShader.replace(`void main() {`, 
    `varying vec2 vUv;
    varying vec3 v_position;
    void main() {
        vUv = uv;
        v_position = position;`
    )

    shader.fragmentShader = shader.fragmentShader.replace(

        /#include <common>/,

        Object.keys(uniforms).map(i => 'uniform ' + uniforms[i].unit + ' ' + i + ';').join('\n') + '\n' + 'varying vec3 v_position; varying vec2 vUv;\n'

        + '\n#include <common>\n'

    )

    shader.fragmentShader = shader.fragmentShader.replace('vec4 diffuseColor = vec4( diffuse, opacity );', `
        float dis = length(v_position - center);
        vec4 diffuseColor;
        if(dis < (innerCircleWidth + circleWidth) && dis > innerCircleWidth) {
            float r = (dis - innerCircleWidth) / circleWidth;
            float cOpacity = reverseOpacity ? (innerCircleWidth / circleMax) : 1. - ( innerCircleWidth / circleMax );
            #ifdef USE_MAP
                vec3 textureColor = texture2D(map, vUv).rgb;
                if(isDisCard && textureColor.r < 0.1 && textureColor.g < 0.1  && textureColor.b < 0.1 ) discard;
            #endif
            diffuseColor = vec4( mix(diff, color3, r) * vec3(intensity, intensity, intensity)  , opacity * cOpacity * opacityScale);
        }
        else {
            if(isDisCard)  discard ;
            else diffuseColor = vec4( diffuse, opacity );
        }
    `)

}

material.needsUpdate = true

mesh.rotation.x = Math.PI / 2

scene.add(mesh)

animate()

function animate() {

    uniforms.innerCircleWidth.value < uniforms.circleMax.value ? uniforms.innerCircleWidth.value += uniforms.circleSpeed.value : uniforms.innerCircleWidth.value = 0

    requestAnimationFrame(animate)

    controls.update()

    renderer.render(scene, camera)

}

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/planeScan.js)

## 小结

- 本文提供 **平面扫描** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

