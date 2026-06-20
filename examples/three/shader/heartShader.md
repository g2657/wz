---
title: "Three.js 心教程"
description: "详解 Three.js 心：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期，涵盖 ShaderMaterial、EffectComposer、UnrealBloomPass 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,心,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,EffectComposer,后期处理,Bloom"
outline: deep
---

### 心 · *Heart Shader* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=heartShader)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![心](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/heartShader.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- EffectComposer 多 Pass 后期处理管线
- UnrealBloomPass 辉光 Bloom 效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **心** 效果：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期；核心用到 ShaderMaterial、EffectComposer、UnrealBloomPass。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass.js';
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 0, 20)

const renderer = new THREE.WebGLRenderer()

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

const composer = new EffectComposer(renderer);

const renderPass = new RenderPass(scene, camera);

composer.addPass(renderPass);

const bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 5, 1.2, 0)

composer.addPass(bloomPass);

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

}

class HeartCurve extends THREE.Curve {
    constructor(scale = 1) {
        super();
        this.scale = scale;
    }

    getPoint(a, optionalTarget = new THREE.Vector3()) {
        const t = a * Math.PI * 2;
        const tx = 16 * Math.pow(Math.sin(t), 3);
        const ty = 13 * Math.cos(t) - 5 * Math.cos(2 * t) - 2 * Math.cos(3 * t) - Math.cos(4 * t);
        const tz = 0;

        return optionalTarget.set(tx, ty, tz).multiplyScalar(this.scale);
    }
}

const geometry = new THREE.TubeGeometry(new HeartCurve(0.5), 100, 0.1, 30, false);

const vertexShader = `
    varying vec2 vUv;
    void main(void) {
        vUv = uv;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }`;
const fragmentShader = `
    varying vec2 vUv;
    uniform float uSpeed;
    uniform float uTime;
    uniform vec2 uFade;
    uniform vec3 uColor;
    uniform float uDirection;
    void main() {
    vec3 color =uColor;
    float s=0.0;
    float v=0.0;
    if(uDirection==1.0){
    v=vUv.x;
    s=-uTime*uSpeed;
    }else{
    v= -vUv.x;
    s= -uTime*uSpeed;
    }
    float d=mod(  (v + s) ,1.0) ;
    if(d>uFade.y)discard;
    else{
    float alpha = smoothstep(uFade.x, uFade.y, d);
    if (alpha < 0.0001) discard;
    gl_FragColor =  vec4(color,alpha);
    }
} `;

const getMaterial = (uniforms) => {
    return new THREE.ShaderMaterial({
        uniforms,
        vertexShader,
        fragmentShader,
        transparent: true
    });
}

const material1 = getMaterial({
    uColor: { value: new THREE.Color('pink') },
    uTime: { value: 0 },
    uDirection: { value: 1 },
    uSpeed: { value: 1 },
    uFade: { value: new THREE.Vector2(0, 0.5) },
    uDirection: { value: 1 },
    uSpeed: { value: 1 }
})

const material2 = getMaterial({
    uColor: { value: new THREE.Color('#00BFFF') },
    uTime: { value: 0.5 },
    uDirection: { value: 1 },
    uSpeed: { value: 1 },
    uFade: { value: new THREE.Vector2(0, 0.5) },
    uDirection: { value: 1 },
    uSpeed: { value: 1 }
})


const mesh = new THREE.Mesh(geometry, material1)
const mesh2 = new THREE.Mesh(geometry, material2)

scene.add(mesh);

scene.add(mesh2)


animate()

function animate() {

    material1.uniforms.uTime.value += 0.002
    material2.uniforms.uTime.value += 0.002

    requestAnimationFrame(animate)

    renderer.render(scene, camera)

    composer.render();

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/heartShader.js)

## 小结

- 本文提供 **心** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

