---
title: "Three.js 下雨效果教程"
description: "详解 Three.js 下雨效果：基于 WebGL 实现「下雨效果」可视化效果，附完整可运行源码，涵盖 onBeforeCompile、OrbitControls、BufferGeometry 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,下雨效果,WebGL,源码,教程,在线案例,onBeforeCompile,shader注入,OrbitControls,相机控制,BufferGeometry"
outline: deep
---

### 下雨效果 · *Rain Roof* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=rainRoof)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![下雨效果](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/rainRoof.jpg)

## 你将学到什么

- onBeforeCompile 注入 GLSL 改造内置材质
- OrbitControls 相机轨道交互
- BufferGeometry 自定义顶点/索引数据
- 监听窗口 `resize` 同步更新 camera 与 renderer

## 效果说明

本案例演示 **下雨效果** 效果：基于 WebGL 实现「下雨效果」可视化效果，附完整可运行源码；核心用到 onBeforeCompile、OrbitControls、BufferGeometry。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **onBeforeCompile** 在 Three 拼好内置 shader 后替换 `#include <xxx>` 片段，适合在 PBR 材质上叠加大屏特效。
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

class DepthData extends THREE.WebGLRenderTarget{
  constructor(size, camParams){
    super(size, size);
    this.texture.minFilter = THREE.NearestFilter;
    this.texture.magFilter = THREE.NearestFilter;
    this.stencilBuffer = false;
    this.depthTexture = new THREE.DepthTexture();
    this.depthTexture.format = THREE.DepthFormat;
    this.depthTexture.type = THREE.UnsignedIntType;
    
    let hw = camParams.width * 0.5;
    let hh = camParams.height * 0.5;
    let d = camParams.depth;
    this.depthCam = new THREE.OrthographicCamera(-hw, hw, hh, -hh, 0, d);
    this.depthCam.layers.set(1);
    this.depthCam.position.set(0, d, 0);
    this.depthCam.lookAt(0, 0, 0);
  }
  
  update(){
    renderer.setRenderTarget(this);
    renderer.render(scene, this.depthCam);
    renderer.setRenderTarget(null);
  }
}

class Rain extends THREE.Line{
  constructor(size, amount){
    let v = new THREE.Vector3();
    let gBase = new THREE.BufferGeometry().setFromPoints([new THREE.Vector2(0, 0), new THREE.Vector2(0, 1)])
    let g = new THREE.InstancedBufferGeometry().copy(gBase)
    g.setAttribute("instPos", new THREE.InstancedBufferAttribute(
      new Float32Array(
        Array.from({length: amount}, () => {
          v.random().subScalar(0.5);
          v.y += 0.5;
          v.multiply(size);
          return [...v];
        }).flat()
      ), 3
    ))
    g.instanceCount = amount;
    
    let m = new THREE.LineBasicMaterial({
      color: 0x4488ff,
      transparent: true,
      onBeforeCompile: shader => {
        shader.uniforms.depthData = gu.depthData;
        shader.uniforms.time = gu.time;
        shader.vertexShader = `
          uniform float time;
          
          attribute vec3 instPos;
          
          varying float colorTransition;
          varying vec3 vPos;
          ${shader.vertexShader}
        `.replace(
          `#include <begin_vertex>`,
          `#include <begin_vertex>
          
          float t = time;
          vec3 iPos = instPos;
          iPos.y = mod(20. - instPos.y - t * 5., 20.);
          
          transformed.y *= 0.5;
          transformed += iPos;
          
          vPos = transformed;
          
          colorTransition = position.y;
          `
        );
        //console.log(shader.vertexShader);
        
        shader.fragmentShader = `
          uniform sampler2D depthData;
          varying float colorTransition;
          varying vec3 vPos;
          ${shader.fragmentShader}
        `.replace(
          `vec4 diffuseColor = vec4( diffuse, opacity );`,
          `
          vec2 depthUV = (vPos.xz + 10.) / 20.;
          depthUV.y = 1. - depthUV.y;
          
          float depthVal = 1. - texture(depthData, depthUV).r;
          float actualDepth = depthVal * 20.;
          
          if(vPos.y < actualDepth) discard;
          
          float trns = 1. - colorTransition;
          
          float distVal = smoothstep(3., 0., vPos.y - actualDepth);
          vec3 col = mix(diffuse, vec3(0.9), distVal); // the closer, the whiter
          vec4 diffuseColor = vec4( mix(col, col + 0.1, pow(trns, 16.)), (opacity * (0.25 + 0.75 * distVal)) * trns );
          `
        );
        //console.log(shader.fragmentShader);
      }
    })
    super(g, m);
    this.frustumCulled = false;
  }
}

let scene = new THREE.Scene();
let camera = new THREE.PerspectiveCamera(30, innerWidth / innerHeight, 1, 1000);
camera.position.set(3, 5, 13).setLength(50);
let renderer = new THREE.WebGLRenderer({antialias: true});
renderer.setSize(innerWidth, innerHeight);
document.body.appendChild(renderer.domElement);

window.addEventListener("resize", event => {
  camera.aspect = innerWidth / innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth, innerHeight);
})

let camShift = new THREE.Vector3(0, 7, 0);
camera.position.add(camShift);
let controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.target.copy(camShift);

let light = new THREE.DirectionalLight(0xffffff, Math.PI);
light.position.setScalar(1);
scene.add(light, new THREE.AmbientLight(0xffffff, Math.PI * 0.5));

let maxHeight = 20;
let depthData = new DepthData(1024, {width: 20, height: 20, depth: maxHeight});

let gu = {
  time: {value: 0},
  depthData: {value: depthData.depthTexture}
}


let lawn = new THREE.Mesh(
  new THREE.PlaneGeometry(20, 20).rotateX(-Math.PI * 0.5),
  new THREE.MeshLambertMaterial({
    map: new THREE.TextureLoader().load(FILE_HOST + 'images/texture/g1.png'),
    //map: gu.depthData.value
  })
)
lawn.layers.enable(1);
scene.add(lawn);

let tent = new THREE.Mesh(
  new THREE.PlaneGeometry(10, 10).rotateX(-Math.PI * 0.65).translate(0, 10, 0),
  new THREE.MeshLambertMaterial({
    map: new THREE.TextureLoader().load(FILE_HOST + 'images/texture/g1.png'),
  })
)
tent.layers.enable(1);
tent.position.set(-2, 2, 0);
tent.rotation.y = Math.PI * 0.25;
scene.add(tent);

let rain = new Rain(new THREE.Vector3(20, 20, 20), 50000);
scene.add(rain);

let clock = new THREE.Clock();
let t = 0;

renderer.setAnimationLoop(() => {
  let dt = clock.getDelta();
  t += dt;
  gu.time.value = t;
  
  controls.update();
  
  tent.position.x = Math.sin(t * 0.2) * 7;
  
  depthData.update();
  renderer.render(scene, camera);
})
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/rainRoof.js)

## 小结

- 本文提供 **下雨效果** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

