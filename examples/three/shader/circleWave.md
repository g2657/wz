---
title: "Three.js 圆波扫光教程"
description: "详解 Three.js 圆波扫光：基于 WebGL 实现「圆波扫光」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,圆波扫光,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 圆波扫光 · *Circle Wave Scan* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=circleWave)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![圆波扫光](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/circleWave.jpg)

## 你将学到什么

- **circleWave** 函数：同心圆径向扩散
- `fract(dist - t)` 形成 moving ring
- **AdditiveBlending** 叠加发光
- 扫描纹理 `wave.png` 调制遮罩

## 效果说明

水平放置的平面上，青色同心圆波从中心向外扩散，带 noise 纹理细节；dat.GUI 可调主色与暗部色。

## 核心概念

### circleWave

```glsl
float circleWave(vec3 p, vec3 origin, float distRatio) {
    float t = uTime;
    float dist = distance(p, origin) * distRatio;
    float radialMove = fract(dist - t);
    float fadeOutMask = 1.0 - smoothstep(1.0, 3.0, dist);
    radialMove *= fadeOutMask;
    float cutInitialMask = 1.0 - step(t, dist);
    return radialMove * cutInitialMask;
}
```

- `fract(dist - t)`：波环随时间外移
- `cutInitialMask`：未到达半径前不显示
- `fadeOutMask`：远处衰减

### 双层波 + 纹理

```glsl
float cw  = circleWave(worldPos, uScanOrigin, 3.2);
float cw2 = circleWave(worldPos, uScanOrigin, 2.8);
float scanMask = texture2D(uScanTex, uv).r;
vec3 scanCol = mix(uScanColorDark, uScanColor, mask1);
gl_FragColor = vec4(col, length(col)); // alpha 随亮度
```

材质：`transparent: true`，`blending: THREE.AdditiveBlending`。

## 实现步骤

1. PlaneGeometry + ShaderMaterial
2. mesh.rotation.x = PI/2 水平铺地
3. GUI 绑定 uScanColor / uScanColorDark
4. uTime 每帧 += 0.005

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GUI } from 'dat.gui'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(50, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 1, 0)

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

// 创建平面几何体
const geometry = new THREE.PlaneGeometry(2, 2);

const texture = new THREE.TextureLoader().load(FILE_HOST + 'images/channels/wave.png') 
texture.wrapS = THREE.RepeatWrapping
texture.wrapT = THREE.RepeatWrapping

// 创建材质
const material = new THREE.ShaderMaterial({
    side: THREE.DoubleSide,
    transparent: true,
    blending: THREE.AdditiveBlending, // 添加混合模式让效果更亮
    uniforms: {
        uTime: { value: 0.0 },
        uScanTex: { value:texture },
        uScanColor: { value: new THREE.Color(0x00ffff) },    // 主要扫描颜色
        uScanColorDark: { value: new THREE.Color(0x0088ff) } // 暗部扫描颜色
    },
    vertexShader: `
        varying vec2 vUv;
        varying vec3 vPosition;
        void main() {
            vUv = uv;
            vPosition = position;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
    `,
    fragmentShader: `
        #define uScanOrigin vec3(0.0, 0.0, 0.0)
        #define uScanWaveRatio1 3.2
        #define uScanWaveRatio2 2.8

        uniform float uTime;
        uniform sampler2D uScanTex;
        uniform vec3 uScanColor;
        uniform vec3 uScanColorDark;
        
        varying vec2 vUv;
        varying vec3 vPosition;

        float circleWave(vec3 p, vec3 origin, float distRatio) {
            float t = uTime;
            float dist = distance(p, origin) * distRatio;
            float radialMove = fract(dist - t);
            float fadeOutMask = 1.0 - smoothstep(1.0, 3.0, dist);
            radialMove *= fadeOutMask;
            float cutInitialMask = 1.0 - step(t, dist);
            return radialMove * cutInitialMask;
        }

        vec3 getScanColor(vec3 worldPos, vec2 uv, vec3 col) {
            // 纹理采样
            float scanMask = texture2D(uScanTex, uv).r;
            
            // 波浪效果
            float cw = circleWave(worldPos, uScanOrigin, uScanWaveRatio1);
            float cw2 = circleWave(worldPos, uScanOrigin, uScanWaveRatio2);
            
            // 扫描遮罩
            float mask1 = smoothstep(0.3, 0.0, 1.0 - cw);
            mask1 *= (1.0 + scanMask * 0.7);
            
            float mask2 = smoothstep(0.07, 0.0, 1.0 - cw2) * 0.8;
            mask1 += mask2;
            
            float mask3 = smoothstep(0.09, 0.0, 1.0 - cw) * 1.5;
            mask1 += mask3;

            // 颜色混合
            vec3 scanCol = mix(uScanColorDark, uScanColor, mask1);
            return scanCol * mask1; // 只返回扫描区域的颜色
        }

        void main() {
            vec3 col = vec3(0.0);
            col = getScanColor(vPosition, vUv * 10.0, col);
            
            // 计算alpha通道
            float alpha = length(col);  // 根据颜色强度计算透明度
            
            gl_FragColor = vec4(col, alpha);
        }
    `
});

// 创建网格并添加到场景
const mesh = new THREE.Mesh(geometry, material);
mesh.rotation.x = Math.PI / 2
scene.add(mesh);

// 创建dat.GUI
const gui = new GUI()
const params = {
  uScanColor: '#00ffff',
  uScanColorDark: '#0088ff'
}

gui.addColor(params, 'uScanColor').onChange((value) => {
  material.uniforms.uScanColor.value.set(value)
})

gui.addColor(params, 'uScanColorDark').onChange((value) => {
  material.uniforms.uScanColorDark.value.set(value)
})

animate()

function animate() {

  requestAnimationFrame(animate)

  controls.update()

  renderer.render(scene, camera)

  material.uniforms.uTime.value += 0.005;

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/circleWave.js)

## 小结

- 本文提供 **圆波扫光** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

