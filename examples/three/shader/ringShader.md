---
title: "Three.js 环形着色器教程"
description: "详解 Three.js 环形着色器：基于 WebGL 实现「环形着色器」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,环形着色器,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 环形着色器 · *Ring Shader* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=ringShader)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![环形着色器](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/ringShader.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **环形着色器** 效果：基于 WebGL 实现「环形着色器」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
camera.position.set(0, 0, 15)
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
        uniform float ratio;

        float M_PI = 3.1415926;
        float M_TWO_PI = 6.28318530718;
        vec3 iMouse = vec3(0.0, 0.0 ,0.0 );
		uniform float iTime;
		uniform vec2 iResolution; 
          
		varying vec2 vUv;
        float rand(vec2 n) {
            return fract(sin(dot(n, vec2(12.9898,12.1414))) * 83758.5453);
        }
        
        float noise(vec2 n) {
            const vec2 d = vec2(0.0, 1.0);
            vec2 b = floor(n);
            vec2 f = smoothstep(vec2(0.0), vec2(1.0), fract(n));
            return mix(mix(rand(b), rand(b + d.yx), f.x), mix(rand(b + d.xy), rand(b + d.yy), f.x), f.y);
        }
        
        vec3 ramp(float t) {
            return t <= .5 ? vec3( 1. - t * 1.4, .2, 1.05 ) / t : vec3( .3 * (1. - t) * 2., .2, 1.05 ) / t;
        }
        vec2 polarMap(vec2 uv, float shift, float inner) {
        
            uv = vec2(0.5) - uv;
            
            
            float px = 1.0 - fract(atan(uv.y, uv.x) / 6.28 + 0.25) + shift;
            float py = (sqrt(uv.x * uv.x + uv.y * uv.y) * (1.0 + inner * 2.0) - inner) * 2.0;
            
            return vec2(px, py);
        }
        float fire(vec2 n) {
            return noise(n) + noise(n * 2.1) * .6 + noise(n * 5.4) * .42;
        }
        
        float shade(vec2 uv, float t) {
            uv.x += uv.y < .5 ? 23.0 + t * .035 : -11.0 + t * .03;    
            uv.y = abs(uv.y - .5);
            uv.x *= 35.0;
            
            float q = fire(uv - t * .013) / 2.0;
            vec2 r = vec2(fire(uv + q / 2.0 + t - uv.x - uv.y), fire(uv + q - t));
            
            return pow((r.y + r.y) * max(.0, uv.y) + .1, 4.0);
        }
        
        vec3 color(float grad) {
            
            float m2 = iMouse.z < 0.0001 ? 1.15 : iMouse.y * 3.0 / iResolution.y;
            grad =sqrt( grad);
            vec3 color = vec3(1.0 / (pow(vec3(0.5, 0.0, .1) + 2.61, vec3(2.0))));
            vec3 color2 = color;
            color = ramp(grad);
            color /= (m2 + max(vec3(0), color));
            
            return color;
        
        }
        
         
        float distanceTo(vec2 src, vec2 dst) {
			float dx = src.x - dst.x;
			float dy = src.y - dst.y;
			float dv = dx * dx + dy * dy;
			return sqrt(dv);
		}

        
		void main() { 
            float m1 = iMouse.z < 0.0001 ? 3.6 : iMouse.x * 5.0 / iResolution.x;
    
            float t = iTime;
            vec2 uv = vUv;
            float ff = 1.0 - uv.y;
            uv.x -= (iResolution.x / iResolution.y - 1.0) / 2.0;
            vec2 uv2 = uv;
            uv2.y = 1.0 - uv2.y;
            uv = polarMap(uv, 1.3, m1);
            uv2 = polarMap(uv2, 1.9, m1);

            vec3 c1 = color(shade(uv, t)) * ff;
            vec3 c2 = color(shade(uv2, t)) * (1.0 - ff);
             

			gl_FragColor = vec4(c1 + c2, 1.0);;
			
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

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/ringShader.js)

## 小结

- 本文提供 **环形着色器** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

