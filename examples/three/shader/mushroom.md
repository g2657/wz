---
title: "Three.js 蘑菇教程"
description: "详解 Three.js 蘑菇：基于 WebGL 实现「蘑菇」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,蘑菇,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 蘑菇 · *Mushroom* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=mushroom)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![蘑菇](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/mushroom.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **蘑菇** 效果：基于 WebGL 实现「蘑菇」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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

const DOM = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(50, DOM.clientWidth / DOM.clientHeight, 0.1, 100000)
camera.position.set(10, 10, 10)
scene.add(camera);

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })
renderer.setSize(DOM.clientWidth, DOM.clientHeight)
renderer.setPixelRatio(window.devicePixelRatio * 2)
renderer.setClearColor(0x000000)
DOM.appendChild(renderer.domElement)
new OrbitControls(camera, renderer.domElement)

const uniforms = {
    Mouse: {
        type: 'v2',
        value: new THREE.Vector2(0, 0)
    },
    Resolution: {
        type: 'v2',
        value: new THREE.Vector2(window.innerWidth, window.innerHeight)
    },
    Time: {
        type: 'f',
        value: 1.0
    }
}

DOM.addEventListener('mousemove', (event) => uniforms.Mouse.value = new THREE.Vector2(
    (event.offsetX / event.target.clientWidth) * 2 - 1,
    -(event.offsetY / event.target.clientHeight) * 2 + 1
))
const geometry = new THREE.BoxGeometry(10, 10, 10);

var material = new THREE.ShaderMaterial({
    uniforms: uniforms,
    vertexShader: `varying vec2 vUv;
    void main(){
        gl_Position=projectionMatrix*modelViewMatrix*vec4(position,1.);
        vUv=uv;
    }`,
    fragmentShader: `#ifdef GL_ES
    precision mediump float;
    #endif
    uniform vec2 Resolution;
    uniform vec3 Mouse;
    uniform float Time;
    varying vec2 vUv;
    
    mat2 rot2D(float angle){
      float s=sin(angle);
      float c=cos(angle);
      return mat2(c,-s,s,c);
    }
    
    float sdCutHollowSphere(vec3 p,float r,float h,float t)
    {
      float w=sqrt(r*r-h*h);
      vec2 q=vec2(length(p.xz),p.y);
      return((h*q.x<w*q.y)?length(q-vec2(w,h)):
      abs(length(q)-r))-t;
    }
    vec4 sdstripe(vec3 p,vec3 color){
      p.xz=abs(p.xz);
      float d1=sdCutHollowSphere(p-vec3(.0,-3.3,0.),.8,.01,.01);
      float d2=sdCutHollowSphere(p-vec3(.9,-3.3,.9),.5,.005,.01);
      float d=min(d1,d2);
      return vec4(d,color);
    }
    vec4 sdCutSphere(vec3 p,float r,float h,vec3 color)
    {
      
      float w=sqrt(r*r-h*h);
      
      vec2 q=vec2(length(p.xz),p.y);
      float s=max((h-r)*q.x*q.x+w*w*(h+r-2.*q.y),h*q.x-w*q.y);
      float d=(s<0.)?length(q)-r:
      (q.x<w)?h-q.y:
      length(q-vec2(w,h));
      
      return vec4(d,color);
    }
    vec4 sdPlane(vec3 p,vec3 color){
      return vec4(-p.y+.2,color);
      
    }
    vec4 sdCappedCone(vec3 p,vec3 a,vec3 b,float ra,float rb,vec3 color)
    {
      float rba=rb-ra;
      float baba=dot(b-a,b-a);
      float papa=dot(p-a,p-a);
      float paba=dot(p-a,b-a)/baba;
      float x=sqrt(papa-paba*paba*baba);
      float cax=max(0.,x-((paba<.5)?ra:rb));
      float cay=abs(paba-.5)-.5;
      float k=rba*rba+baba;
      float f=clamp((rba*(x-ra)+paba*baba)/k,0.,1.);
      float cbx=x-ra-f*rba;
      float cby=paba-f;
      float s=(cbx<0.&&cay<0.)?-1.:1.;
      return vec4(s*sqrt(min(cax*cax+cay*cay*baba,
          cbx*cbx+cby*cby*baba)),color);
        }
        float smin(float d1,float d2,float k){
          float h=clamp(.5+.5*(d2-d1)/k,0.,1.);
          return mix(d2,d1,h)-k*h*(1.-h);
        }
        vec4 colorMin(vec4 a,vec4 b){
          if(a.x<b.x){
            return a;
          }else{
            return b;
          }
        }
        //模糊摆动，y的值越大，摆动频率越大
        vec3 bendPoint(vec3 p,float k)
        {
          float c=cos(k*p.y);
          float s=sin(k*p.y);
          mat2 m=mat2(c,-s,s,c);
          vec3 q=vec3(m*p.xy,p.z);
          return q;
        }
        vec4 map(vec3 p){
          vec3 q=p;
          p=bendPoint(p,sin(Time*5.));
          vec3 pp2=vec3(0.,.8,0.);
          vec3 pp1=vec3(0.,-.2,0.);
          vec4 CappedConesdf=sdCappedCone(-p,pp1,pp2,.2,.1,vec3(.8667,.8667,.7216));
          vec4 CutSpheresdf=sdCutSphere(-p-vec3(0.,.4,0.),.5,.2,vec3(.9608,.4667,.4))-.1;
          vec4 entity=colorMin(CappedConesdf,CutSpheresdf);
          entity=colorMin(entity,sdstripe(p*4.,vec3(3.5))/4.);
          entity=colorMin(entity,sdPlane(q,vec3(.4196,.5529,.3647)));
          return entity;
        }
        
        void main(){
          vec3 ro=vec3(0.,0.,-8.);//起始位置
          vec3 rd=normalize(vec3(vUv-.5,1.));//方向
          ro.xz*=rot2D(-Time);
          rd.xz*=rot2D(-Time);
          ro.y-=4.;
          rd.y+=.5;
          float t=0.;
          vec4 color=vec4(0.);
          for(int i=0;i<80;i++){
            vec3 p=ro+rd*t;
            vec4 d=map(p)/1.8;
            t+=d.x;
            //优化效率
            if(t>100.||d.x<.001){
              break;
            }
            color=vec4(t*d.yzw*.13,1.);
          }
          
          gl_FragColor=color;
          
        }`,
    uniforms
});

material.side = THREE.DoubleSide

var mesh = new THREE.Mesh(geometry, material);
mesh.scale.set(1, -1, 1)
scene.add(mesh);

render()
function render() {
    uniforms.Time.value += 0.005;
    renderer.render(scene, camera)
    requestAnimationFrame(render)
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/mushroom.js)

## 小结

- 本文提供 **蘑菇** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

