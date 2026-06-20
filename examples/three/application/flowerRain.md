---
title: "Three.js 花瓣雨教程"
description: "详解 Three.js 花瓣雨：基于 WebGL 实现「花瓣雨」可视化效果，附完整可运行源码，涵盖 OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,花瓣雨,WebGL,源码,教程,在线案例,OrbitControls,相机控制"
outline: deep
---
### 花瓣雨 · *Flower Rain* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=flowerRain)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![花瓣雨](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/flowerRain.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **花瓣雨** 效果：基于 WebGL 实现「花瓣雨」可视化效果，附完整可运行源码；核心用到 OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **OrbitControls** 轨道旋转缩放；开 `enableDamping` 时每帧需 `controls.update()`。

## 实现步骤

1. 搭建 Scene / Camera / Renderer 与 OrbitControls
2. rAF 循环中 update 并 render

## 代码要点

```js
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

const box = document.getElementById('box');

const scene = new THREE.Scene();
/**
 * 花瓣分组
 */
const petal = new THREE.Group();

const width = box.clientWidth;
const height = box.clientHeight;
//窗口宽高比
const k = width / height;
//三维场景的显示的上下范围
const s = 200;
const camera = new THREE.OrthographicCamera(-s * k, s * k, s, -s, 1, 1000);

const renderer = new THREE.WebGLRenderer();

function create() {
    //设置相机位置
    camera.position.set(0, 200, 500)
    camera.lookAt(scene.position)

    //设置渲染区域尺寸
    renderer.setSize(width, height)
    //设置背景颜色

    //body元素中插入canvas对象
    box.appendChild(renderer.domElement)

    // const axisHelper = new THREE.AxisHelper(1000);
    // scene.add(axisHelper)

    var flowerTexture1 = new THREE.TextureLoader().load(FILE_HOST + "examples/flowerAndHouse/img/flower1.png");
    var flowerTexture2 = new THREE.TextureLoader().load(FILE_HOST + "examples/flowerAndHouse/img/flower2.png");
    var flowerTexture3 = new THREE.TextureLoader().load(FILE_HOST + "examples/flowerAndHouse/img/flower3.png");
    var flowerTexture4 = new THREE.TextureLoader().load(FILE_HOST + "examples/flowerAndHouse/img/flower4.png");
    var flowerTexture5 = new THREE.TextureLoader().load(FILE_HOST + "examples/flowerAndHouse/img/flower5.png");
    var imageList = [flowerTexture1, flowerTexture2, flowerTexture3, flowerTexture4, flowerTexture5];

    for (let i = 0; i < 400; i++) {
        var spriteMaterial = new THREE.SpriteMaterial({
            map: imageList[Math.floor(Math.random() * imageList.length)],//设置精灵纹理贴图
        });
        var sprite = new THREE.Sprite(spriteMaterial);
        petal.add(sprite);

        sprite.scale.set(40, 50, 1); 
        sprite.position.set(2000 * (Math.random() - 0.5), 500 * Math.random(), 2000 * (Math.random() - 0.5))
    }
    scene.add(petal)
}


function render() {
    petal.children.forEach(sprite => {
        sprite.position.y -= 5;
        sprite.position.x += 0.5;
        if (sprite.position.y < - height / 2) {
            sprite.position.y = height / 2;
        }
        if (sprite.position.x > 1000) {
            sprite.position.x = -1000
        }
    });

    renderer.render(scene, camera)

    requestAnimationFrame(render)
}

create()
render()

const controls = new OrbitControls(camera, renderer.domElement);
controls.autoRotate = true;
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/flowerRain.js)

## 小结

- 本文提供 **花瓣雨** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

