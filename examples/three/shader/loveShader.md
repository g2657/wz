---
title: "Three.js 爱心教程"
description: "详解 Three.js 爱心：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理，涵盖 ShaderMaterial、THREE.Points、Canvas 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,爱心,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,粒子特效,Points,Canvas纹理"
outline: deep
---

### 爱心 · *Love Shader* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=loveShader)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![爱心](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/loveShader.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- THREE.Points 粒子点渲染
- Canvas 动态纹理贴图
- GSAP 时间轴与补间动画
- BufferGeometry 自定义顶点/索引数据
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **爱心** 效果：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理；核心用到 ShaderMaterial、THREE.Points、Canvas。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **THREE.Points** 将每个顶点渲染为可控大小的粒子；可用自定义 attribute（如 `u_index`）驱动片元/顶点动画。
- **CanvasTexture** 每帧或按需把 2D Canvas 内容上传 GPU，适合动态文字、图表、视频帧贴图。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
3. 在定时器或 GSAP 时间轴中更新 uniform / 变换，驱动特效播放
4. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from "three";
import { gsap } from "gsap";

class LoveParticlesWorld {
  constructor() {
    // 基本参数设置 - 增大粒子数量和默认尺寸
    this.params = {
      heartParticles: 1800,
      starParticles: 700,
      colorCycle: 0
    };
    
    this.initScene();
    this.initCamera();
    this.initRenderer();
    this.createHeart();
    this.createStars();
    this.createGlow();
    this.addEvents();
    
    this.clock = new THREE.Clock();
    this.animate();
    this.addUIElements();
  }
  
  // 场景初始化
  initScene() {
    this.scene = new THREE.Scene();
    
    // 创建更丰富的渐变背景
    const bgColors = [new THREE.Color(0x1a0033), new THREE.Color(0x000022)];
    const bgTexture = this.createGradientTexture(bgColors);
    this.scene.background = bgTexture;
    
    // 增强光源
    this.scene.add(new THREE.AmbientLight(0x404040, 1.8));
    
    this.pointLight = new THREE.PointLight(0xff3388, 1.5, 100);
    this.pointLight.position.set(0, 0, 5);
    this.scene.add(this.pointLight);
  }
  
  createGradientTexture(colors) {
    const canvas = document.createElement('canvas');
    canvas.width = 2;
    canvas.height = 256;
    const context = canvas.getContext('2d');
    const gradient = context.createLinearGradient(0, 0, 0, 256);
    gradient.addColorStop(0, colors[0].getStyle());
    gradient.addColorStop(1, colors[1].getStyle());
    context.fillStyle = gradient;
    context.fillRect(0, 0, 2, 256);
    
    const texture = new THREE.CanvasTexture(canvas);
    texture.magFilter = THREE.LinearFilter;
    texture.minFilter = THREE.LinearFilter;
    return texture;
  }
  
  initCamera() {
    this.camera = new THREE.PerspectiveCamera(
      75, window.innerWidth / window.innerHeight, 0.1, 100
    );
    this.camera.position.z = 4.5;
    this.scene.add(this.camera);
  }
  
  initRenderer() {
    this.renderer = new THREE.WebGLRenderer({ antialias: true });
    this.renderer.setSize(window.innerWidth, window.innerHeight);
    this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    document.body.appendChild(this.renderer.domElement);
  }
  
  // 创建爱心粒子 - 增大粒子尺寸和发光强度
  createHeart() {
    const heartMaterial = new THREE.ShaderMaterial({
      fragmentShader: `
        varying vec3 vColor;
        varying vec2 vUv;
        varying float vIntensity;
        
        void main() {
          // 更强的发光效果
          float strength = 1.0 - 2.0 * distance(vUv, vec2(0.5));
          strength = pow(strength, 1.4); // 更柔和的边缘
          
          // 增强发光和颜色饱和度
          vec3 finalColor = vColor * (1.0 + vIntensity * 0.8);
          
          // 更亮的白色光晕
          finalColor += vec3(1.0) * pow(strength, 6.0) * 0.4;
          
          gl_FragColor = vec4(finalColor * strength, strength);
        }
      `,
      vertexShader: `
        #define PI 3.1415926535897932384626433832795
        uniform float uTime;
        uniform float uSize;
        
        attribute float aScale;
        attribute vec3 aColor;
        attribute float aSpeed;
        attribute float aOffset;
        attribute float aPhase;
        
        varying vec3 vColor;
        varying vec2 vUv;
        varying float vIntensity;
        
        vec3 heartShape(float t, float scale) {
          t = aPhase + sin(uTime * 0.2) * 0.05;
          
          float sign = 2.0 * (step(0.5, aOffset) - 0.5);
          t = sign * mod(-uTime * aSpeed * 0.004 + 9.0 * aSpeed * aSpeed, PI);
          
          float a = pow(t, 2.0) * pow((t - sign * PI), 2.0);
          float r = 0.16 * scale; // 增大基础大小
          
          // 优化的心形曲线
          float x = r * 16.0 * pow(sin(t), 2.0) * sin(t);
          float y = r * (13.0 * cos(t) - 5.0 * cos(2.0 * t) - 2.0 * cos(3.0 * t) - cos(4.0 * t));
          float z = 0.13 * a * (aOffset - 0.5) * sin(12.0 * sin(0.3 * uTime + aOffset * 3.0) * t);
          
          // 更丰富的浮动动画
          y += sin(uTime * 0.5) * 0.06;
          x += cos(uTime * 0.3) * 0.04;
          
          return vec3(x, y, z);
        }
        
        void main() {
          vec3 heartPosition = heartShape(0.0, 1.0);
          
          vec4 modelPosition = modelMatrix * vec4(heartPosition, 1.0);
          vec4 viewPosition = viewMatrix * modelPosition;
          
          // 更强的脉动效果
          float pulse = 0.6 + 0.2 * sin(uTime * (0.6 + aOffset * 0.5) + aPhase * 5.0);
          float sizeScale = aScale * uSize * pulse;
          
          viewPosition.xyz += position * sizeScale;
          
          gl_Position = projectionMatrix * viewPosition;
          
          vColor = aColor;
          vUv = uv;
          vIntensity = pulse + 0.3 * sin(uTime * aSpeed * 0.1);
        }
      `,
      uniforms: {
        uTime: { value: 0 },
        uSize: { value: 0.38 }  // 显著增大粒子默认尺寸
      },
      transparent: true,
      blending: THREE.AdditiveBlending,
      depthWrite: false
    });
    
    const count = this.params.heartParticles;
    const geometry = new THREE.PlaneGeometry(1, 1);
    const instancedGeometry = new THREE.InstancedBufferGeometry();
    
    Object.keys(geometry.attributes).forEach(attr => {
      instancedGeometry.attributes[attr] = geometry.attributes[attr];
    });
    instancedGeometry.index = geometry.index;
    
    // 创建实例化属性数组
    const scales = new Float32Array(count);
    const colors = new Float32Array(count * 3);
    const speeds = new Float32Array(count);
    const offsets = new Float32Array(count);
    const phases = new Float32Array(count);
    
    // 更鲜艳的配色方案
    this.colorSchemes = [
      // 鲜艳粉红色系
      [0xffffff, 0xff1177, 0xff66bb, 0xff44aa, 0xffddee],
      // 明亮蓝紫色系
      [0x99eeff, 0x66aaff, 0x4466ff, 0x9944ff, 0xeeddff],
      // 金橙黄色系
      [0xffdd22, 0xff9911, 0xff6633, 0xffdd99, 0xffbb55]
    ];
    
    const colorChoices = this.colorSchemes[this.params.colorCycle];
    
    for (let i = 0; i < count; i++) {
      // 增大粒子尺寸范围
      scales[i] = 0.15 + Math.random() * 0.45;
      speeds[i] = 0.5 + Math.random() * 4.0;
      offsets[i] = Math.random();
      phases[i] = Math.random() * Math.PI * 2;
      
      const color = new THREE.Color(colorChoices[Math.floor(Math.random() * colorChoices.length)]);
      colors[i * 3] = color.r;
      colors[i * 3 + 1] = color.g;
      colors[i * 3 + 2] = color.b;
    }
    
    instancedGeometry.setAttribute("aScale", new THREE.InstancedBufferAttribute(scales, 1));
    instancedGeometry.setAttribute("aColor", new THREE.InstancedBufferAttribute(colors, 3));
    instancedGeometry.setAttribute("aSpeed", new THREE.InstancedBufferAttribute(speeds, 1));
    instancedGeometry.setAttribute("aOffset", new THREE.InstancedBufferAttribute(offsets, 1));
    instancedGeometry.setAttribute("aPhase", new THREE.InstancedBufferAttribute(phases, 1));
    
    this.heart = new THREE.Mesh(instancedGeometry, heartMaterial);
    this.scene.add(this.heart);
  }
  
  // 简化的星星创建
  createStars() {
    const starsMaterial = new THREE.ShaderMaterial({
      fragmentShader: `
        varying vec3 vColor;
        varying float vSize;
        
        void main() {
          float distanceToCenter = length(gl_PointCoord - vec2(0.5));
          float strength = 1.0 - smoothstep(0.0, 0.5, distanceToCenter);
          vec3 finalColor = vColor * pow(strength, 1.5) * 1.5;
          gl_FragColor = vec4(finalColor, strength * 0.8);
        }
      `,
      vertexShader: `
        uniform float uTime;
        attribute float aSize;
        attribute vec3 aColor;
        attribute float aSpeed;
        
        varying vec3 vColor;
        varying float vSize;
        
        void main() {
          vec3 pos = position;
          
          // 闪烁效果
          float flicker = 0.8 + 0.5 * sin(uTime * aSpeed + position.x * 100.0);
          
          // 轻微漂浮
          pos.x += sin(uTime * 0.1 * aSpeed + position.y) * 0.15;
          pos.y += cos(uTime * 0.1 * aSpeed + position.x) * 0.15;
          
          vec4 mvPosition = modelViewMatrix * vec4(pos, 1.0);
          gl_Position = projectionMatrix * mvPosition;
          
          gl_PointSize = aSize * flicker * (350.0 / length(mvPosition.xyz));
          
          vColor = aColor;
          vSize = flicker;
        }
      `,
      uniforms: {
        uTime: { value: 0 }
      },
      transparent: true,
      blending: THREE.AdditiveBlending,
      depthWrite: false
    });
    
    const starsGeometry = new THREE.BufferGeometry();
    const starsCount = this.params.starParticles;
    const positions = new Float32Array(starsCount * 3);
    const colors = new Float32Array(starsCount * 3);
    const sizes = new Float32Array(starsCount);
    const speeds = new Float32Array(starsCount);
    
    const starColors = [
      new THREE.Color(0xffffff), new THREE.Color(0xaaccff),
      new THREE.Color(0xffccee), new THREE.Color(0xddddff)
    ];
    
    for (let i = 0; i < starsCount; i++) {
      const radius = 10 + Math.random() * 20;
      const theta = Math.random() * Math.PI * 2;
      const phi = Math.acos(2 * Math.random() - 1);
      
      positions[i * 3] = radius * Math.sin(phi) * Math.cos(theta);
      positions[i * 3 + 1] = radius * Math.sin(phi) * Math.sin(theta);
      positions[i * 3 + 2] = radius * Math.cos(phi) - 15;
      
      // 更大的星星
      sizes[i] = 4 + Math.random(); 
      speeds[i] = 0.1 + Math.random() * 2.0;
      
      const color = starColors[Math.floor(Math.random() * starColors.length)];
      colors[i * 3] = color.r;
      colors[i * 3 + 1] = color.g;
      colors[i * 3 + 2] = color.b;
    }
    
    starsGeometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    starsGeometry.setAttribute('aColor', new THREE.BufferAttribute(colors, 3));
    starsGeometry.setAttribute('aSize', new THREE.BufferAttribute(sizes, 1));
    starsGeometry.setAttribute('aSpeed', new THREE.BufferAttribute(speeds, 1));
    
    this.stars = new THREE.Points(starsGeometry, starsMaterial);
    this.scene.add(this.stars);
  }
  
  // 简化的光晕效果
  createGlow() {
    const glowGeometry = new THREE.SphereGeometry(1.8, 32, 32);
    const glowMaterial = new THREE.ShaderMaterial({
      fragmentShader: `
        varying float vIntensity;
        varying vec3 vColor;
        
        void main() {
          float alpha = pow(vIntensity, 2.5) * 0.55;
          gl_FragColor = vec4(vColor, alpha);
        }
      `,
      vertexShader: `
        uniform float uTime;
        varying float vIntensity;
        varying vec3 vColor;
        
        void main() {
          vec3 pos = position;
          float noise = sin(pos.x * 5.0 + uTime) * sin(pos.y * 5.0 + uTime) * sin(pos.z * 5.0 + uTime) * 0.15;
          pos += normal * noise;
          
          vec4 mvPosition = modelViewMatrix * vec4(pos, 1.0);
          gl_Position = projectionMatrix * mvPosition;
          
          vIntensity = 1.0 - length(pos) / 2.0;
          vColor = vec3(0.95, 0.4, 0.7);
        }
      `,
      uniforms: {
        uTime: { value: 0 }
      },
      transparent: true,
      blending: THREE.AdditiveBlending,
      side: THREE.BackSide,
      depthWrite: false
    });
    
    this.glow = new THREE.Mesh(glowGeometry, glowMaterial);
    this.scene.add(this.glow);
  }
  
  // 简化的事件处理
  addEvents() {
    window.addEventListener('mousemove', (e) => {
      const x = e.clientX / window.innerWidth - 0.5;
      const y = e.clientY / window.innerHeight - 0.5;
      
      gsap.to(this.camera.position, {
        x: x * 1.0, // 增大鼠标移动的影响
        y: -y * 1.0,
        duration: 1.0,
        ease: "power2.out"
      });
      
      gsap.to(this.pointLight.position, {
        x: x * 4,
        y: -y * 4,
        duration: 1.5,
        ease: "power1.out"
      });
    });
    
    window.addEventListener('click', () => {
      this.params.colorCycle = (this.params.colorCycle + 1) % this.colorSchemes.length;
      
      // 更强的爆炸效果
      gsap.to(this.heart.material.uniforms.uSize, {
        value: 0.5, // 更大的爆炸尺寸
        duration: 0.2,
        ease: "power2.out",
        onComplete: () => {
          gsap.to(this.heart.material.uniforms.uSize, {
            value: 0.38,
            duration: 0.5,
            ease: "power2.in"
          });
        }
      });
      
      const colors = new Float32Array(this.params.heartParticles * 3);
      const colorChoices = this.colorSchemes[this.params.colorCycle];
      
      for (let i = 0; i < this.params.heartParticles; i++) {
        const color = new THREE.Color(colorChoices[Math.floor(Math.random() * colorChoices.length)]);
        colors[i * 3] = color.r;
        colors[i * 3 + 1] = color.g;
        colors[i * 3 + 2] = color.b;
      }
      
      this.heart.geometry.setAttribute(
        "aColor", 
        new THREE.InstancedBufferAttribute(colors, 3)
      );
      
      const lightColor = new THREE.Color(colorChoices[0]);
      gsap.to(this.pointLight.color, {
        r: lightColor.r,
        g: lightColor.g,
        b: lightColor.b,
        duration: 1
      });
    });
    
    window.addEventListener('resize', () => {
      this.camera.aspect = window.innerWidth / window.innerHeight;
      this.camera.updateProjectionMatrix();
      this.renderer.setSize(window.innerWidth, window.innerHeight);
    });
  }
  
  // 简化的UI元素
  addUIElements() {
    const style = document.createElement('style');
    style.textContent = `
      body { margin: 0; overflow: hidden; background: #000; }
      .info {
        position: fixed; bottom: 20px; left: 20px;
        color: rgba(255,255,255,0.9); font-family: 'Arial', sans-serif; font-size: 16px;
        background: rgba(0,0,0,0.3); padding: 12px 18px; border-radius: 20px;
        backdrop-filter: blur(5px); border: 1px solid rgba(255,255,255,0.15);
        box-shadow: 0 4px 15px rgba(0,0,0,0.3);
      }
      .info p {
        margin: 6px 0;
        text-shadow: 0 2px 3px rgba(0,0,0,0.8);
      }
    `;
    document.head.appendChild(style);
    
    const info = document.createElement('div');
    info.className = 'info';
    info.innerHTML = `
      <p>✨ 移动鼠标改变视角</p>
      <p>💖 点击屏幕切换颜色</p>
    `;
    document.body.appendChild(info);
  }
  
  // 简化的动画循环
  animate() {
    const elapsedTime = this.clock.getElapsedTime();
    
    if (this.heart) {
      this.heart.material.uniforms.uTime.value = elapsedTime;
      this.heart.rotation.y = Math.sin(elapsedTime * 0.2) * 0.18;
      this.heart.rotation.z = Math.sin(elapsedTime * 0.1) * 0.06;
    }
    
    if (this.stars) {
      this.stars.material.uniforms.uTime.value = elapsedTime;
      this.stars.rotation.y = elapsedTime * 0.02;
    }
    
    if (this.glow) {
      this.glow.material.uniforms.uTime.value = elapsedTime;
      const pulseScale = 1 + Math.sin(elapsedTime * 0.8) * 0.12;
      this.glow.scale.set(pulseScale, pulseScale, pulseScale);
    }
    
    this.camera.lookAt(this.scene.position);
    this.renderer.render(this.scene, this.camera);
    
    requestAnimationFrame(this.animate.bind(this));
  }
}

// 创建世界
new LoveParticlesWorld();
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/loveShader.js)

## 小结

- 本文提供 **爱心** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

