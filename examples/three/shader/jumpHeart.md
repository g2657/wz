---
title: "Three.js 跳动的心教程"
description: "详解 Three.js 跳动的心：基于 WebGL 实现「跳动的心」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,跳动的心,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 跳动的心 · *Jump Heart* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=jumpHeart)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![跳动的心](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/jumpHeart.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **跳动的心** 效果：基于 WebGL 实现「跳动的心」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
camera.position.set(0, 0, 10)
const renderer = new THREE.WebGLRenderer()
renderer.setSize(box.clientWidth, box.clientHeight)
box.appendChild(renderer.domElement)
new OrbitControls(camera, renderer.domElement)
scene.add(new THREE.AxesHelper(50000))
window.onresize = () => {
    renderer.setSize(box.clientWidth, box.clientHeight)
    camera.aspect = box.clientWidth / box.clientHeight
    camera.updateProjectionMatrix()
}

const { mesh, uniforms } = getShaderMesh()
scene.add(mesh)

animate()
function animate() {
    uniforms.iTime.value += 0.01
    requestAnimationFrame(animate)
    renderer.render(scene, camera)
}

function getShaderMesh() {

    const uniforms = {
        iTime: {
            value: 0
        },
        iResolution: {
            value: new THREE.Vector2(1900, 1900)
        },
        iChannel0: {
            value: window.iChannel0
        }
    }

    const geometry = new THREE.PlaneGeometry(20, 20);
    const material = new THREE.ShaderMaterial({
        uniforms,
        side: 2,
        depthWrite: false,
        transparent: true,
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
		varying vec2 vUv;  
         
        void main(void) {
            vec2 p = (vUv - 0.5 ) * 2.0;
            // vec2 p = (2.0*fragCoord-iResolution.xy)/min(iResolution.y,iResolution.x);
	
            // background color
            vec3 bcol = vec3(1.0,0.8,0.7-0.07*p.y)*(1.0-0.25*length(p));
        
            // animate
            float tt = mod(iTime,1.5)/1.5;
            float ss = pow(tt,.2)*0.5 + 0.5;
            ss = 1.0 + ss*0.5*sin(tt*6.2831*3.0 + p.y*0.5)*exp(-tt*4.0);
            p *= vec2(0.5,1.5) + ss*vec2(0.5,-0.5);
        
            // shape
        #if 0
            p *= 0.8;
            p.y = -0.1 - p.y*1.2 + abs(p.x)*(1.0-abs(p.x));
            float r = length(p);
            float d = 0.5;
        #else
            p.y -= 0.25;
            float a = atan(p.x,p.y)/3.141593;
            float r = length(p);
            float h = abs(a);
            float d = (13.0*h - 22.0*h*h + 10.0*h*h*h)/(6.0-5.0*h);
        #endif
            
            // color
            float s = 0.75 + 0.75*p.x;
            s *= 1.0-0.4*r;
            s = 0.3 + 0.7*s;
            s *= 0.5+0.5*pow( 1.0-clamp(r/d, 0.0, 1.0 ), 0.1 );
            vec3 hcol = vec3(1.0,0.4*r,0.3)*s;
            
            vec3 col = mix( bcol, hcol, smoothstep( -0.01, 0.01, d-r) );
        
            gl_FragColor = vec4(col,1.0);
	        // gl_FragColor = vec4(col*1.3, 1.0);
        }
        `
    })
    const mesh = new THREE.Mesh(geometry, material);

    return {
        mesh,
        uniforms
    }
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/jumpHeart.js)

## 小结

- 本文提供 **跳动的心** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

