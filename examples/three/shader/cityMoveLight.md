---
title: "Three.js 智慧城市扫光教程"
description: "详解 Three.js 智慧城市扫光：基于 WebGL 实现「智慧城市扫光」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls、FBXLoader 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,智慧城市扫光,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制,FBX"
outline: deep
---

### 智慧城市扫光 · *City Move* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=cityMoveLight)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![智慧城市扫光](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/cityMoveLight.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- FBXLoader 加载 FBX 城市/角色模型
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **智慧城市扫光** 效果：基于 WebGL 实现「智慧城市扫光」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls、FBXLoader。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 异步加载模型 / 3D Tiles / GeoJSON 等资源并加入 scene 或 entities
3. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
4. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
5. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { FBXLoader } from 'three/examples/jsm/loaders/FBXLoader.js'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(10, 10, 10)

const renderer = new THREE.WebGLRenderer()

renderer.setPixelRatio(window.devicePixelRatio * 1.5)

renderer.setSize(box.clientWidth, box.clientHeight)

new OrbitControls(camera, renderer.domElement)

window.onresize = () => {

  renderer.setSize(box.clientWidth, box.clientHeight)

  camera.aspect = box.clientWidth / box.clientHeight

  camera.updateProjectionMatrix()
  
}

box.appendChild(renderer.domElement)

// 坐标轴
scene.add(new THREE.AxesHelper(100000))

// 着色器
const uniforms = {

    innerCircleWidth: { value: 0 },

    circleWidth: { value: 300 },

    diff: { value: new THREE.Color(1., 1., 1.) },

    color: { value: new THREE.Color('#8f95ff') },

    opacity: { value: 0.6 },

    center: { value: new THREE.Vector3(0, 0, 0) }

}

const material = new THREE.ShaderMaterial({

    uniforms,

    transparent: true,

    vertexShader: `
        varying vec2 vUv;
        varying vec3 v_position;
        void main() {
            vUv = uv;
            v_position = position;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
    `,

    fragmentShader: `
        varying vec2 vUv;
        varying vec3 v_position;

        uniform float innerCircleWidth;
        uniform float circleWidth;
        uniform float opacity;
        uniform vec3 center;
        
        uniform vec3 color;
        uniform vec3 diff;

        void main() {
            float dis = length(v_position - center);
            if(dis < (innerCircleWidth + circleWidth) && dis > innerCircleWidth) {
                float r = (dis - innerCircleWidth) / circleWidth;
            
                gl_FragColor = mix(vec4(diff, opacity), vec4(color, opacity), r);
            }else {
                gl_FragColor = vec4(diff, opacity);
            }
        }
    `

})

// 加载模型
new FBXLoader().load(HOST + '/files/model/city.FBX', (object3d) => {

    scene.add(object3d)

    object3d.scale.set(0.001, 0.001, 0.001)

    object3d.traverse((child) => {

        if (child.isMesh) child.material = material

    })

})

// 渲染
animate()

function animate() {

    if (uniforms.innerCircleWidth.value < 1000) uniforms.innerCircleWidth.value += 3
    
    else uniforms.innerCircleWidth.value = 0

    renderer.render(scene, camera)

    requestAnimationFrame(animate)

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/cityMoveLight.js)

## 小结

- 本文提供 **智慧城市扫光** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

