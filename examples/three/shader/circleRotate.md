---
title: "Three.js 旋转的圆教程"
description: "详解 Three.js 旋转的圆：基于 WebGL 实现「旋转的圆」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls、水面反射/镜像材质 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,旋转的圆,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制,水面"
outline: deep
---

### 旋转的圆 · *Circle Rotate* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=circleRotate)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![旋转的圆](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/circleRotate.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- 水面反射/镜像材质
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **旋转的圆** 效果：基于 WebGL 实现「旋转的圆」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls、水面反射/镜像材质。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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

camera.position.set(0, 0, 1.5)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

window.onresize = () => {

  renderer.setSize(box.clientWidth, box.clientHeight)

  camera.aspect = box.clientWidth / box.clientHeight

  camera.updateProjectionMatrix()

}

const uniforms = {

  iTime: {

    value: 0

  },

  iResolution: {

    value: new THREE.Vector2(box.clientWidth, box.clientHeight)

  }
  
}

const geometry = new THREE.PlaneGeometry(1, 1)

const material = new THREE.ShaderMaterial({

  uniforms,

  transparent: true,

  side: THREE.DoubleSide,

  vertexShader: `
      varying vec3 vPosition;
      varying vec2 vUv;
      void main() { 
          vUv = uv; 
          vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
          gl_Position = projectionMatrix * mvPosition;
      }
  `,
  fragmentShader: `
    uniform float iTime;
    uniform vec2 iResolution; 
    varying vec2 iMouse;
    varying vec2 vUv;


    #define PI 3.1415926
    #define NUM 20.
    #define PALETTE vec3(.0, 1.4, 2.)+1.5

    #define COLORED
    #define MIRROR
    //#define ROTATE
    #define ROT_OFST
    #define TRIANGLE_NOISE

    //#define SHOW_TRIANGLE_NOISE_ONLY

    mat2 mm2(in float a){float c = cos(a), s = sin(a);return mat2(c,-s,s,c);}
    float tri(in float x){return abs(fract(x)-.5);}
    vec2 tri2(in vec2 p){return vec2(tri(p.x+tri(p.y*2.)),tri(p.y+tri(p.x*2.)));}
    mat2 m2 = mat2( 0.970,  0.242, -0.242,  0.970 );

    float triangleNoise(in vec2 p)
    {
        float z=1.5;
        float z2=1.5;
      float rz = 0.;
        vec2 bp = p;
      for (float i=0.; i<=3.; i++ )
      {
            vec2 dg = tri2(bp*2.)*.8;
            dg *= mm2(iTime*.3);
            p += dg/z2;

            bp *= 1.6;
            z2 *= .6;
        z *= 1.8;
        p *= 1.2;
            p*= m2;
            
            rz+= (tri(p.x+tri(p.y)))/z;
      }
      return rz;
    }
            
    void main(void) {
        float time = iTime* 1.2;
        float aspect = iResolution.x/iResolution.y;
    float w = 50./sqrt(iResolution.x*aspect+iResolution.y);

        vec2 p = (vUv -0.5) * 2.0 ;
        p.x *= aspect;
        p*= 1.05;
        vec2 bp = p;
        
        #ifdef ROTATE
        p *= mm2(time*.25);
        #endif
        
        float lp = length(p);
        float id = floor(lp*NUM+.5)/NUM;
        
        #ifdef ROT_OFST
        p *= mm2(id*11.);
        #endif
        
        #ifdef MIRROR
        p.y = abs(p.y); 
        #endif
        
        //polar coords
        vec2 plr = vec2(lp, atan(p.y, p.x));
        
        //Draw concentric circles
        float rz = 1.-pow(abs(sin(plr.x*PI*NUM))*1.25/pow(w,0.25),2.5);
        
        //get the current arc length for a given id
        float enp = plr.y+sin((time+id*5.5))*1.52-1.5;
        rz *= smoothstep(0., 0.05, enp);
        
        //smooth out both sides of the arcs (and clamp the number)
        rz *= smoothstep(0.,.022*w/plr.x, enp)*step(id,1.);
        #ifndef MIRROR
        rz *= smoothstep(-0.01,.02*w/plr.x,PI-plr.y);
        #endif
        
        #ifdef TRIANGLE_NOISE
        rz *= (triangleNoise(p/(w*w))*0.9+0.4);
        vec3 col = (sin(PALETTE+id*5.+time)*0.5+0.5)*rz;
        col += smoothstep(.4,1.,rz)*0.15;
        col *= smoothstep(.2,1.,rz)+1.;
        
        #else
        vec3 col = (sin(PALETTE+id*5.+time)*0.5+0.5)*rz;
        col *= smoothstep(.8,1.15,rz)*.7+.8;
        #endif
        
        #ifndef COLORED
        col = vec3(dot(col,vec3(.7)));
        #endif
        
        #ifdef SHOW_TRIANGLE_NOISE_ONLY
        col = vec3(triangleNoise(bp));
        #endif

        // 剔除黑色
        if (col.r < 0.1 && col.g < 0.1 && col.b < 0.1) {
            discard;
        }
        
        gl_FragColor = vec4(col,1.0);
    }
    `
})

const mesh = new THREE.Mesh(geometry, material)

scene.add(mesh)

animate()

function animate() {

  uniforms.iTime.value += 0.01

  requestAnimationFrame(animate)

  controls.update()

  renderer.render(scene, camera)

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/circleRotate.js)

## 小结

- 本文提供 **旋转的圆** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

