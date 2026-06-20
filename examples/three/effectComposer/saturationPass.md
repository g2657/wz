---
title: "Three.js 饱和度(自定义Pass)教程"
description: "详解 Three.js 饱和度(自定义Pass)：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期，涵盖 ShaderMaterial、EffectComposer、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 后期处理 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,饱和度(自定义Pass),WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,EffectComposer,后期处理,OrbitControls"
outline: deep
---

### 饱和度(自定义Pass) · *Saturation* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=effectComposer&id=saturationPass)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![饱和度(自定义Pass)](https://z2586300277.github.io/three-cesium-examples/threeExamples/effectComposer/saturationPass.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- EffectComposer 多 Pass 后期处理管线
- OrbitControls 相机轨道交互
- 自定义 ShaderPass 调色/特效
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **饱和度(自定义Pass)** 效果：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期；核心用到 ShaderMaterial、EffectComposer、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **EffectComposer** 以多 Pass 链式渲染：RenderPass → 特效 Pass → 输出屏幕，替代直接 `renderer.render`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
3. 组装 EffectComposer Pass 链，在 animate 中调用 `composer.render()`
4. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
5. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer.js";
import { RenderPass } from "three/examples/jsm/postprocessing/RenderPass.js";
import { OutputPass } from "three/examples/jsm/postprocessing/OutputPass.js";
import { ShaderPass } from "three/examples/jsm/postprocessing/ShaderPass.js";
import { GUI } from "three/examples/jsm/libs/lil-gui.module.min.js";

window.addEventListener('load', e => {
    init();
    initComposer();
    addMesh();
    render();
})

let scene, renderer, camera, orbit;
let composer;

function init() {
    scene = new THREE.Scene();
    renderer = new THREE.WebGLRenderer({ alpha: true, antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    document.body.appendChild(renderer.domElement);

    camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 2000);
    camera.add(new THREE.PointLight(0xffffff, 1, 1000, 0.01));
    camera.position.set(10, 10, 10);
    scene.add(camera);

    orbit = new OrbitControls(camera, renderer.domElement);
    orbit.enableDamping = true;

    scene.add(new THREE.GridHelper(10, 10));
}

function initComposer() {
    composer = new EffectComposer(renderer);
    let renderPass = new RenderPass(scene, camera);
    let outputPass = new OutputPass();
    composer.addPass(renderPass);
    composer.addPass(outputPass);

    let finalMaterial = new THREE.ShaderMaterial({
        uniforms: {
            baseTexture: { value: null },
            saturation: { value: 0.2 },
            brightness: { value: 0.2 }
        },
        vertexShader: `
            varying vec2 vUv;
            void main(){
                vUv = uv;
                gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
            }
        `,
        fragmentShader: `

            vec3 hsv2rgb( in vec3 c ){
                vec3 rgb = clamp( abs(mod(c.x*6.0+vec3(0.0,4.0,2.0),6.0)-3.0)-1.0, 0.0, 1.0 );

                return c.z * mix( vec3(1.0), rgb, c.y);
            }

            vec3 rgbToHsv(vec3 rgb){
                vec3 hsv = vec3(0);
                float maxC = max(max(rgb.r,rgb.g),rgb.b);
                float minC = min(min(rgb.r,rgb.g),rgb.b);
                float delta = maxC - minC;
                if (maxC == rgb.r) hsv.x = mod((rgb.g - rgb.b)/delta,6.0)/6.0;
                if (maxC == rgb.g) hsv.x = (rgb.b - rgb.r)/(delta*6.0) + 1.0/3.0;
                if (maxC == rgb.b) hsv.x = (rgb.r - rgb.g)/(delta*6.0) + 2.0/3.0;
                hsv.y = delta/maxC;
                hsv.z = maxC;
                return hsv;
            }

            varying vec2 vUv;
            uniform sampler2D tDiffuse;
            uniform float saturation;
            uniform float brightness;
            void main(){
                vec4 col = texture2D(tDiffuse,vUv);

                vec3 hsvCol = rgbToHsv(col.rgb);
                hsvCol.y *= saturation;
                hsvCol.z *= brightness;
                vec3 col2 = hsv2rgb(hsvCol);
                gl_FragColor = vec4(col2,col.a);

                #include <colorspace_fragment>
            }
        `
    });

    let finalPass = new ShaderPass(finalMaterial)
    composer.addPass(finalPass);

    let gui = new GUI();
    gui.add(finalMaterial.uniforms.saturation, 'value', 0, 1, 0.01).name('饱和度')
    gui.add(finalMaterial.uniforms.brightness, 'value', 0, 1, 0.01).name('亮度')

}

function addMesh() {
    let geometry = new THREE.BoxGeometry(1, 1, 1);

    for (let i = 0; i < 100; i++) {
        let material = new THREE.MeshStandardMaterial({ color: 0xffffff * Math.random() });
        let mesh = new THREE.Mesh(geometry, material);
        mesh.position.x = Math.random() * 10 - 5;
        mesh.position.y = Math.random() * 10 - 5;
        mesh.position.z = Math.random() * 10 - 5;
        scene.add(mesh);
    }
}

function render() {
    // renderer.render(scene,camera);
    composer.render();
    orbit.update();
    requestAnimationFrame(render);
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/effectComposer/saturationPass.js)

## 小结

- 本文提供 **饱和度(自定义Pass)** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

