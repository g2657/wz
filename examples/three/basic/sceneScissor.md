---
title: "Three.js 场景剪切 - 后处理教程"
description: "详解 Three.js 场景剪切 - 后处理：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期，涵盖 EffectComposer、UnrealBloomPass、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,场景剪切 - 后处理,WebGL,源码,教程,在线案例,EffectComposer,后期处理,Bloom,辉光,OrbitControls,相机控制"
outline: deep
---

### 场景剪切 - 后处理 · *Scissor Compare* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=basic&id=sceneScissor)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

## 你将学到什么

- EffectComposer 多 Pass 后期处理管线
- UnrealBloomPass 辉光 Bloom 效果
- OrbitControls 相机轨道交互

## 效果说明

100 个随机立方体。左侧 **无辉光**，右侧 **UnrealBloomPass**，中间竖线可拖拽，类似 PS 对比滑块。

## 核心概念

```js
renderer.setScissorTest(true);
renderer.setScissor(0, 0, splitX, height);
composer_original.render();

renderer.setScissor(splitX, 0, width - splitX, height);
composer_bloom.render();
renderer.setScissorTest(false);
```

同一 scene/camera，不同 composer 输出到 canvas 不同区域。

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass.js';
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';

const box = document.getElementById('box')
const scene = new THREE.Scene()
const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)
camera.position.set(0, 10, 10)
const renderer = new THREE.WebGLRenderer({antialias: true, alpha: true , logarithmicDepthBuffer: true})
renderer.setSize(box.clientWidth, box.clientHeight)
renderer.setAnimationLoop(animate)
box.appendChild(renderer.domElement)
new OrbitControls(camera, renderer.domElement)

const renderPass = new RenderPass(scene, camera);

// 无辉光渲染
const composer_original = new EffectComposer(renderer);
composer_original.addPass(renderPass);

// 辉光渲染
const composer_bloom = new EffectComposer(renderer);
composer_bloom.addPass(renderPass);
composer_bloom.addPass( new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0, 0));


// 设置分割线位置
let initialWidth = 350
createSlider(document.body, initialWidth, (left) => initialWidth = left)

function animate() {

    renderer.setScissorTest( true )

    renderer.setScissor( 0, 0, initialWidth, box.offsetHeight );
    composer_original.render()

    renderer.setScissor( initialWidth, 0, box.clientWidth - initialWidth, box.offsetHeight );
    composer_bloom.render();

    renderer.setScissorTest( false )

}

// 物体
for (let i = 0; i < 100; i++) {
    const geometry = new THREE.BoxGeometry(1, 1, 1)
    const material = new THREE.MeshBasicMaterial({ color: Math.random() * 0xffffff })
    const cube = new THREE.Mesh(geometry, material)
    cube.position.set(Math.random() * 10 - 5, Math.random() * 10 - 5, Math.random() * 10 - 5)
    scene.add(cube)
}

/* 分割滑块方法 */
function createSlider(box, initialWidth, callback) {

    const minLeftWidth = 50;
    const minRightWidth = 100;
    const slider_dom = document.createElement('div')
    box.prepend(slider_dom)

    const color = 'rgba(255, 255, 255, 0.5)'
    Object.assign(slider_dom.style, {
        position: 'absolute',
        left: initialWidth + 'px',
        height: box.clientHeight + 'px',
        transition: 'background 0.5s',
        backgroundColor: color,
        width: '2px',
        cursor: 'ew-resize',
    })

    const move = () => {
        slider_dom.style.backgroundColor = '#277CD5'
        document.body.style.cursor = 'ew-resize'
    }
    const leave = () => {
        slider_dom.style.backgroundColor = color
        document.body.style.cursor = 'default'
    }

    slider_dom.onmousemove = move
    slider_dom.onmouseleave = leave

    slider_dom.ondblclick = function () {
        slider_dom.style.left = initialWidth + 'px'
        callback?.(initialWidth)
    }

    slider_dom.onmousedown = function (e) {

        e.preventDefault()
        let old_left = slider_dom.getBoundingClientRect().left - box.getBoundingClientRect().left

        document.onmousemove = function (e) {

            move()

            if (old_left + e.movementX < minLeftWidth) return
            if (old_left + e.movementX > box.clientWidth - minRightWidth) return

            old_left += e.movementX;
            slider_dom.style.left = old_left + 'px';
            callback?.(old_left)
        }

        document.onmouseup = function () {
            document.onmousemove = null;
            leave()
        }
    }
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/basic/sceneScissor.js)

## 小结
