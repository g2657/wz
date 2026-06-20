---
title: "Three.js 边缘模糊效果教程"
description: "详解 Three.js 边缘模糊效果：基于 WebGL 实现「边缘模糊效果」可视化效果，附完整可运行源码，涵盖 RawShaderMaterial 等关键实现，附完整源码与在线 Demo，适合 Three.js 后期处理 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,边缘模糊效果,WebGL,源码,教程,在线案例,RawShaderMaterial,GLSL"
outline: deep
---

### 边缘模糊效果 · *Edge Blur* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=effectComposer&id=EdgeBlurringEffect)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![边缘模糊效果](https://z2586300277.github.io/3d-file-server/images/four/EdgeBlurringEffect.png)

## 你将学到什么

- RawShaderMaterial 手写顶点/片元着色器
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **边缘模糊效果** 效果：基于 WebGL 实现「边缘模糊效果」可视化效果，附完整可运行源码；核心用到 RawShaderMaterial。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- 阅读下方完整源码时，建议从 `init` / `load` / `animate` 三条主线入手，再深入 shader 与工具函数。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
3. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js

import * as THREE from "three";
import { Color, UniformsUtils } from "three";

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(
    75,
    window.innerWidth / window.innerHeight,
    0.1,
    1000
);

scene.background = new Color(0xffffff);

const vertexShader = `
    precision highp float;
    precision highp int;
    uniform mat4 modelViewMatrix;
    uniform mat4 projectionMatrix;
    attribute vec3 position;
    attribute vec2 uv;
    varying vec2 vUv;
    void main() {
        vUv = uv;
        gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
    }
`;
const fragmentShader = `
    precision highp float;
    precision highp int;

    uniform float edge;
    uniform float opacity;

    uniform sampler2D map;

    varying vec2 vUv;

    void main(){
      float edgeMin = edge;
      float edgeMax = 1.0 - edge;

      gl_FragColor = texture2D( map, vUv );

      if(vUv.x < edgeMin){
        if(vUv.y < edgeMin){ // 1
            gl_FragColor.a = (min(vUv.x / edgeMin, vUv.y / edgeMin)) * opacity;
        }
        else if(vUv.y >= edgeMin && vUv.y <= edgeMax){ // 4
            gl_FragColor.a = (vUv.x / edgeMin) * opacity;
        }
        else if(vUv.y > edgeMax){ // 7
            gl_FragColor.a = (min(vUv.x / edgeMin, 1.0 - ((vUv.y - edgeMax) / edgeMin))) * opacity;
        }
        else{
            gl_FragColor = vec4(1.0, 0.0, 0.0, 1.0); // for debug
        }
      }
      else if(vUv.x >= edgeMin && vUv.x <= edgeMax){
            if(vUv.y < edgeMin){ // 2
                gl_FragColor.a = (vUv.y / edgeMin) * opacity;
            }
            else if(vUv.y >= edgeMin && vUv.y <= edgeMax){ // 5(center)
                gl_FragColor.a = 1.0 * opacity;
            }
            else if(vUv.y > edgeMax){ // 8
                gl_FragColor.a = (1.0 - ((vUv.y - edgeMax) / edgeMin)) * opacity;
            }
      }
      else if(vUv.x > edgeMax){
            float xNormal = 1.0 - ((vUv.x - edgeMax) / edgeMin);
        
            if(vUv.y < edgeMin){ // 3
                gl_FragColor.a = (min(vUv.y / edgeMin, xNormal)) * opacity;
            }
            else if(vUv.y >= edgeMin && vUv.y <= edgeMax){ // 6
                gl_FragColor.a = (xNormal) * opacity;
            }
            else if(vUv.y > edgeMax){ // 9
                gl_FragColor.a = (min(xNormal, 1.0 - ((vUv.y - edgeMax) / edgeMin))) * opacity;
            }
      }
    }
`;

const geometry = new THREE.PlaneGeometry(2, 4);

const material1 = new THREE.MeshBasicMaterial({ color: 0x00ff00 });
const material2 = new THREE.RawShaderMaterial({
    uniforms: UniformsUtils.clone({
        map: { type: "t", value: null },
        edge: { type: "float", value: 0.1 },
        opacity: { type: "float", value: 1 },
    }),
    transparent: true,
    opacity: 1,
    alphaTest: 1,
    depthTest: true,
    vertexShader: vertexShader,
    fragmentShader: fragmentShader,
});

const cube = new THREE.Mesh(geometry, material1);
const rect = new THREE.Mesh(geometry, material2);
cube.position.set(-2, 0, 0);
rect.position.set(0, 0, 0);

scene.add(rect);

camera.position.z = 5;

const renderer = new THREE.WebGLRenderer();
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.prepend(renderer.domElement);

function animate() {
    requestAnimationFrame(animate);
    renderer.render(scene, camera);
}

animate();
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/effectComposer/EdgeBlurringEffect.js)

## 小结

- 本文提供 **边缘模糊效果** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

