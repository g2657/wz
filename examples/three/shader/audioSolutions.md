---
title: "Three.js 音乐可视化教程"
description: "详解 Three.js 音乐可视化：Lights，涵盖 onBeforeCompile、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,音乐可视化,WebGL,源码,教程,在线案例,onBeforeCompile,shader注入,OrbitControls,相机控制"
outline: deep
---

### 音乐可视化 · *Audio Visualization* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=audioSolutions)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![音乐可视化](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/audioSolutions.jpg)

## 你将学到什么

- **THREE.AudioAnalyser** 读取 FFT 频谱
- 频谱均值驱动 **uStrength** uniform
- **Simplex 4D Noise** 沿法线顶点位移
- `customDepthMaterial` 同步修改保证阴影正确

## 效果说明

点击 Play 播放 MP3，高精度 **IcosahedronGeometry** 表面随节拍起伏，片元输出法线色形成「脉动球体」。

## 核心概念

### 音频链路

```js
const listener = new THREE.AudioListener();
const audio = new THREE.Audio(listener);
const mediaElement = new Audio(url);
mediaElement.play();
audio.setMediaElementSource(mediaElement);
analyser = new THREE.AudioAnalyser(audio, 4096);
```

### 频谱 → 强度

```js
analyser.getFrequencyData();
let sum = 0;
for (let i = 0; i < analyser.data.length; i++) sum += analyser.data[i];
uniform.uStrength.value = sum / (analyser.data.length * 25.5);
```

### 顶点噪声位移

在 `onBeforeCompile` 的 `#include <fog_vertex>` 后：

```glsl
newPos += normal * simplexNoise4d(vec4(position, uTime)) * uStrength;
gl_Position = projectionMatrix * modelViewMatrix * vec4(newPos, 1.0);
```

**depthMaterial** 同样注入，否则 shadow pass 与 color pass 几何不一致。

片元改为 `gl_FragColor = vec4(vNormal, 1.)`，用法线 RGB 当可视化色。

## 实现步骤

1. init 场景：高细分 Icosahedron + 地面 + 灯光阴影
2. MeshStandardMaterial.onBeforeCompile 注入 GLSL
3. createButton 绑定 Play/Pause
4. tick 里 updateOffsetData + uTime + render

## 代码要点

```js
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";

let mediaElement;
let analyser;
let scene;
let camera;
let renderer;
let controls;
let mesh;

const fftSize = 4096;
const clock = new THREE.Clock();
const uniform = {
  uTime: { value: 0 },
  tAudioData: { value: 0 },
  uStrength: { value: 0 },
};
const box = document.getElementById("box");

const init = () => {
  // Scene
  scene = new THREE.Scene();

  const material = new THREE.MeshStandardMaterial();
  material.roughness = 0.7;
  const depthMaterial = new THREE.MeshDepthMaterial({
    depthPacking: THREE.RGBADepthPacking,
  });

  material.onBeforeCompile = (shader) => {
    shader.uniforms.uTime = uniform.uTime;
    shader.uniforms.uStrength = uniform.uStrength;

    shader.vertexShader = shader.vertexShader.replace(
      "#include <common>",
      `
              #include <common>
              attribute float aOffset;
              varying vec2 vUv;
              uniform float uTime;
              uniform float uStrength;

              //	Simplex 4D Noise
  //	by Ian McEwan, Ashima Arts
  //
  vec4 permute(vec4 x){return mod(((x*34.0)+1.0)*x, 289.0);}
  float permute(float x){return floor(mod(((x*34.0)+1.0)*x, 289.0));}
  vec4 taylorInvSqrt(vec4 r){return 1.79284291400159 - 0.85373472095314 * r;}
  float taylorInvSqrt(float r){return 1.79284291400159 - 0.85373472095314 * r;}

  vec4 grad4(float j, vec4 ip)
  {
    const vec4 ones = vec4(1.0, 1.0, 1.0, -1.0);
    vec4 p,s;

    p.xyz = floor( fract (vec3(j) * ip.xyz) * 7.0) * ip.z - 1.0;
    p.w = 1.5 - dot(abs(p.xyz), ones.xyz);
    s = vec4(lessThan(p, vec4(0.0)));
    p.xyz = p.xyz + (s.xyz*2.0 - 1.0) * s.www;

    return p;
  }

  float simplexNoise4d(vec4 v)
  {
    const vec2  C = vec2( 0.138196601125010504,  // (5 - sqrt(5))/20  G4
                          0.309016994374947451); // (sqrt(5) - 1)/4   F4
    // First corner
    vec4 i  = floor(v + dot(v, C.yyyy) );
    vec4 x0 = v -   i + dot(i, C.xxxx);

    // Other corners

    // Rank sorting originally contributed by Bill Licea-Kane, AMD (formerly ATI)
    vec4 i0;

    vec3 isX = step( x0.yzw, x0.xxx );
    vec3 isYZ = step( x0.zww, x0.yyz );
    //  i0.x = dot( isX, vec3( 1.0 ) );
    i0.x = isX.x + isX.y + isX.z;
    i0.yzw = 1.0 - isX;

    //  i0.y += dot( isYZ.xy, vec2( 1.0 ) );
    i0.y += isYZ.x + isYZ.y;
    i0.zw += 1.0 - isYZ.xy;

    i0.z += isYZ.z;
    i0.w += 1.0 - isYZ.z;

    // i0 now contains the unique values 0,1,2,3 in each channel
    vec4 i3 = clamp( i0, 0.0, 1.0 );
    vec4 i2 = clamp( i0-1.0, 0.0, 1.0 );
    vec4 i1 = clamp( i0-2.0, 0.0, 1.0 );

    //  x0 = x0 - 0.0 + 0.0 * C
    vec4 x1 = x0 - i1 + 1.0 * C.xxxx;
    vec4 x2 = x0 - i2 + 2.0 * C.xxxx;
    vec4 x3 = x0 - i3 + 3.0 * C.xxxx;
    vec4 x4 = x0 - 1.0 + 4.0 * C.xxxx;

    // Permutations
    i = mod(i, 289.0);
    float j0 = permute( permute( permute( permute(i.w) + i.z) + i.y) + i.x);
    vec4 j1 = permute( permute( permute( permute (
               i.w + vec4(i1.w, i2.w, i3.w, 1.0 ))
             + i.z + vec4(i1.z, i2.z, i3.z, 1.0 ))
             + i.y + vec4(i1.y, i2.y, i3.y, 1.0 ))
             + i.x + vec4(i1.x, i2.x, i3.x, 1.0 ));
    // Gradients
    // ( 7*7*6 points uniformly over a cube, mapped onto a 4-octahedron.)
    // 7*7*6 = 294, which is close to the ring size 17*17 = 289.

    vec4 ip = vec4(1.0/294.0, 1.0/49.0, 1.0/7.0, 0.0) ;

    vec4 p0 = grad4(j0,   ip);
    vec4 p1 = grad4(j1.x, ip);
    vec4 p2 = grad4(j1.y, ip);
    vec4 p3 = grad4(j1.z, ip);
    vec4 p4 = grad4(j1.w, ip);

    // Normalise gradients
    vec4 norm = taylorInvSqrt(vec4(dot(p0,p0), dot(p1,p1), dot(p2, p2), dot(p3,p3)));
    p0 *= norm.x;
    p1 *= norm.y;
    p2 *= norm.z;
    p3 *= norm.w;
    p4 *= taylorInvSqrt(dot(p4,p4));

    // Mix contributions from the five corners
    vec3 m0 = max(0.6 - vec3(dot(x0,x0), dot(x1,x1), dot(x2,x2)), 0.0);
    vec2 m1 = max(0.6 - vec2(dot(x3,x3), dot(x4,x4)            ), 0.0);
    m0 = m0 * m0;
    m1 = m1 * m1;
    return 49.0 * ( dot(m0*m0, vec3( dot( p0, x0 ), dot( p1, x1 ), dot( p2, x2 )))
                 + dot(m1*m1, vec2( dot( p3, x3 ), dot( p4, x4 ) ) ) ) ;

  }

          `
    );

    shader.vertexShader = shader.vertexShader.replace(
      "#include <fog_vertex>",
      `#include <fog_vertex>
            vec3 newPos = position;
            float strength = uStrength;
            newPos += normal * simplexNoise4d(vec4(position, uTime)) * strength;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(newPos, 1.0);

          `
    );
    shader.fragmentShader = shader.fragmentShader.replace(
      "#include <common>",
      `#include <common>
          uniform float uTime;
          uniform float uStrength;
        `
    );

    shader.fragmentShader = shader.fragmentShader.replace(
      "#include <opaque_fragment>",
      `#include <opaque_fragment>
          gl_FragColor = vec4(vNormal, 1.);
        `
    );
  };
  depthMaterial.onBeforeCompile = (shader) => {
    shader.uniforms.uTime = uniform.uTime;
    shader.uniforms.uStrength = uniform.uStrength;

    shader.vertexShader = shader.vertexShader.replace(
      "#include <common>",
      `
              #include <common>
              attribute float aOffset;
              varying vec2 vUv;
              uniform float uTime;
              uniform float uStrength;

              //	Simplex 4D Noise
  //	by Ian McEwan, Ashima Arts
  //
  vec4 permute(vec4 x){return mod(((x*34.0)+1.0)*x, 289.0);}
  float permute(float x){return floor(mod(((x*34.0)+1.0)*x, 289.0));}
  vec4 taylorInvSqrt(vec4 r){return 1.79284291400159 - 0.85373472095314 * r;}
  float taylorInvSqrt(float r){return 1.79284291400159 - 0.85373472095314 * r;}

  vec4 grad4(float j, vec4 ip)
  {
    const vec4 ones = vec4(1.0, 1.0, 1.0, -1.0);
    vec4 p,s;

    p.xyz = floor( fract (vec3(j) * ip.xyz) * 7.0) * ip.z - 1.0;
    p.w = 1.5 - dot(abs(p.xyz), ones.xyz);
    s = vec4(lessThan(p, vec4(0.0)));
    p.xyz = p.xyz + (s.xyz*2.0 - 1.0) * s.www;

    return p;
  }

  float simplexNoise4d(vec4 v)
  {
    const vec2  C = vec2( 0.138196601125010504,  // (5 - sqrt(5))/20  G4
                          0.309016994374947451); // (sqrt(5) - 1)/4   F4
    // First corner
    vec4 i  = floor(v + dot(v, C.yyyy) );
    vec4 x0 = v -   i + dot(i, C.xxxx);

    // Other corners

    // Rank sorting originally contributed by Bill Licea-Kane, AMD (formerly ATI)
    vec4 i0;

    vec3 isX = step( x0.yzw, x0.xxx );
    vec3 isYZ = step( x0.zww, x0.yyz );
    //  i0.x = dot( isX, vec3( 1.0 ) );
    i0.x = isX.x + isX.y + isX.z;
    i0.yzw = 1.0 - isX;

    //  i0.y += dot( isYZ.xy, vec2( 1.0 ) );
    i0.y += isYZ.x + isYZ.y;
    i0.zw += 1.0 - isYZ.xy;

    i0.z += isYZ.z;
    i0.w += 1.0 - isYZ.z;

    // i0 now contains the unique values 0,1,2,3 in each channel
    vec4 i3 = clamp( i0, 0.0, 1.0 );
    vec4 i2 = clamp( i0-1.0, 0.0, 1.0 );
    vec4 i1 = clamp( i0-2.0, 0.0, 1.0 );

    //  x0 = x0 - 0.0 + 0.0 * C
    vec4 x1 = x0 - i1 + 1.0 * C.xxxx;
    vec4 x2 = x0 - i2 + 2.0 * C.xxxx;
    vec4 x3 = x0 - i3 + 3.0 * C.xxxx;
    vec4 x4 = x0 - 1.0 + 4.0 * C.xxxx;

    // Permutations
    i = mod(i, 289.0);
    float j0 = permute( permute( permute( permute(i.w) + i.z) + i.y) + i.x);
    vec4 j1 = permute( permute( permute( permute (
               i.w + vec4(i1.w, i2.w, i3.w, 1.0 ))
             + i.z + vec4(i1.z, i2.z, i3.z, 1.0 ))
             + i.y + vec4(i1.y, i2.y, i3.y, 1.0 ))
             + i.x + vec4(i1.x, i2.x, i3.x, 1.0 ));
    // Gradients
    // ( 7*7*6 points uniformly over a cube, mapped onto a 4-octahedron.)
    // 7*7*6 = 294, which is close to the ring size 17*17 = 289.

    vec4 ip = vec4(1.0/294.0, 1.0/49.0, 1.0/7.0, 0.0) ;

    vec4 p0 = grad4(j0,   ip);
    vec4 p1 = grad4(j1.x, ip);
    vec4 p2 = grad4(j1.y, ip);
    vec4 p3 = grad4(j1.z, ip);
    vec4 p4 = grad4(j1.w, ip);

    // Normalise gradients
    vec4 norm = taylorInvSqrt(vec4(dot(p0,p0), dot(p1,p1), dot(p2, p2), dot(p3,p3)));
    p0 *= norm.x;
    p1 *= norm.y;
    p2 *= norm.z;
    p3 *= norm.w;
    p4 *= taylorInvSqrt(dot(p4,p4));

    // Mix contributions from the five corners
    vec3 m0 = max(0.6 - vec3(dot(x0,x0), dot(x1,x1), dot(x2,x2)), 0.0);
    vec2 m1 = max(0.6 - vec2(dot(x3,x3), dot(x4,x4)            ), 0.0);
    m0 = m0 * m0;
    m1 = m1 * m1;
    return 49.0 * ( dot(m0*m0, vec3( dot( p0, x0 ), dot( p1, x1 ), dot( p2, x2 )))
                 + dot(m1*m1, vec2( dot( p3, x3 ), dot( p4, x4 ) ) ) ) ;

  }

          `
    );

    shader.vertexShader = shader.vertexShader.replace(
      "#include <clipping_planes_vertex>",
      `#include <clipping_planes_vertex>
            vec3 newPos = position;
            float strength = uStrength;
            newPos += normal * simplexNoise4d(vec4(position, uTime)) * strength;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(newPos, 1.0);
          `
    );
  };
  // const geometry = new THREE.SphereGeometry(0.5, 256, 256);
  const geometry = new THREE.IcosahedronGeometry(2.5, 50);
  mesh = new THREE.Mesh(geometry, material);
  mesh.customDepthMaterial = depthMaterial;
  mesh.castShadow = true;

  scene.add(mesh);
  const plane = new THREE.Mesh(
    new THREE.PlaneGeometry(25, 25),
    new THREE.MeshStandardMaterial()
  );
  plane.rotation.x = -Math.PI * 0.5;
  plane.position.y = -5;
  plane.receiveShadow = true;

  scene.add(plane);

  /**
   * Lights
   */
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.3);
  scene.add(ambientLight);

  const directionalLight = new THREE.DirectionalLight("#ffffff", 0.3);
  directionalLight.castShadow = true;
  directionalLight.shadow.mapSize.set(1024, 1024);
  directionalLight.shadow.camera.far = 40;
  directionalLight.castShadow = true;
  directionalLight.position.set(2, 2, -2);
  scene.add(directionalLight);

  const directionalLightCameraHelper = new THREE.CameraHelper(
    directionalLight.shadow.camera
  );
  // 关闭助手
  scene.add(directionalLightCameraHelper);
  /**
   * Sizes
   */
  const sizes = {
    width: box.clientWidth,
    height: box.clientHeight,
  };
  window.addEventListener("resize", () => {
    // Update sizes
    sizes.width = box.clientWidth;
    sizes.height = box.clientHeight;
    // Update camera
    camera.aspect = sizes.width / sizes.height;

    camera.updateProjectionMatrix();
    // Update renderer
    renderer.setSize(sizes.width, sizes.height);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  });

  /**
   * Camera
   */
  // Base camera
  camera = new THREE.PerspectiveCamera(
    75,
    sizes.width / sizes.height,
    0.1,
    100
  );
  camera.position.set(4, 4, 6);
  scene.add(camera);

  /**
   * Renderer
   */
  renderer = new THREE.WebGLRenderer();
  renderer.setSize(sizes.width, sizes.height);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.setClearColor("#111");
  renderer.shadowMap.enabled = true;
  box.appendChild(renderer.domElement);
  // Controls
  controls = new OrbitControls(camera, renderer.domElement);
  // controls.enableDamping = true;
};

const tick = () => {
  const elapsedTime = clock.getElapsedTime();
  controls?.update();

  // Update material

  updateOffsetData();

  if (uniform?.uTime) {
    uniform.uTime.value = elapsedTime;
  }

  // Render
  renderer.render(scene, camera);
  // Call tick again on the next frame
  window.requestAnimationFrame(tick);
};

const updateOffsetData = () => {
  if (analyser?.getFrequencyData) {
    analyser.getFrequencyData();
    const analyserData = analyser?.data;
    let sum = 0;
    for (let i = 0; i < analyserData.length; i++) {
      sum += analyserData[i];
    }
    sum /= analyserData.length * 25.5;
    uniform.uStrength.value = sum;
  }
};

const play = () => {
  const listener = new THREE.AudioListener();
  const audio = new THREE.Audio(listener);

  const file = FILE_HOST + "files/audio/Avicii-WeBurn.mp3";
  mediaElement = new Audio(file);
  mediaElement.crossOrigin = "crossOrigin";
  mediaElement.play();
  audio.setMediaElementSource(mediaElement);
  analyser = new THREE.AudioAnalyser(audio, fftSize);
};

const pause = () => {
  mediaElement.pause();
};

const createButton = () => {
  const playButton = document.createElement("button");
  playButton.textContent = "Play";
  playButton.style.position = "absolute";
  playButton.style.right = "140px";
  playButton.style.top = "30px";
  playButton.style.padding = "10px 20px";
  box.appendChild(playButton);
  playButton.addEventListener("click", play);

  const pauseButton = document.createElement("button");
  pauseButton.textContent = "Pause";
  pauseButton.style.position = "absolute";
  pauseButton.style.right = "30px";
  pauseButton.style.top = "30px";
  pauseButton.style.padding = "10px 20px";
  box.appendChild(pauseButton);
  pauseButton.addEventListener("click", pause);
};

init();
createButton();
tick();
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/audioSolutions.js)

## 小结

- 本文提供 **音乐可视化** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

