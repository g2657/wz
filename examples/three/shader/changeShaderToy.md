---
title: "Three.js 切换ShaderToy教程"
description: "详解 Three.js 切换ShaderToy：基于 WebGL 实现「切换ShaderToy」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,切换ShaderToy,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 切换ShaderToy · *shaderToy* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=changeShaderToy)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![切换ShaderToy](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/changeShaderToy.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **切换ShaderToy** 效果：基于 WebGL 实现「切换ShaderToy」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
import * as dat from 'dat.gui'

const DOM = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, DOM.clientWidth / DOM.clientHeight, 0.1, 1000)

camera.position.set(0, 10, 10)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(DOM.clientWidth, DOM.clientHeight)

DOM.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

window.onresize = () => {

    renderer.setSize(DOM.clientWidth, DOM.clientHeight)

    camera.aspect = DOM.clientWidth / DOM.clientHeight

    camera.updateProjectionMatrix()

}

scene.add(new THREE.AmbientLight(0xffffff, 8))

const mesh = new THREE.Mesh(new THREE.BoxGeometry(4, 4, 4), new THREE.MeshBasicMaterial({ color: 0xffffff }));

scene.add(mesh);

// GUI 对象
const GUI = new dat.GUI()

const fileList = new Array(6).fill().map((_, i) => FILE_HOST + `files/glsl/${i}.frag`)

GUI.add({ url: fileList[0] }, 'url', fileList).onChange((url) => changeShader(url))

changeShader(fileList[5])

let shader = null

animate()

// 渲染 
function animate() {

    shader && (shader.uniforms.u_time.value += 0.02)

    renderer.render(scene, camera)

    requestAnimationFrame(animate)

}

async function changeShader(url) {

    const str = await fetch(url).then(res => res.text())

    shader = {

        uniforms: THREE.UniformsUtils.merge([

            THREE.ShaderLib['phong'].uniforms,

            {
                u_resolution: {
                    type: 'v2',
                    value: new THREE.Vector2(DOM.clientWidth, DOM.clientHeight)
                },

                u_time: {
                    type: 'f',
                    value: 0.0
                },

                u_mouse: {
                    type: 'v2',
                    value: new THREE.Vector2(0, 0)
                }

            }

        ]),

        side: THREE.DoubleSide,

        vertexShader: `
            varying vec2 vUv;
            void main() {
                vUv = uv;
                vec4 mvPosition = modelViewMatrix * vec4( position, 1.0 );
                gl_Position = projectionMatrix * mvPosition;
            }
        `,

        fragmentShader: str,

    }

    shader.fragmentShader = shader.fragmentShader.replace(/gl_FragCoord/, 'vUv * u_resolution.xy')

    shader.fragmentShader = shader.fragmentShader.replace(/uniform float u_time;/, `
        uniform float u_time;
        varying vec2 vUv;
    `)

    const material = new THREE.ShaderMaterial(shader);

    mesh.material.dispose()

    mesh.material = material

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/changeShaderToy.js)

## 小结

- 本文提供 **切换ShaderToy** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

