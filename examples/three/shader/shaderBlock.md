---
title: "Three.js 方块着色器教程"
description: "详解 Three.js 方块着色器：基于 WebGL 实现「方块着色器」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,方块着色器,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 方块着色器 · *Shader Block* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=shaderBlock)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![方块着色器](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/shaderBlock.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **方块着色器** 效果：基于 WebGL 实现「方块着色器」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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

const uniforms = {
    iResolution: {
        type: 'v2',
        value: new THREE.Vector2(box.clientWidth, box.clientHeight)
    },
    iTime: {
        type: 'f',
        value: 1.0
    }
}
animate()
function animate() {
    uniforms.iTime.value += 0.01
    requestAnimationFrame(animate)
    renderer.render(scene, camera)
}


let geometry = new THREE.PlaneGeometry(2, 2);

let material = new THREE.ShaderMaterial({
    uniforms: uniforms,
    vertexShader: `
    void main() {
        gl_Position = vec4( position, 1.0 );
    }
    `,
    fragmentShader: `
    // 屏幕尺寸
    
    precision highp float;
    uniform vec2 iResolution;
    uniform float iTime;



    float gTime = 0.;
    const float REPEAT = 5.0;

    // 回転行列
    mat2 rot(float a) {
        float c = cos(a), s = sin(a);
        return mat2(c,s,-s,c);
    }

    float sdBox( vec3 p, vec3 b )
    {
        vec3 q = abs(p) - b;
        return length(max(q,0.0)) + min(max(q.x,max(q.y,q.z)),0.0);
    }

    float box(vec3 pos, float scale) {
        pos *= scale;
        float base = sdBox(pos, vec3(.4,.4,.1)) /1.5;
        pos.xy *= 5.;
        pos.y -= 3.5;
        pos.xy *= rot(.75);
        float result = -base;
        return result;
    }

    float box_set(vec3 pos, float iTime) {
        vec3 pos_origin = pos;
        pos = pos_origin;
        pos .y += sin(gTime * 0.4) * 2.5;
        pos.xy *=   rot(.8);
        float box1 = box(pos,2. - abs(sin(gTime * 0.4)) * 1.5);
        pos = pos_origin;
        pos .y -=sin(gTime * 0.4) * 2.5;
        pos.xy *=   rot(.8);
        float box2 = box(pos,2. - abs(sin(gTime * 0.4)) * 1.5);
        pos = pos_origin;
        pos .x +=sin(gTime * 0.4) * 2.5;
        pos.xy *=   rot(.8);
        float box3 = box(pos,2. - abs(sin(gTime * 0.4)) * 1.5);	
        pos = pos_origin;
        pos .x -=sin(gTime * 0.4) * 2.5;
        pos.xy *=   rot(.8);
        float box4 = box(pos,2. - abs(sin(gTime * 0.4)) * 1.5);	
        pos = pos_origin;
        pos.xy *=   rot(.8);
        float box5 = box(pos,.5) * 6.;	
        pos = pos_origin;
        float box6 = box(pos,.5) * 6.;	
        float result = max(max(max(max(max(box1,box2),box3),box4),box5),box6);
        return result;
    }

    float map(vec3 pos, float iTime) {
        vec3 pos_origin = pos;
        float box_set1 = box_set(pos, iTime);

        return box_set1;
    }
    
    void main() {
        vec2 p = (gl_FragCoord.xy * 1. - iResolution.xy) / min(iResolution.x, iResolution.y);
        vec3 ro = vec3(0., -0.2 ,iTime * 4.);
        vec3 ray = normalize(vec3(p, 1.5));
        ray.xy = ray.xy * rot(sin(iTime * .03) * 5.);
        ray.yz = ray.yz * rot(sin(iTime * .05) * .2);
        float t = 0.1;
        vec3 col = vec3(0.);
        float ac = 0.0;


        for (int i = 0; i < 99; i++){
            vec3 pos = ro + ray * t;
            pos = mod(pos-2., 4.) -2.;
            gTime = iTime -float(i) * 0.01;
            
            float d = map(pos, iTime);

            d = max(abs(d), 0.01);
            ac += exp(-d*23.);

            t += d* 0.55;
        }

        col = vec3(ac * 0.02);

        col +=vec3(0.,0.2 * abs(sin(iTime)),0.5 + sin(iTime) * 0.2);


        gl_FragColor = vec4(col ,1.0 - t * (0.02 + 0.02 * sin (iTime)));
    }
    `
});

let mesh = new THREE.Mesh(geometry, material);
scene.add(mesh);
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/shaderBlock.js)

## 小结

- 本文提供 **方块着色器** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

