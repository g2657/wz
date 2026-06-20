---
title: "Three.js 围墙着色器教程"
description: "详解 Three.js 围墙着色器：基于 WebGL 实现「围墙着色器」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,围墙着色器,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 围墙着色器 · *Fence Wall* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=fenceWall)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![围墙着色器](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/fenceWall.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互

## 效果说明

本案例演示 **围墙着色器** 效果：基于 WebGL 实现「围墙着色器」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
        const geometry = new THREE.CylinderGeometry(30, 30, 20, 4, 64, true);
        const material = new THREE.ShaderMaterial({
            side: THREE.DoubleSide,
            transparent: true,
            depthWrite: false,
            uniforms: {
                uTime: this.deltaTime,
            },
            vertexShader: `
                        varying vec2 vUv; 
                        void main(){
                            vUv = uv;
                            gl_Position = projectionMatrix * modelViewMatrix * vec4(position,1.0);
                        }
                        `,
            fragmentShader: `
                        uniform float uTime;
                        varying vec2 vUv;
                        #define PI 3.14159265

                        void main(){

                        vec4 baseColor = vec4(0.0,1.,0.5,1.);
						
                        vec4 finalColor;
                            
                        float amplitude = 1.;
                        float frequency = 10.;
                        
                        float x = vUv.x;

                        float y = sin(x * frequency) ;
                        float t = 0.01*(-uTime*130.0);
                        y += sin(x*frequency*2.1 + t)*4.5;
                        y += sin(x*frequency*1.72 + t*1.121)*4.0;
                        y += sin(x*frequency*2.221 + t*0.437)*5.0;
                        y += sin(x*frequency*3.1122+ t*4.269)*2.5;
                        y *= amplitude*0.06;
                        y /= 3.;
                        y += 0.55;

                        vec4 color = gl_FragColor.rgba;

                        float r = step(0.5, fract(vUv.y - uTime));

                        baseColor.a = step(vUv.y,y) * (y-vUv.y)/y;
                        
                        gl_FragColor = baseColor;

                        }
                        `,
        });
        const cylinder = new THREE.Mesh(geometry, material);
        cylinder.position.y += 10;
        this.scene.add(cylinder);
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
        this.camera.position.set(0, 40, 40);

        this.scene = new THREE.Scene();

        this.controls = new OrbitControls(this.camera, this.renderer.domElement);

        this.deltaTime = {
            value: 0,
        };

        const grid = new THREE.GridHelper(100);
        this.scene.add(grid);

        const light = new THREE.AmbientLight(0xffffff, 0.5);
        this.scene.add(light);
    }
    animate() {
        this.controls.update();
        this.renderer.render(this.scene, this.camera);
        this.deltaTime.value = this.clock.getElapsedTime();
    }
}
new Base();
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/fenceWall.js)

## 小结

- 本文提供 **围墙着色器** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

