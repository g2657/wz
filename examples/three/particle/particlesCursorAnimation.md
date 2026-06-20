---
title: "Three.js 鼠标轨迹粒子教程"
description: "详解 Three.js 鼠标轨迹粒子：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理，涵盖 ShaderMaterial、OrbitControls、THREE.Points 等关键实现，附完整源码与在线 Demo，适合 Three.js 粒子 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,鼠标轨迹粒子,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制,粒子特效"
outline: deep
---

### 鼠标轨迹粒子 · *ParticlesCursorAnimation* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=particle&id=particlesCursorAnimation)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![鼠标轨迹粒子](https://z2586300277.github.io/three-cesium-examples/threeExamples/particle/particlesCursorAnimation.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- THREE.Points 粒子点渲染
- Canvas 动态纹理贴图
- glTF/Draco 模型加载与优化
- Raycaster 鼠标拾取与交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **鼠标轨迹粒子** 效果：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理，支持鼠标拾取、绘制或拖拽交互；核心用到 ShaderMaterial、OrbitControls、THREE.Points。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- **THREE.Points** 将每个顶点渲染为可控大小的粒子；可用自定义 attribute（如 `u_index`）驱动片元/顶点动画。
- **CanvasTexture** 每帧或按需把 2D Canvas 内容上传 GPU，适合动态文字、图表、视频帧贴图。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 异步加载模型 / 3D Tiles / GeoJSON 等资源并加入 scene 或 entities
3. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
4. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
5. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";
const vertexShader=`
precision mediump float;

varying vec3 vNormal;
varying vec3 vPosition;
varying vec2 vUv;
varying vec3 vColor;

uniform vec2 uResolution;
uniform sampler2D uTexture;
uniform sampler2D uDispanecmentTexture;

attribute float aParticlesIntensity;
attribute float aAngle ;

void main(){
    vec3 newPosition=position;
    /*
    Dispanecment
    */
    // 读取canvas纹理中的红色通道作为R，影响粒子偏移
    float dispanecmentIntensity=texture2D(uDispanecmentTexture,uv).r;
    // 关键点：smoothstep
    // 这里0.1是为了过滤掉一些极小值，让图像再鼠标悬浮后能够复原。
    //同时设置一个0.7，让鼠标离开后能够保留粒子偏移后的轨迹一段时间
    dispanecmentIntensity =smoothstep( 0.1,0.7,dispanecmentIntensity);
    // 粒子的偏移向量，这里在xy平面进行随机旋转，在z轴上进行随机偏移
    vec3 displacement=vec3(
        cos(aAngle)*0.2,
        sin(aAngle)*0.2,
        1.0
    );
     displacement=normalize(displacement);
    displacement*=dispanecmentIntensity;
    displacement*=3.0;
    displacement*=aParticlesIntensity;


    newPosition+=displacement;



    float picIntensity=texture2D(uTexture,uv).r;
    
    vec4 modelPosition=modelMatrix*vec4(newPosition,1.);
    vec4 viewPosition=viewMatrix*modelPosition;
    vec4 projectedPosition=projectionMatrix*viewPosition;
    gl_Position=projectedPosition;
    
    // 计算法线变换矩阵：逆矩阵并转置
    mat3 normalMatrix=transpose(inverse(mat3(modelMatrix)));
    // 使用法线变换矩阵变换法线
    vec3 transformedNormal=normalize(normalMatrix*normal);
    vNormal=transformedNormal;
    
    vPosition=modelPosition.xyz;
    
    // 粒子动画的初始模版
    gl_PointSize=0.08*picIntensity*uResolution.y;
    gl_PointSize*=(1./-viewPosition.z);
    

    vColor=vec3(pow(picIntensity,2.0 ));
}
`
const fragmentShader=`
precision mediump float;

uniform vec2 uResolution;

uniform vec3 uSunDirection;
uniform vec3 uAtmosphereDayColor;
uniform vec3 uAtmosphereNightColor;
uniform sampler2D uTexture;

varying vec3 vNormal;
varying vec3 vPosition;
varying vec3 vColor;

void main(){
  vec3 viewDirection=normalize(vPosition-cameraPosition);
  vec3 color=vec3(0.6392, 0.0392, 0.0392);
  vec2 uv=gl_PointCoord;
  float distanceToCenter=distance(uv,vec2(0.5,0.5) );
  if(distanceToCenter>0.5)
    discard;

  // color=vec3(alpha);

  gl_FragColor=vec4(vColor,1.0);
  
  #include <tonemapping_fragment>
  #include <colorspace_fragment>
}
`
/**
 * Base
 */
// Debug

// Canvas
const box = document.getElementById('box')

// Scene
const scene = new THREE.Scene();

/**
 * Sizes
 */
const sizes = {
  width: window.innerWidth,
  height: window.innerHeight,
  resolution: null,
  pixelRatio: Math.min(window.devicePixelRatio, 2),
};
sizes.resolution = new THREE.Vector2(
  window.innerWidth * sizes.pixelRatio,
  window.innerHeight * sizes.pixelRatio
);
/**
 * Loaders
 */
const textureLoader = new THREE.TextureLoader();
const gltfLoader = new GLTFLoader();
/**
 * Diplacement
 */
const dispanecment = {};
dispanecment.canvas = document.createElement("canvas");
//2D Canvas
dispanecment.canvas = document.createElement("canvas");
// 这里应该和绘制图像中的粒子数目保持一致
dispanecment.canvas.width = 128;
dispanecment.canvas.height = 128;
document.body.appendChild(dispanecment.canvas);
dispanecment.canvas.style.position = "fixed";
dispanecment.canvas.style.top = 0;
dispanecment.canvas.style.left = 0;
dispanecment.canvas.style.width = "128px";
dispanecment.canvas.style.height = "128px";
dispanecment.ctx = dispanecment.canvas.getContext("2d");
dispanecment.ctx.fillRect(
  0,
  0,
  dispanecment.canvas.width,
  dispanecment.canvas.height
);
dispanecment.glowImage = new Image();
dispanecment.glowImage.crossOrigin = "Anonymous";
dispanecment.glowImage.src = "https://coderfmc.github.io/three.js-demo/glow.png";
// dispanecment.glowImage.onload = function () {
//   dispanecment.ctx.drawImage(dispanecment.glowImage, 20, 20, 32, 32);
// };

/**
 * Interacitive plane
 */
const interacitivePlane = new THREE.Mesh(
  new THREE.PlaneGeometry(10, 10),
  new THREE.MeshBasicMaterial({
    // 设置双面后，让平面背面也能够进行射线监测，从而具有动画效果
    side: THREE.DoubleSide,
  })
);
interacitivePlane.position.z = -0.01;
scene.add(interacitivePlane);
// 设置不可见
interacitivePlane.visible = false;
/**
 * Raycaster
 */
dispanecment.raycaster = new THREE.Raycaster();
// 设置初始鼠标位置超出屏幕。避免初始位置在屏幕内，有异常发生
dispanecment.screenCursor = new THREE.Vector2(9999, 9999);
dispanecment.canvasCursor = new THREE.Vector2(9999, 9999);
dispanecment.canvasCursorPre = new THREE.Vector2(9999, 9999);
window.addEventListener("pointermove", (e) => {
  // 鼠标位置转换到裁剪坐标,参考gemes101,可以知道为什么要转化到裁剪坐标系
  dispanecment.screenCursor.x = (e.clientX / window.innerWidth) * 2 - 1;
  dispanecment.screenCursor.y = ((e.clientY / window.innerHeight) * 2 - 1) * -1;
  // console.log(screenCursor);
});

/**
 * Texture
 */
dispanecment.texture = new THREE.CanvasTexture(dispanecment.canvas);

const texture = textureLoader.load(
  "https://coderfmc.github.io/three.js-demo/picture-1.png"
);

//model

const material = new THREE.ShaderMaterial({
  vertexShader: vertexShader,
  fragmentShader: fragmentShader,
  uniforms: {
    uResolution: {
      value: sizes.resolution,
    },
    uTime: {
      value: 0,
    },
    uTexture: new THREE.Uniform(texture),
    uDispanecmentTexture: new THREE.Uniform(dispanecment.texture),
  },
});
const particlesGeometry = new THREE.PlaneGeometry(10, 10, 128, 128);

const count = particlesGeometry.attributes.position.count;
const particlesIntensitys = new Float32Array(count);
const angleIntensitys = new Float32Array(count);
for (let i = 0; i < count; i++) {
  particlesIntensitys[i] = Math.random();
  angleIntensitys[i] = Math.random() * Math.PI * 2;
}
particlesGeometry.setAttribute(
  "aParticlesIntensity",
  new THREE.BufferAttribute(particlesIntensitys, 1)
);
particlesGeometry.setAttribute(
  "aAngle",
  new THREE.BufferAttribute(angleIntensitys, 1)
);
//设置用于粒子动画的几何体的索引为空，提高性能
particlesGeometry.setIndex(null);
particlesGeometry.deleteAttribute("normal");
const particles = new THREE.Points(particlesGeometry, material);
scene.add(particles);

window.addEventListener("resize", () => {
  // Update sizes
  sizes.width = window.innerWidth;
  sizes.height = window.innerHeight;
  sizes.pixelRatio = Math.min(window.devicePixelRatio, 2);
  sizes.resolution.set(
    window.innerWidth * sizes.pixelRatio,
    window.innerHeight * sizes.pixelRatio
  );

  // Update camera
  camera.aspect = sizes.width / sizes.height;
  camera.updateProjectionMatrix();

  // Update renderer
  renderer.setSize(sizes.width, sizes.height);
  renderer.setPixelRatio(sizes.pixelRatio);
});

/**
 * Camera
 */
// Base camera
const camera = new THREE.PerspectiveCamera(
  75,
  sizes.width / sizes.height,
  0.1,
  100
);
camera.position.set(0, 0, 7);
scene.add(camera);
scene.add(new THREE.AxesHelper(0, 0, 5));

/**
 * Renderer
 */
const renderer = new THREE.WebGLRenderer({
  antialias: true,
});
renderer.setSize(sizes.width, sizes.height);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.setClearColor("#000011");
box.appendChild(renderer.domElement);
// Controls
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
/**
 * Animate
 */
const clock = new THREE.Clock();

const tick = () => {
  const elapsedTime = clock.getElapsedTime();
  //Raycaster
  dispanecment.raycaster.setFromCamera(dispanecment.screenCursor, camera);
  const intersections =
    dispanecment.raycaster.intersectObject(interacitivePlane);
  if (intersections.length > 0) {
    const uv = intersections[0].uv;

    dispanecment.canvasCursor.x = dispanecment.canvas.width * uv.x;
    // 1-uv.y 反转,让鼠标y轴移动方向与绘制方向一致
    dispanecment.canvasCursor.y = dispanecment.canvas.height * (1 - uv.y);
  }



  /**
   * Diplacement
   */
  const glowSize = dispanecment.canvas.width * 0.2;
  // 设置为默认值，新图形绘制在旧图形上
  dispanecment.ctx.globalCompositeOperation = "source-over";
  // 设置全局透明度为0.1；
  dispanecment.ctx.globalAlpha = 0.02;
  dispanecment.ctx.fillRect(
    0,
    0,
    dispanecment.canvas.width,
    dispanecment.canvas.height
  );


  // cursorDistance 用作鼠标的移动速度，当悬停在图像上时，速度为0，透明度为0，笔迹就不显示，也就没有位移
  const cursorDistance = dispanecment.canvasCursorPre.distanceTo(
    dispanecment.canvasCursor
  );
  dispanecment.canvasCursorPre.copy(dispanecment.canvasCursor);
  const alpha = Math.min(1.0,cursorDistance);
  // globalCompositeOperation='lighten'   保留新绘制图形中的比较亮的颜色
  dispanecment.ctx.globalCompositeOperation = "lighten";
  dispanecment.ctx.globalAlpha = alpha;
  dispanecment.ctx.drawImage(
    dispanecment.glowImage,
    dispanecment.canvasCursor.x - glowSize / 2,
    dispanecment.canvasCursor.y - glowSize / 2,
    glowSize,
    glowSize
  );

  /**
   * Texture
   */
  dispanecment.texture.needsUpdate = true;

  // Update controls
  controls.update();
  // material.uniforms.uTime.value =  elapsedTime;
  // Render
  renderer.render(scene, camera);

  // Call tick again on the next frame
  window.requestAnimationFrame(tick);
};

tick();
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/particle/particlesCursorAnimation.js)

## 小结

- 本文提供 **鼠标轨迹粒子** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

