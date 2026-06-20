---
title: "Three.js 图像波动教程"
description: "详解 Three.js 图像波动：基于 WebGL 实现「图像波动」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,图像波动,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 图像波动 · *Wave Shader* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=waveShader)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![图像波动](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/waveShader.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- 监听窗口 `resize` 同步更新 camera 与 renderer

## 效果说明

本案例演示 **图像波动** 效果：基于 WebGL 实现「图像波动」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import Stats from 'three/examples/jsm/libs/stats.module.js';

class ThreeCore {
    dom;
    scene;
    camera;
    defaultCamera;
    renderer;
    clock;
    options;
    stats;
    // 要执行动画的对象集合, 子类可以把自己的动画写进 onRender 也可以 this.addAnimate() 添加到父类动画集合里
    animates;
    constructor(dom, options) {
        this.dom = dom;
        this.options = options;
        this.scene = new THREE.Scene();
        this.clock = new THREE.Clock();
        this.animates = {};
        const k = dom.clientWidth / dom.clientHeight;
        if ("fov" in options.cameraOptions) {
            this.defaultCamera = new THREE.PerspectiveCamera(options.cameraOptions.fov, k, options.cameraOptions.near, options.cameraOptions.far);
        }
        else {
            const s = options.cameraOptions.s;
            this.defaultCamera = new THREE.OrthographicCamera(-s * k, s * k, s, -s, options.cameraOptions.near, options.cameraOptions.far);
        }
        this.camera = this.defaultCamera;
        this.scene.add(this.camera);
        const rendererOptions = {
            // 抗锯齿
            antialias: true,
            alpha: true,
            // 深度缓冲, 解决模型重叠部分不停闪烁问题
            // 这个属性会导致精灵材质会被后面的物体遮挡
            // 只能出现问题的时候, 在那个场景 new ThreeCore继承类的时候, 传入rendererOptions参数, 将此参数改为 false
            logarithmicDepthBuffer: true
        };
        const renderer = new THREE.WebGLRenderer(Object.assign({}, rendererOptions, options.rendererOptions));
        renderer.setSize(dom.clientWidth, dom.clientHeight);
        renderer.setPixelRatio(window.devicePixelRatio);
        dom.appendChild(renderer.domElement);
        this.renderer = renderer;
        this.stats = new Stats();
        dom.appendChild(this.stats.dom);
        window.addEventListener("resize", this.onResize, false);
        //利用 setTimeout 宏任务最后执行特性, 使js执行过程要等所有微任务和同步代码执行完再执行, 否则 this.init() 可能会在场景未搭建完毕就执行报错或没有生产对象

        this.init();
        this.animate();

    }
    // 提供给子类覆写
    onRenderer() { }
    // 提供给子类覆写
    onDestroy() { }
    animate() {
        this.renderer.setAnimationLoop(() => {
            this.onRenderer();
            this.stats.update();
            // 执行动画
            for (const key in this.animates) {
                this.animates[key](this.clock.getDelta());
            }
            this.renderer.render(this.scene, this.camera);
        });
    }
    addAnimate(name, func) {
        this.animates[name] = func;
    }
    removeAnimate(name) {
        delete this.animates[name];
    }
    onResize = () => {
        const width = this.dom.clientWidth;
        const height = this.dom.clientHeight;
        const k = width / height;
        // 更新相机
        if (this.camera instanceof THREE.PerspectiveCamera) {
            this.camera.aspect = k;
        }
        else {
            const s = this.options.cameraOptions.s;
            this.camera.left = -s * k;
            this.camera.right = s * k;
            this.camera.top = s;
            this.camera.bottom = -s;
        }
        this.camera.updateProjectionMatrix();
        // 更新renderer
        this.renderer.setSize(width, height);
        this.renderer.setPixelRatio(window.devicePixelRatio);
    };
    destroyed() {
        // 需要手动移除掉 gui, 否则刷新页面时会出现多个gui
        document.querySelector(".dg.main.a")?.remove();
        window.removeEventListener("resize", this.onResize.bind(this));
        this.onDestroy();
        this.renderer.setAnimationLoop(null);
        this.renderer.renderLists.dispose();
        this.renderer.dispose();
        this.renderer.forceContextLoss();
        this.renderer.domElement.innerHTML = "";
        this.scene.clear();
        THREE.Cache.clear();
    }
}

class ThreeProject extends ThreeCore {
    orbit;
    shadeMaterial;
    params = {
        uWaresFrequency: 20,
        uScale: 0.1,
        uNoiseFrequency: 40,
        uNoiseScale: 2,
        uXzScale: 2,
        uLowColor: 0x000000,
        uHighColor: 0xffffff,
        uOpacity: 0.5,
    };
    constructor(dom) {
        super(dom, {
            cameraOptions: {
                fov: 45,
                near: 0.1,
                far: 100
            }
        });
        this.scene.background = new THREE.Color(0x000000);
        this.camera.position.set(0, 0, 3);
        this.orbit = new OrbitControls(this.camera, this.renderer.domElement);
        const texture = new THREE.TextureLoader().load(FILE_HOST + 'threeExamples/shader/flag.webp');
        const shadeMaterial = new THREE.ShaderMaterial({
            vertexShader: `precision mediump float;
                    uniform float uTime;
                    varying vec2 vUv;
                    
                    void main() {
                        vUv = uv;
                        vec4 modelPosition = modelMatrix * vec4(position, 1.0);
                        modelPosition.z = sin((modelPosition.x + uTime) * 10.0) * 0.05;
                        modelPosition.z += sin((modelPosition.y + uTime) * 10.0) * 0.05;
                        gl_Position = projectionMatrix * viewMatrix * modelPosition;
                    }`,
            fragmentShader: `precision mediump float;
                    uniform sampler2D uTexture;
                    varying vec2 vUv;
                    
                    void main() {
                        vec4 textureColor = texture2D(uTexture, vUv);
                        gl_FragColor = textureColor;
                    }`,
            side: THREE.DoubleSide,
            uniforms: {
                uTime: {
                    value: 0
                },
                uTexture: {
                    value: texture
                }
            }
        });
        this.shadeMaterial = shadeMaterial;
        const plane = new THREE.Mesh(new THREE.PlaneGeometry(2, 1, 128, 128), shadeMaterial);
        this.scene.add(plane);
    }
    init() {
    }
    onRenderer() {
        const elapsed = this.clock.getElapsedTime();
        this.orbit.update();
        this.shadeMaterial.uniforms.uTime.value = elapsed;
    }
}

new ThreeProject(document.getElementById("box"));
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/waveShader.js)

## 小结

- 本文提供 **图像波动** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

