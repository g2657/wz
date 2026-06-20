---
title: "Three.js 栅格网格教程"
description: "详解 Three.js 栅格网格：基于 WebGL 实现「栅格网格」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,栅格网格,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 栅格网格 · *Raster Grid* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=rasterGrid)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![栅格网格](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/rasterGrid.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互

## 效果说明

本案例演示 **栅格网格** 效果：基于 WebGL 实现「栅格网格」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
import * as THREE from "three";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";

const { innerWidth, innerHeight } = window;
const aspect = innerWidth / innerHeight;

class Base {
    constructor() {
        this.init();
        this.main();
    }
    main() {
        const geometry = new THREE.PlaneGeometry(10,10,100,100);

        const vertexShader = `
        varying vec2 vUv;

        void main(){
            vUv = uv;

            gl_Position = projectionMatrix * modelViewMatrix * vec4(position,1.0);

        }
        `;
        const fragmentShader = `
        uniform float iTime;
        varying vec2 vUv;
        #define PI 3.14159

        vec3 palette(float t){
            vec3 a = vec3(0.5,0.5,0.5);
            vec3 b = vec3(0.5,0.5,0.5);
            vec3 c = vec3(1.0,1.0,1.0);
            vec3 d = vec3(0.263,0.416,0.557);

            return a+b*cos(PI*2.0*(c*t+d));
        }

        vec4 mainImage(){
            vec3 finalColor = vec3(0.0);
            vec2 uuv = vUv*2.0-1.0;
            vec2 uv = vUv*2.0-1.0;
            for(float i = 0.0;i<4.0;i++){
                uv = fract(uv*1.5)-0.5;
                float d = length(uv) * exp(-length(uuv));

                vec3 col = palette(length(uuv) + i*.4 + iTime*.4);

                d = sin(d*8. + iTime)/8.;
                d = abs(d);

                d = pow(0.01 / d, 1.2);

                finalColor += col * d;
            }
            return  vec4(finalColor,1.0);
        }
        void main(){
            gl_FragColor = mainImage();
        }
        `;
        const material = new THREE.ShaderMaterial({
            vertexShader,
            fragmentShader,
            uniforms: {
                iTime: {
                    value: 0,
                },
            },
            side: THREE.DoubleSide,
        });

        const plane = new THREE.Mesh(geometry, material);
        this.material = plane.material;
        this.scene.add(plane);
    }
    init() {
        this.clock = new THREE.Clock();

        this.renderer = new THREE.WebGLRenderer({
            antialias: true,
            logarithmicDepthBuffer: true,
        });
        this.renderer.setPixelRatio(window.devicePixelRatio);
        this.renderer.setSize(innerWidth, innerHeight);
        this.renderer.setAnimationLoop(this.animate.bind(this));
        document.body.appendChild(this.renderer.domElement);

        this.camera = new THREE.PerspectiveCamera(60, aspect, 0.01, 10000);
        this.camera.position.set(5, 5, 5);

        this.scene = new THREE.Scene();

        this.controls = new OrbitControls(
            this.camera,
            this.renderer.domElement
        );

        const grid = new THREE.GridHelper(100);
        this.scene.add(grid);

        const light = new THREE.AmbientLight(0xffffff, 0.5);
        this.scene.add(light);
    }
    animate() {
        this.controls.update();
        this.renderer.render(this.scene, this.camera);
        this.material.uniforms.iTime.value += 0.01;
    }
}
new Base();
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/rasterGrid.js)

## 小结

- 本文提供 **栅格网格** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

