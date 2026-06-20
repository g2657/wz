---
title: "Three.js 相机动画教程"
description: "详解 Three.js 相机动画：基于 WebGL 实现「相机动画」可视化效果，附完整可运行源码，涵盖 OrbitControls、glTF/Draco、GSAP 等关键实现，附完整源码与在线 Demo，适合 Three.js 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,相机动画,WebGL,源码,教程,在线案例,OrbitControls,相机控制,glTF,模型加载,GSAP,动画"
outline: deep
---

### 相机动画 · *Camera FX* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=basic&id=cameraAnimate)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

## 你将学到什么

- gsap **timeline repeat:-1** 无限抖动
- 可选择抖动 **camera.position** 或 **controls.target**
- 相机 + target **同步 Y 轴跳跃**

## 效果说明

导弹模型 + 天空盒背景。GUI：**目标抖动**（随机偏移 ±3）、**停止抖动**、**整体跳跃**（camera 与 target 同时 yoyo）。

## 核心概念

```js
shakeAnimation = gsap.timeline({ repeat: -1, yoyo: true });
shakeAnimation.to(c, {
    x: orig.x + (Math.random() - 0.5) * intensity,
    duration: 0.3,
    ease: 'power1.inOut'
});
// 停止：shakeAnimation.kill();
```

受击反馈、地震、手持摄影机效果常用此模式。

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { GUI } from 'dat.gui'
import gsap from 'gsap'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(50, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(20, 40, 60)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

animate()

function animate() {

  requestAnimationFrame(animate)

  controls.update()

  renderer.render(scene, camera)

}

window.onresize = () => {

  renderer.setSize(box.clientWidth, box.clientHeight)

  camera.aspect = box.clientWidth / box.clientHeight

  camera.updateProjectionMatrix()

}

// 文件地址
const urls = [0, 1, 2, 3, 4, 5].map(k => (FILE_HOST + 'files/sky/skyBox0/' + (k + 1) + '.png'));

const textureCube = new THREE.CubeTextureLoader().load(urls);

scene.background = textureCube;

new GLTFLoader().load(FILE_HOST + 'models/glb/daodan.glb', gltf => {

  gltf.scene.traverse(child => {

    if (child.isMesh) {

      child.material.envMap = textureCube

    }

  })

  scene.add(gltf.scene)

})

const gui = new GUI()

let shakeAnimation;

const t = { type: 'camera' }

gui.add(t, 'type', ['camera', 'target']).name('类型');

gui.add({
  fn: () => {

    let c = t.type === 'camera' ? camera.position : controls.target;

    // 镜头无限慢慢抖动
    const shakeIntensity = 6;
    const shakeDuration = 0.3;
    const shake = () => {
      const originalPosition = c.clone();
      shakeAnimation = gsap.timeline({ repeat: -1, yoyo: true });
      shakeAnimation.to(c, {
        x: originalPosition.x + (Math.random() - 0.5) * shakeIntensity,
        y: originalPosition.y + (Math.random() - 0.5) * shakeIntensity,
        z: originalPosition.z + (Math.random() - 0.5) * shakeIntensity,
        duration: shakeDuration,
        ease: 'power1.inOut'
      });
    };

    shake();

  }
}, 'fn').name('目标抖动');

gui.add({
  fn: () => {
    // 停止镜头抖动
    if (shakeAnimation) {
      shakeAnimation.kill();
      shakeAnimation = null;
    }
  }
}, 'fn').name('停止抖动');

// 
gui.add({
  fn: () => {
    const c = camera.position;
    const t = controls.target;
    const jumpHeight = 50;
    const jumpDuration = 0.5;
    const jump = () => {
      // 同时为相机位置和目标点添加跳跃动画
      [c, t].forEach(obj => {
        gsap.to(obj, {
          y: obj.y + jumpHeight,
          duration: jumpDuration,
          ease: 'power2.out',
          yoyo: true,
          repeat: 1,
          onComplete: () => {
    
          }
        });
      });
    };

    jump();
  }
}, 'fn').name('整体跳跃');
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/basic/cameraAnimate.js)

## 小结
