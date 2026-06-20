---
title: "Three.js 蛛网箱子教程"
description: "详解 Three.js 蛛网箱子：基于 WebGL 实现「蛛网箱子」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,蛛网箱子,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 蛛网箱子 · *Cobweb Box* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=cobwebBox)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![蛛网箱子](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/cobwebBox.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **蛛网箱子** 效果：基于 WebGL 实现「蛛网箱子」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 0, 5)

const renderer = new THREE.WebGLRenderer()

renderer.setSize(box.clientWidth, box.clientHeight)

new OrbitControls(camera, renderer.domElement)
window.onresize = () => {

  renderer.setSize(box.clientWidth, box.clientHeight)

  camera.aspect = box.clientWidth / box.clientHeight

  camera.updateProjectionMatrix()
  
}

box.appendChild(renderer.domElement)

/* 顶点着色器 */
const vertexShader = `
varying vec2 vUv;
void main(){
	vec4 mPosition=modelMatrix*vec4(position,1.);
	gl_Position=projectionMatrix*viewMatrix*mPosition;
	vUv = uv;
}`

/* 片元着色器 */
const fragmentShader = `
varying vec2 vUv;
uniform float uTime;

void main(){
  vec2 uv=vUv;
  uv-=.5;
  float r=length(uv);// 极径
  float a=atan(uv.x,uv.y);// 极角
  // a += u_time; // 旋转
  vec2 puv=vec2(a,r);// 转换为极坐标 puv.x 范围是 -pi ~ pi
  puv=vec2(puv.x/6.2831+.5,puv.y);// puv.x 范围是 0 ~ 1
  
  vec3 color=vec3(.0);
  // 圆圈
  float circular=fract(puv.y*12.);
  circular=smoothstep(.09,0.,circular);
  // 线
  float line=fract(puv.x*8.);
  line*=1.-line;
  line=line*3.;
  line=smoothstep(.05,0.,line);
  
  line+=circular;
  // line *= step(puv.y,0.5);
  line*=smoothstep(.0,.1,abs(puv.y ));
  color+=line;
  color-=smoothstep(.3,.5,abs(uv.y))+smoothstep(.3,.5,abs(uv.x));

  color *=vec3(0.08, 0.93, 0.11);
  gl_FragColor=vec4(color,1.);
}`

// 自定义材质
const material = new THREE.ShaderMaterial({
  fragmentShader: fragmentShader,
  vertexShader: vertexShader,
  uniforms: {
    uTime: { value: 0 },
  },
  // side: THREE.DoubleSide,
})

const geometry = new THREE.BoxGeometry(2, 2, 2)
const mesh = new THREE.Mesh(geometry, material) //网格模型对象Mesh

scene.add(mesh)

function animate() {
    requestAnimationFrame(animate)
    mesh.rotation.x += 0.02
    mesh.rotation.y += 0.02
    material.uniforms.uTime.value += 0.03
    renderer.render(scene, camera)
}
animate()
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/cobwebBox.js)

## 小结

- 本文提供 **蛛网箱子** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

