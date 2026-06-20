---
title: "Three.js 图片抖动教程"
description: "详解 Three.js 图片抖动：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理，涵盖 ShaderMaterial、THREE.Points、Canvas 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,图片抖动,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,粒子特效,Points,Canvas纹理"
outline: deep
---

### 图片抖动 · *Image Shake* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=imageShake)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![图片抖动](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/imageShake.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- THREE.Points 粒子点渲染
- Canvas 动态纹理贴图
- BufferGeometry 自定义顶点/索引数据
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **图片抖动** 效果：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理；核心用到 ShaderMaterial、THREE.Points、Canvas。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **THREE.Points** 将每个顶点渲染为可控大小的粒子；可用自定义 attribute（如 `u_index`）驱动片元/顶点动画。
- **CanvasTexture** 每帧或按需把 2D Canvas 内容上传 GPU，适合动态文字、图表、视频帧贴图。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
3. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three';
import { TrackballControls } from 'three/examples/jsm/controls/TrackballControls.js';

var container;
var camera, scene, renderer;
var controls;

var shaderUniforms, shaderAttributes;

var particles = [];
var particleSystem;

var imageWidth = 640;
var imageHeight = 400;
var imageData = null;

var animationTime = 0;
var animationDelta = 0.005;

init();
// tick();

function init() {
  createScene();
  createControls();
  createPixelData();

  window.addEventListener('resize', onWindowResize, false);
}

function createScene() {
  container = document.getElementById('box');

  scene = new THREE.Scene();

  camera = new THREE.PerspectiveCamera(20, window.innerWidth / window.innerHeight, 1, 10000);
  camera.position.z = 3000;
  camera.lookAt(scene.position)

  renderer = new THREE.WebGLRenderer({
    antialias: true
  });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setClearColor(0x000000, 1);

  container.appendChild(renderer.domElement);

  scene.add(new THREE.AxesHelper(1000));
  scene.add(new THREE.AmbientLight(0xffffff, 3));
}

function createControls() {
    
  controls = new TrackballControls(camera, renderer.domElement);

  controls.rotateSpeed = 1.0;
  controls.zoomSpeed = 1.2;
  controls.panSpeed = 0.8;

  controls.noZoom = false;
  controls.noPan = true;

  controls.staticMoving = true;
  controls.dynamicDampingFactor = 0.3;
}

function createPixelData() {
    var image = document.createElement("img");
    var canvas = document.createElement("canvas");
    var context = canvas.getContext("2d");
    
    image.crossOrigin = "Anonymous";
    image.onload = function() {
        canvas.width = image.width;
        canvas.height = image.height;
        imageWidth = image.width;
        imageHeight = image.height;
      
      context.fillStyle = context.createPattern(image, 'no-repeat');
      context.fillRect(0, 0, imageWidth, imageHeight);
      
      imageData = context.getImageData(0, 0, imageWidth, imageHeight).data;

      createPaticles();
      tick();
    };
    
    image.src = HOST + 'files/author/z2586300277.png'
  }

  function createPaticles() {
    var colors = [];
    var weights = [0.2126, 0.7152, 0.0722];
    var c = 0;

    var geometry, material;
    var x, y;
    var zRange = 400;

    geometry = new THREE.BufferGeometry();
    var positions = [];
    var colors = [];

    x = imageWidth * -0.5;
    y = imageHeight * 0.5;

    shaderUniforms = {
      amplitude: {
        type: "f",
        value: 0.5
      },
      vertexColor: {
        type: "c",
        value: []
      }
    };

    var shaderMaterial = new THREE.ShaderMaterial({
        uniforms: shaderUniforms,
        vertexShader: `
          uniform float amplitude;
          attribute vec3 customColor;
          varying vec3 vColor;
          void main() {
            vColor = customColor;
            vec4 pos = vec4(position, 1.0);
            pos.z *= amplitude;
            vec4 mvPosition = modelViewMatrix * pos;
            gl_PointSize = 2.0 * ( 300.0 / -mvPosition.z );
            gl_Position = projectionMatrix * mvPosition;
          }
        `,
        fragmentShader: `
          varying vec3 vColor;
          void main() {
            gl_FragColor = vec4( vColor, 1.0 );
          }
        `,
        vertexColors: true
      });
      
      var positions = [];
      var colors = [];
      for (var i = 0; i < imageHeight; i++) {
        for (var j = 0; j < imageWidth; j++) {
          var color = new THREE.Color();
          color.setRGB(imageData[c] / 255, imageData[c + 1] / 255, imageData[c + 2] / 255);
          var weight = color.r * weights[0] +
          color.g * weights[1] +
          color.b * weights[2];

        var vertex = new THREE.Vector3();
        vertex.x = x;
        vertex.y = y;
        vertex.z = (zRange * -0.5) + (zRange * weight);
          positions.push(vertex.x, vertex.y, vertex.z);
          colors.push(color.r, color.g, color.b);
          c += 4;
          x++;
        }
      
        x = imageWidth * -0.5;
        y--;
      }
      var geometry = new THREE.BufferGeometry();
      geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
      geometry.setAttribute('customColor', new THREE.Float32BufferAttribute(colors, 3));
      particleSystem = new THREE.Points(geometry, shaderMaterial);
      scene.add(particleSystem);
  }

  function tick() {
    requestAnimationFrame(tick);
    update();
    render();
  }

  function update() {
    shaderUniforms.amplitude.value = Math.sin(animationTime);
    animationTime += animationDelta;
    controls.update();
  }

  function render() {
    renderer.render(scene, camera);
  }

  function onWindowResize() {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  }
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/imageShake.js)

## 小结

- 本文提供 **图片抖动** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

