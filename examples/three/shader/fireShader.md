---
title: "Three.js 火焰教程"
description: "详解 Three.js 火焰：基于 WebGL 实现「火焰」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,火焰,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 火焰 · *Fire Shader* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=fireShader)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![火焰](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/fireShader.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **火焰** 效果：基于 WebGL 实现「火焰」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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

camera.position.set(0, 0, 0.6)

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
    const float PI = 3.14159265359; 
        
    uniform float iTime;
    uniform vec2 iResolution; 
      
    varying vec2 vUv;
    
    vec3 firePalette(float i){
    
        float T = 1400. + 1300.*i; // Temperature range (in Kelvin).
        vec3 L = vec3(7.4, 5.6, 4.4); // Red, green, blue wavelengths (in hundreds of nanometers).
        L = pow(L,vec3(5)) * (exp(1.43876719683e5/(T*L)) - 1.);
        return 1. - exp(-5e8/L); // Exposure level. Set to "50." For "70," change the "5" to a "7," etc.
    } 
    vec3 hash33(vec3 p){ 
        
        float n = sin(dot(p, vec3(7, 157, 113)));    
        return fract(vec3(2097152, 262144, 32768)*n); 
    }
    
    float voronoi(vec3 p){
    
        vec3 b, r, g = floor(p);
        p = fract(p); // "p -= g;" works on some GPUs, but not all, for some annoying reason.
        
        float d = 1.;  
        for(int j = -1; j <= 1; j++) {
            for(int i = -1; i <= 1; i++) {
                
                b = vec3(i, j, -1);
                r = b - p + hash33(g+b);
                d = min(d, dot(r,r));
                
                b.z = 0.0;
                r = b - p + hash33(g+b);
                d = min(d, dot(r,r));
                
                b.z = 1.;
                r = b - p + hash33(g+b);
                d = min(d, dot(r,r));
                    
            }
        }
        
        return d; // Range: [0, 1]
    }
    
    float noiseLayers(in vec3 p) {
    
        vec3 t = vec3(0., 0., p.z + iTime*1.5);
    
        const int iter = 5; // Just five layers is enough.
        float tot = 0., sum = 0., amp = 1.; // Total, sum, amplitude.
    
        for (int i = 0; i < iter; i++) {
            tot += voronoi(p + t) * amp; // Add the layer to the total.
            p *= 2.; // Position multiplied by two.
            t *= 1.5; // Time multiplied by less than two.
            sum += amp; // Sum of amplitudes.
            amp *= .5; // Decrease successive layer amplitude, as normal.
        }
        
        return tot/sum; // Range: [0, 1].
    }
    float distanceTo(vec2 src, vec2 dst) {
        float dx = src.x - dst.x;
        float dy = src.y - dst.y;
        float dv = dx * dx + dy * dy;
        return sqrt(dv);
    }
    
    
    void main() { 
        float len = distanceTo(vec2(0.5, 0.5), vec2(vUv.x, vUv.y)) * 2.0;  
        vec2 uv = (vUv-0.5) * 2.0;
        
        uv += vec2(sin(iTime*.5)*.25, cos(iTime*.5)*.125);
        
        vec3 rd = normalize(vec3(uv.x, uv.y, 3.1415926535898/8.));
    
        float cs = cos(iTime*.25), si = sin(iTime*.25); 
        rd.xy = rd.xy*mat2(cs, -si, si, cs);  
        float c = noiseLayers(rd*2.);
        
        c = max(c + dot(hash33(rd)*2. - 1., vec3(.015)), 0.);
    
        c *= sqrt(c)*1.5; // Contrast.
        vec3 col = firePalette(c); // Palettization.
        col = mix(col, col.zyx*.15 + c*.85, min(pow(dot(rd.xy, rd.xy)*1.2, 1.5), 1.)); // Color dispersion.
        col = pow(col, vec3(1.25)); // Tweaking the contrast a little.
    
        gl_FragColor = vec4(sqrt(clamp(col, 0., 1.)),  1.0 - pow(len, 2.0));
        
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

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/fireShader.js)

## 小结

- 本文提供 **火焰** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

