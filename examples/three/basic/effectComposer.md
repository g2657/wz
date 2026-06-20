---
title: "Three.js 渲染器配置教程"
description: "详解 Three.js 渲染器配置：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期，涵盖 EffectComposer、OrbitControls、glTF/Draco 等关键实现，附完整源码与在线 Demo，适合 Three.js 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,渲染器配置,WebGL,源码,教程,在线案例,EffectComposer,后期处理,OrbitControls,相机控制,glTF,模型加载"
outline: deep
---

### 渲染器配置 · *Renderer Config* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=basic&id=effectComposer)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

## 你将学到什么

- **toneMapping** 系列：Linear / Reinhard / ACESFilmic / AgX …
- **outputColorSpace** SRGB vs Linear
- **RoomEnvironment** + PMREM 快速 IBL
- GUI 切换 **effect / normal / both** 渲染路径

## 效果说明

LittlestTokyo 模型 + 室内环境光。调节曝光、色调映射，对比 **直接 renderer.render** 与 **composer+OutputPass** 成片差异。

## 核心概念

```js
scene.environment = pmrem.fromScene(new RoomEnvironment()).texture;

renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1;
renderer.outputColorSpace = THREE.SRGBColorSpace;

// OutputPass 做输出色彩空间最终转换
composer.addPass(new RenderPass(scene, camera));
composer.addPass(new OutputPass());
```

现代 Three.js **物理正确输出** 应走 OutputPass 或等价配置。

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js'
import { RoomEnvironment } from 'three/addons/environments/RoomEnvironment.js';
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
import { OutputPass } from 'three/examples/jsm/postprocessing/OutputPass.js'
import { GUI } from 'dat.gui'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(40, box.clientWidth / box.clientHeight, 0.1, 100000)
camera.position.set(5, 2, 8);

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })
renderer.setSize(box.clientWidth, box.clientHeight)
renderer.setClearColor(0xbfe3dd, 1)
box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)
controls.enableDamping = true

scene.environment = new THREE.PMREMGenerator(renderer).fromScene(new RoomEnvironment(), 0.04).texture;

const composer = new EffectComposer(renderer);

const renderPass = new RenderPass(scene, camera)
composer.addPass(renderPass);

const outputPass = new OutputPass()
composer.addPass(outputPass)

const data = { renderType: 'effect' }

const ToneMappingList = {
    No: THREE.NoToneMapping,
    Linear: THREE.LinearToneMapping,
    Reinhard: THREE.ReinhardToneMapping,
    Cineon: THREE.CineonToneMapping,
    ACESFilmic: THREE.ACESFilmicToneMapping,
    AgX: THREE.AgXToneMapping,
    Neutral: THREE.NeutralToneMapping,
    Custom: THREE.CustomToneMapping
}

const gui = new GUI()
gui.add(renderer, 'outputColorSpace', [THREE.SRGBColorSpace, THREE.LinearSRGBColorSpace])
gui.add(renderer, 'toneMapping', Object.keys(ToneMappingList)).onChange(value => renderer.toneMapping = ToneMappingList[value])
gui.add(renderer, 'toneMappingExposure', 0, 10)
gui.add(data, 'renderType', ['effect', 'normal', 'both'])
gui.add(outputPass, 'enabled').name('OutputPass_enabled')

animate()

function animate() {

    controls.update()
    if (data.renderType === 'effect') composer.render()
    else if (data.renderType === 'normal') renderer.render(scene, camera)
    else {
        renderer.render(scene, camera)
        composer.render()
    }
    requestAnimationFrame(animate)

}

const loader = new GLTFLoader()
loader.setDRACOLoader(new DRACOLoader().setDecoderPath(FILE_HOST + 'js/three/draco/'))
loader.load(
    FILE_HOST + '/files/model/LittlestTokyo.glb',
    gltf => {
        gltf.scene.position.set(1, 1, 0);
        gltf.scene.scale.set(0.01, 0.01, 0.01);
        scene.add(gltf.scene)
    }
)
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/basic/effectComposer.js)

## 小结
