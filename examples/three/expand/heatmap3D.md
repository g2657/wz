---
title: "Three.js 3D热力图教程"
description: "详解 Three.js 3D热力图：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理，涵盖 ShaderMaterial、OrbitControls、Canvas 等关键实现，附完整源码与在线 Demo，适合 Three.js 扩展 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,3D热力图,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制,Canvas纹理"
outline: deep
---

### 3D热力图 · *Heatmap 3D* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=expand&id=heatmap3D)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![3D热力图](https://z2586300277.github.io/three-cesium-examples/threeExamples/expand/heatmap3D.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- Canvas 动态纹理贴图
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **3D热力图** 效果：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理；核心用到 ShaderMaterial、OrbitControls、Canvas。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- **CanvasTexture** 每帧或按需把 2D Canvas 内容上传 GPU，适合动态文字、图表、视频帧贴图。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
3. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
4. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import * as dat from 'dat.gui'
/* heatmap.js 自行安装module 方式引入  此处我为src 方式引入  */

const DOM = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, DOM.clientWidth / DOM.clientHeight, 0.1, 1000)

camera.position.set(0, 40, 40)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(DOM.clientWidth, DOM.clientHeight)

DOM.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

scene.add(new THREE.AxesHelper(500), new THREE.AmbientLight(0xffffff, 3))

window.onresize = () => {

    renderer.setSize(DOM.clientWidth, DOM.clientHeight)

    camera.aspect = DOM.clientWidth / DOM.clientHeight

    camera.updateProjectionMatrix()

}

animate()

// 渲染 
function animate() {

    renderer.render(scene, camera)

    requestAnimationFrame(animate)

}

const getRandom = (max, min) => Math.round((Math.random() * (max - min + 1) + min) * 10) / 10

var heatmap = h337.create({
    container: document.createElement('div'),
    width: 256,
    height: 256,
    blur: '0.8',
    radius: 10
});

var i = 0, max = 10, data = [];
while (i < 2000) {
    data.push({ x: getRandom(1, 256), y: getRandom(1, 256), value: getRandom(1, 6) });
    i++;
}

heatmap.setData({
    max: max,
    data: data
});

const texture = new THREE.CanvasTexture(heatmap._renderer.canvas);
const geometry = new THREE.PlaneGeometry(50, 50, 1000, 1000);
geometry.rotateX(-Math.PI * 0.5);

const material = new THREE.ShaderMaterial({
    uniforms: {
        heightMap: { value: texture },
        heightRatio: { value: 5 }
    },
    vertexShader: `	uniform sampler2D heightMap;
			uniform float heightRatio;
			varying vec2 vUv;
			varying float hValue;
			varying vec3 cl;
			void main() {
			    vUv = uv;
			    vec3 pos = position;
		        cl = texture2D(heightMap, vUv).rgb;
		        hValue = texture2D(heightMap, vUv).r;
		        pos.y = hValue * heightRatio;
		        gl_Position = projectionMatrix * modelViewMatrix * vec4(pos,1.0);
		    }`,
    fragmentShader: `varying float hValue;
			varying vec3 cl;
			void main() {
		 		float v = abs(hValue - 1.);
		 		gl_FragColor = vec4(cl, .8 - v * v) ; 
		    }`,
    transparent: true,
})

const mesh = new THREE.Mesh(geometry, material);
scene.add(mesh);

new dat.GUI().add(mesh.material.uniforms.heightRatio, "value", 1, 15).name("heightRatio")
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/expand/heatmap3D.js)

## 小结

- 本文提供 **3D热力图** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

