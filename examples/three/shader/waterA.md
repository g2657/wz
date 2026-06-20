---
title: "Three.js 波浪效果教程"
description: "详解 Three.js 波浪效果：基于 WebGL 实现「波浪效果」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,波浪效果,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 波浪效果 · *Water Effect* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=waterA)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![波浪效果](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/waterA.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **波浪效果** 效果：基于 WebGL 实现「波浪效果」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js' 
import * as dat from 'dat.gui'
const gui = new dat.GUI();

const box = document.getElementById("box");


// 假设typeState已经被定义
const typeState = {
  color: "#0670c1",
  speed: 0.1,
  brightness: 1.5,
  flowSpeed: { x: 0.01, y: 0.01 },
};

let vertexShader = `// Examples of variables passed from vertex to fragment shader
varying vec2 vUv;

void main(){
	// To pass variables to the fragment shader, you assign them here in the
	// main function. Traditionally you name the varying with vAttributeName
	vUv=uv;
	
	// This sets the position of the vertex in 3d space. The correct math is
	// provided below to take into account camera and object data.
	gl_Position=projectionMatrix*modelViewMatrix*vec4(position,1.);
}`;
let fragmentShader = `#define TAU 6.28318530718
#define MAX_ITER 5

uniform vec2 resolution;
uniform vec3 backgroundColor;
uniform vec3 color;
uniform float speed;
uniform vec2 flowSpeed;
uniform float brightness;
uniform float time;

varying vec2 vUv;

void main(){
	vec2 uv=(vUv.xy+(time*flowSpeed))*resolution;
	
	vec2 p=mod(uv*TAU,TAU)-250.;
	vec2 i=vec2(p);
	
	float c=1.;
	float inten=.005;
	
	for(int n=0;n<MAX_ITER;n++){
		float t=time*speed*(1.-(3.5/float(n+1)));
		i=p+vec2(cos(t-i.x)+sin(t+i.y),sin(t-i.y)+cos(t+i.x));
		c+=1./length(vec2(p.x/(sin(i.x+t)/inten),p.y/(cos(i.y+t)/inten)));
	}
	
	c/=float(MAX_ITER);
	c=1.17-pow(c,brightness);
	
	vec3 rgb=vec3(pow(abs(c),8.));
	
	gl_FragColor=vec4(rgb*color+backgroundColor,length(rgb)+.1);
}`;

// 创建场景
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(
  45,
  window.innerWidth / window.innerHeight,
  0.1,
  10000
);
camera.position.set(15, 15, 15);

// 创建渲染器
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(box.clientWidth, box.clientHeight);
renderer.setClearColor("#201919"); 
box.appendChild(renderer.domElement);
// 添加环境光
const ambientLight = new THREE.AmbientLight(0x0a0a0a0);
scene.add(ambientLight);

// 创建控制器
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

// 创建平面几何体
const geometry = new THREE.PlaneGeometry(10, 10);

// 创建着色器材质
const material = new THREE.ShaderMaterial({
  uniforms: {
    resolution: { value: new THREE.Vector2(1, 1) },
    backgroundColor: { value: new THREE.Color("#0670c1") },
    color: { value: new THREE.Color("#fff") },
    speed: { value: 0.1 },
    flowSpeed: { value: new THREE.Vector2(0.01, 0.01) },
    brightness: { value: 1.5 },
    time: { value: 0.1 },
  },
  vertexShader: vertexShader,
  fragmentShader: fragmentShader,
  side: THREE.DoubleSide,
  transparent: true,
  depthWrite: false,
  depthTest: true,
});

// 创建网格
const mesh = new THREE.Mesh(geometry, material);
scene.add(mesh);
// 设置网格的旋转，注意Three.js中旋转的单位是弧度
mesh.rotation.x = -Math.PI / 2; // 将x轴旋转-90度

// 设置网格的位置，这里设置y坐标为1
mesh.position.y = 1;

// 添加辅助网格
const gridHelper = new THREE.GridHelper(10, 10);
scene.add(gridHelper);

// 渲染循环
function animate() {
  requestAnimationFrame(animate);
  controls.update();
  // 更新时间
  material.uniforms.time.value += 0.1;

  renderer.render(scene, camera);
}
animate();

// 创建颜色控制器
const colorController = gui
  .addColor(typeState, "color")
  .onChange((value) => {
    console.log(`颜色值改变为: ${value}`);

    material.uniforms.backgroundColor.value = new THREE.Color(value);
  })
  .name("颜色");

// 创建速度控制器，设置最小值、最大值和步长
const speedController = gui
  .add(typeState, "speed", 0.1, 1, 0.1)
  .onChange((value) => {
    console.log(`速度值改变为: ${value}`);
    material.uniforms.speed.value = value;
  })
  .name("速度");

// 创建亮度控制器，设置最小值、最大值和步长
const brightnessController = gui
  .add(typeState, "brightness", 0.1, 2, 0.1)
  .onChange((value) => {
    console.log(`亮度值改变为: ${value}`);
    material.uniforms.brightness.value = value;
  })
  .name("亮度");

// 创建流动速度控制器，dat.GUI不支持倒置的控制器，因此需要手动处理
const flowSpeedFolder = gui.addFolder("流动速度");
const flowSpeedXController = flowSpeedFolder
  .add(typeState.flowSpeed, "x", 0.01, 0.6, 0.02)
  .onChange((value) => {
    console.log(`流动速度X值改变为: ${value}`);
    material.uniforms.flowSpeed.value.x = value;
  });
const flowSpeedYController = flowSpeedFolder
  .add(typeState.flowSpeed, "y", 0.01, 0.6, 0.02)
  .onChange((value) => {
    console.log(`流动速度Y值改变为: ${value}`);
    material.uniforms.flowSpeed.value.y = value;
  });
// 手动处理倒置逻辑
flowSpeedXController.__min = 0.6;
flowSpeedXController.__max = 0.01;
flowSpeedYController.__min = 0.6;
flowSpeedYController.__max = 0.01;
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/waterA.js)

## 小结

- 本文提供 **波浪效果** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

