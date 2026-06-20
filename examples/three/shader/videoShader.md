---
title: "Three.js 视频着色器教程"
description: "详解 Three.js 视频着色器：基于 WebGL 实现「视频着色器」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,视频着色器,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 视频着色器 · *Video Shader* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=videoShader)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![视频着色器](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/videoShader.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **视频着色器** 效果：基于 WebGL 实现「视频着色器」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
import * as dat from 'dat.gui'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 10, 10)

const renderer = new THREE.WebGLRenderer()

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

window.onresize = () => {

  renderer.setSize(box.clientWidth, box.clientHeight)

  camera.aspect = box.clientWidth / box.clientHeight

  camera.updateProjectionMatrix()
  
}

scene.add(new THREE.AxesHelper(50000)) // 坐标轴

const amibientLight = new THREE.AmbientLight(0xffffff, 4) // 环境光

scene.add(amibientLight) // 添加环境光

const geometry = new THREE.BoxGeometry(5, 5, 5) // 立方体

const video = document.createElement('video')

video.crossOrigin = 'anonymous' // 跨域

video.src = 'https://z2586300277.github.io/3d-file-server/video/test.mp4'

video.loop = true // 循环播放

video.muted = true // 静音

video.play()

const texture = await new Promise(r => video.onloadeddata = () => r(new THREE.VideoTexture(video))) // 创建视频纹理

// 使用 shader 库中的phong材质 进行修改
const shader = {
    
    uniforms: THREE.UniformsUtils.merge([

        THREE.ShaderLib['phong'].uniforms,

        {
            r: {
                type: 'v2',
                value: new THREE.Vector2(box.clientWidth, box.clientHeight)
            },
            t: {
                type: 'f',
                value: 0.0
            },
            colorTexture: { value: texture },
            calcType: {
                value: 2
            }
        }

    ]),

    vertexShader: THREE.ShaderLib['phong'].vertexShader,

    fragmentShader: THREE.ShaderLib['phong'].fragmentShader,

}

// GUI 切换混合运算类型
const GUI = new dat.GUI()

GUI.add(shader.uniforms.calcType, 'value', [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]).name('混合运算类型');

// 动画
animate()

function animate() {

    shader.uniforms.t.value += 0.1

    renderer.render(scene, camera)

    requestAnimationFrame(animate)

}

shader.vertexShader = shader.vertexShader.replace(/#include <common>/, `
    varying vec2 vUv;
    #include <common>    
`)

shader.vertexShader = shader.vertexShader.replace('void main() {', `
    void main() {
    vUv = uv; 
`)

shader.fragmentShader = shader.fragmentShader.replace(/#include <common>/, `
    precision highp float;
    varying vec2 vUv;
    uniform vec2 r;
    uniform float t;
    uniform float calcType;
    uniform sampler2D colorTexture;
    #include <common> 
`)

shader.fragmentShader = shader.fragmentShader.replace('vec4 diffuseColor = vec4( diffuse, opacity );', `
   vec3 c;
    float l,z=t;
    for(int i=0;i<3;i++) {
        vec2 uv,p=gl_FragCoord.xy/r/2.0;
        uv=p +  2.0 * vUv;
        p-=.5;
        p.x*=r.x/r.y;
        z+=.07;
        l=length(p);
        uv+=p/l*(sin(z)+1.)*abs(sin(l*9.-z-z));
        c[i]=.01/length(mod(uv,1.)-.5);
    }
  vec3 color = texture2D( colorTexture, vUv ).rgb;
  vec3 mixedColor;
  if (calcType == 0.0)  mixedColor = max(color, c);
  else if(calcType == 1.0) mixedColor = min(color, c);
  else if(calcType == 2.0) mixedColor = mix(color, c, 0.5);
  else if(calcType == 3.0) mixedColor = mod(color, c);
  else if(calcType == 4.0) mixedColor = pow(color, c);
  else if(calcType == 5.0) mixedColor = step(color, c);
  else if(calcType == 6.0) mixedColor = color + c;
  else if(calcType == 7.0) mixedColor = color - c;
  else if(calcType == 8.0) mixedColor = c  - color;
  else if(calcType == 9.0) mixedColor = color + c - vec3(1.0) * c * color;
  else mixedColor = color;
  vec4 diffuseColor = vec4( diffuse * mixedColor, opacity );
`)

const material = new THREE.ShaderMaterial(shader)

material.lights = true // 光照影响

const mesh = new THREE.Mesh(geometry, material)

scene.add(mesh)
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/videoShader.js)

## 小结

- 本文提供 **视频着色器** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

