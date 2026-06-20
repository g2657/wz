---
title: "Three.js 溶解动画教程"
description: "详解 Three.js 溶解动画：基于 WebGL 实现「溶解动画」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,溶解动画,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 溶解动画 · *Dissolve* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=dissolveAnimate)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![溶解动画](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/dissolveAnimate.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **溶解动画** 效果：基于 WebGL 实现「溶解动画」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js"
import Stats from "three/examples/jsm/libs/stats.module.js"

var scene, camera, renderer, clock, controller, stats
var dissolveMaterials = []

init();
animate();

function init() {
    scene = new THREE.Scene();
    clock = new THREE.Clock();
    camera = new THREE.PerspectiveCamera(
        45,
        window.innerWidth / window.innerHeight,
        0.1,
        1000
    );
    camera.position.set(15, 5, 15)
    renderer = new THREE.WebGLRenderer({
        antialias: true, // 开启抗锯齿处理
        alpha: true,
    });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(window.devicePixelRatio)

    let shader_material = new THREE.ShaderMaterial({
        uniforms: {
            dissolveMap: {
                value: new THREE.TextureLoader().load(FILE_HOST + "threeExamples/shader/tex2.png")
            },
            texture2: {
                value: new THREE.TextureLoader().load(FILE_HOST + "threeExamples/shader/earth1.jpg")
            },
            color: {
                value: new THREE.Color(1, 0, 0)
            },
            time: {
                value: 0
            },
            flag: {
                value: true
            }
        },
        vertexShader: `    varying vec2 vUv;
  varying vec3 worldPos;
  void main() {
      vUv = uv;
      worldPos = (modelMatrix * vec4(position, 1.0)).xyz;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
  }`,
        fragmentShader: `
  varying vec2 vUv;
 
  uniform sampler2D dissolveMap;
  uniform sampler2D texture2;
  uniform float time;
  varying vec3 worldPos;
  uniform bool flag;
  void main() {
    
    float bottom = -2.0;
    float top = 2.0;
    float yScale = (worldPos.y - bottom)/(top - bottom);
 
    vec4 color = texture2D( texture2, vUv);
    //vec4 color = vec4(1.0, 0.0, 0.0, 0.3);
    
    //float t = 1. - fract(time);
    float t;
    if(flag) {
      t = fract(time);
    }else {
      t = 1. - fract(time);
    }
    
    float h = texture2D( dissolveMap, vUv).r;

    float dissolveWidth = 4.0; // 值越大越窄

    float condition_if_1 = step(h + yScale*dissolveWidth, t*(dissolveWidth + 1.0) + 0.04);
    float condition_if_2 = step(h + yScale*dissolveWidth, t*(dissolveWidth + 1.0));
    color = mix(color, vec4(1.0 ,1.0 , 0.0, 1.0), condition_if_1 );
    color = color * (1. - condition_if_2);
    
    gl_FragColor = color;
  }`,
        transparent: true,
        depthWrite: false
    });
    dissolveMaterials.push(shader_material)

    var axisHelper = new THREE.AxesHelper(10);
    scene.add(axisHelper)
    stats = new Stats()
    document.body.appendChild(stats.dom);

    var geometry = new THREE.BoxGeometry(4, 4, 4);
    var cube = new THREE.Mesh(geometry, shader_material);
    scene.add(cube);

    controller = new OrbitControls(camera, renderer.domElement);
    document.body.appendChild(renderer.domElement);
    window.onresize = onResize;
}

function animate() {
    requestAnimationFrame(animate);
    renderer.render(scene, camera);
    stats.update();
    controller.update(clock.getDelta());
    updateMaterial()
}

function updateMaterial() {
    dissolveMaterials.map(m => {
        m.uniforms.time.value += 0.005
        if (m.uniforms.time.value >= 1) {
            m.uniforms.time.value = 0
            m.uniforms.flag.value = !m.uniforms.flag.value
        }
    })
}

function onResize() {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/dissolveAnimate.js)

## 小结

- 本文提供 **溶解动画** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

