---
title: "Three.js 蜡烛教程"
description: "详解 Three.js 蜡烛：基于 WebGL 实现「蜡烛」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls、CatmullRomCurve3 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,蜡烛,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制,样条曲线"
outline: deep
---

### 蜡烛 · *Candle Shader* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=candleShader)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![蜡烛](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/candleShader.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- CatmullRomCurve3 样条曲线路径
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **蜡烛** 效果：基于 WebGL 实现「蜡烛」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls、CatmullRomCurve3。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- 曲线类 `getPoints(n)` 将贝塞尔/样条离散为路径点，再写入 BufferGeometry 驱动飞线或路径动画。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
3. 用曲线离散点构建 BufferGeometry，写入自定义 attribute 驱动动画
4. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
5. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from "three"
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js"

var scene = new THREE.Scene();
var camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 1, 1000);
camera.position.set(3, 5, 8).setLength(15);
var renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setClearColor(0x101005);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);
window.addEventListener("resize", event => {
    camera.aspect = innerWidth / innerHeight;
    camera.updateProjectionMatrix()
    renderer.setSize(innerWidth, innerHeight);
})

var controls = new OrbitControls(camera, renderer.domElement);
controls.enablePan = false;
controls.minPolarAngle = THREE.MathUtils.degToRad(60);
controls.maxPolarAngle = THREE.MathUtils.degToRad(95);
controls.minDistance = 4;
controls.maxDistance = 20;
controls.autoRotate = true;
controls.autoRotateSpeed = 1;
controls.target.set(0, 2, 0);
controls.update();

var light = new THREE.DirectionalLight(0xffffff, 0.025);
light.position.setScalar(10);
scene.add(light);
scene.add(new THREE.AmbientLight(0xffffff, 0.0625));

var casePath = new THREE.Path();
casePath.moveTo(0, 0);
casePath.lineTo(0, 0);
casePath.absarc(1.5, 0.5, 0.5, Math.PI * 1.5, Math.PI * 2);
casePath.lineTo(2, 1.5);
casePath.lineTo(1.99, 1.5);
casePath.lineTo(1.9, 0.5);
var caseGeo = new THREE.LatheGeometry(casePath.getPoints(), 64);
var caseMat = new THREE.MeshStandardMaterial({ color: "silver" });
var caseMesh = new THREE.Mesh(caseGeo, caseMat);
caseMesh.castShadow = true;

var paraffinPath = new THREE.Path();
paraffinPath.moveTo(0, -.25);
paraffinPath.lineTo(0, -.25);
paraffinPath.absarc(1, 0, 0.25, Math.PI * 1.5, Math.PI * 2);
paraffinPath.lineTo(1.25, 0);
paraffinPath.absarc(1.89, 0.1, 0.1, Math.PI * 1.5, Math.PI * 2);
var paraffinGeo = new THREE.LatheGeometry(paraffinPath.getPoints(), 64);
paraffinGeo.translate(0, 1.25, 0);
var paraffinMat = new THREE.MeshStandardMaterial({ color: 0xffff99, side: THREE.BackSide, metalness: 0, roughness: 0.75 });
var paraffinMesh = new THREE.Mesh(paraffinGeo, paraffinMat);
caseMesh.add(paraffinMesh);

var candlewickProfile = new THREE.Shape();
candlewickProfile.absarc(0, 0, 0.0625, 0, Math.PI * 2);

var candlewickCurve = new THREE.CatmullRomCurve3([
    new THREE.Vector3(0, 0, 0),
    new THREE.Vector3(0, 0.5, -0.0625),
    new THREE.Vector3(0.25, 0.5, 0.125)
]);

var candlewickGeo = new THREE.ExtrudeGeometry(candlewickProfile, {
    steps: 8,
    bevelEnabled: false,
    extrudePath: candlewickCurve
});

var colors = [];
var color1 = new THREE.Color("black");
var color2 = new THREE.Color(0x994411);
var color3 = new THREE.Color(0xffff44);
for (let i = 0; i < candlewickGeo.attributes.position.count; i++) {
    if (candlewickGeo.attributes.position.getY(i) < 0.4) {
        color1.toArray(colors, i * 3);
    }
    else {
        color2.toArray(colors, i * 3);
    };
    if (candlewickGeo.attributes.position.getY(i) < 0.15) color3.toArray(colors, i * 3);
}
candlewickGeo.setAttribute('color', new THREE.BufferAttribute(new Float32Array(colors), 3));
candlewickGeo.translate(0, 0.95, 0);
var candlewickMat = new THREE.MeshBasicMaterial({ vertexColors: true });

var candlewickMesh = new THREE.Mesh(candlewickGeo, candlewickMat);
caseMesh.add(candlewickMesh);

// candle light
var candleLight = new THREE.PointLight(0xffaa33, 1, 5, 2);
candleLight.position.set(0, 3, 0);
candleLight.castShadow = true;
caseMesh.add(candleLight);
var candleLight2 = new THREE.PointLight(0xffaa33, 1, 10, 2);
candleLight2.position.set(0, 4, 0);
candleLight2.castShadow = true;
caseMesh.add(candleLight2);

var flameMaterials = [];
function flame(isFrontSide) {
    let flameGeo = new THREE.SphereGeometry(0.5, 32, 32);
    flameGeo.translate(0, 0.5, 0);
    let flameMat = getFlameMaterial(true);
    flameMaterials.push(flameMat);
    let flame = new THREE.Mesh(flameGeo, flameMat);
    flame.position.set(0.06, 1.2, 0.06);
    flame.rotation.y = THREE.MathUtils.degToRad(-45);
    caseMesh.add(flame);
}

flame(false);
flame(true);

// table
var tableGeo = new THREE.CylinderGeometry(14, 14, 0.5, 64);
tableGeo.translate(0, -0.25, 0);
var tableMat = new THREE.MeshStandardMaterial({ map: new THREE.TextureLoader().load("https://threejs.org/examples/textures/hardwood2_diffuse.jpg"), metalness: 0, roughness: 0.75 });
var tableMesh = new THREE.Mesh(tableGeo, tableMat);
tableMesh.receiveShadow = true;

tableMesh.add(caseMesh);
scene.add(tableMesh);


var clock = new THREE.Clock();
var time = 0;

render();
function render() {
    requestAnimationFrame(render);
    time += clock.getDelta();
    flameMaterials[0].uniforms.time.value = time;
    flameMaterials[1].uniforms.time.value = time;
    candleLight2.position.x = Math.sin(time * Math.PI) * 0.25;
    candleLight2.position.z = Math.cos(time * Math.PI * 0.75) * 0.25;
    candleLight2.intensity = 2 + Math.sin(time * Math.PI * 2) * Math.cos(time * Math.PI * 1.5) * 0.25;
    controls.update();
    renderer.render(scene, camera);
}

//reference Paul https://codepen.io/prisoner849/pen/XPVGLp
function getFlameMaterial(isFrontSide) {
    let side = isFrontSide ? THREE.FrontSide : THREE.BackSide;
    return new THREE.ShaderMaterial({
        uniforms: {
            time: { value: 0 }
        },
        vertexShader: `
        uniform float time;
        varying vec2 vUv;
        varying float hValue;
        float random (in vec2 st) {
            return fract(sin(dot(st.xy,
                                 vec2(12.9898,78.233)))
                         * 43758.5453123);
        }

        float noise (in vec2 st) {
            vec2 i = floor(st);
            vec2 f = fract(st);

            // Four corners in 2D of a tile
            float a = random(i);
            float b = random(i + vec2(1.0, 0.0));
            float c = random(i + vec2(0.0, 1.0));
            float d = random(i + vec2(1.0, 1.0));

            // Smooth Interpolation

            // Cubic Hermine Curve.  Same as SmoothStep()
            vec2 u = f*f*(3.0-2.0*f);
            // u = smoothstep(0.,1.,f);

            // Mix 4 coorners percentages
            return mix(a, b, u.x) +
                    (c - a)* u.y * (1.0 - u.x) +
                    (d - b) * u.x * u.y;
        }

        void main() {
          vUv = uv;
          vec3 pos = position;

          pos *= vec3(0.8, 2, 0.725);
          hValue = position.y;
          //float sinT = sin(time * 2.) * 0.5 + 0.5;
          float posXZlen = length(position.xz);

          pos.y *= 1. + (cos((posXZlen + 0.25) * 3.1415926) * 0.25 + noise(vec2(0, time)) * 0.125 + noise(vec2(position.x + time, position.z + time)) * 0.5) * position.y; // flame height

          pos.x += noise(vec2(time * 2., (position.y - time) * 4.0)) * hValue * 0.0312; // flame trembling
          pos.z += noise(vec2((position.y - time) * 4.0, time * 2.)) * hValue * 0.0312; // flame trembling

          gl_Position = projectionMatrix * modelViewMatrix * vec4(pos,1.0);
        }
      `,
        fragmentShader: `
        varying float hValue;
        varying vec2 vUv;

        // honestly stolen from https://www.shadertoy.com/view/4dsSzr
        vec3 heatmapGradient(float t) {
          return clamp((pow(t, 1.5) * 0.8 + 0.2) * vec3(smoothstep(0.0, 0.35, t) + t * 0.5, smoothstep(0.5, 1.0, t), max(1.0 - t * 1.7, t * 7.0 - 6.0)), 0.0, 1.0);
        }

        void main() {
          float v = abs(smoothstep(0.0, 0.4, hValue) - 1.);
          float alpha = (1. - v) * 0.99; // bottom transparency
          alpha -= 1. - smoothstep(1.0, 0.97, hValue); // tip transparency
          gl_FragColor = vec4(heatmapGradient(smoothstep(0.0, 0.3, hValue)) * vec3(0.95,0.95,0.4), alpha) ;
          gl_FragColor.rgb = mix(vec3(0,0,1), gl_FragColor.rgb, smoothstep(0.0, 0.3, hValue)); // blueish for bottom
          gl_FragColor.rgb += vec3(1, 0.9, 0.5) * (1.25 - vUv.y); // make the midst brighter
          gl_FragColor.rgb = mix(gl_FragColor.rgb, vec3(0.66, 0.32, 0.03), smoothstep(0.95, 1., hValue)); // tip
        }
      `,
        transparent: true,
        side: side
    })

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/candleShader.js)

## 小结

- 本文提供 **蜡烛** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

