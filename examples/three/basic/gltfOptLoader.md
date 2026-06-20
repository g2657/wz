---
title: "Three.js Opt 解压（su7 模型）教程"
description: "详解 Three.js Opt 解压（su7 模型）：基于 WebGL 实现「Opt 解压（su7 模型）」可视化效果，附完整可运行源码，涵盖 OrbitControls、glTF/Draco 等关键实现，附完整源码与在线 Demo，适合 Three.js 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,Opt 解压（su7 模型）,WebGL,源码,教程,在线案例,OrbitControls,相机控制,glTF,模型加载"
outline: deep
---

### Opt 解压（su7 模型） · *GLTF Meshopt + HDR* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=basic&id=gltfOptLoader)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![Opt解压](https://z2586300277.github.io/three-cesium-examples/threeExamples/basic/gltfOptLoader.jpg)

## 你将学到什么

- **Meshopt** 几何压缩与 `setMeshoptDecoder`
- **HDR + PMREMGenerator** 生成 PBR 环境贴图
- 加载后为 Mesh **逐个设置 envMap**

## 效果说明

加载小米 SU7 的 **Meshopt 压缩 glTF** 模型，在 **HDR 环境反射** 下展示车身 PBR 质感。OrbitControls 环绕观察。

## 核心概念

### Meshopt 压缩

除 Draco 外，glTF 还常用 **EXT_meshopt_compression**（Meshopt）压缩顶点数据，体积更小、解码 often 更快：

```js
import { MeshoptDecoder } from 'three/examples/jsm/libs/meshopt_decoder.module.js';

const loader = new GLTFLoader();
loader.setMeshoptDecoder(MeshoptDecoder);
loader.load('sm_car.gltf', gltf => scene.add(gltf.scene));
```

| 压缩 | 解码器 |
|------|--------|
| Draco | `DRACOLoader` |
| Meshopt | `MeshoptDecoder` |

### HDR → PMREM 环境贴图

```js
const pmremGenerator = new THREE.PMREMGenerator(renderer);
const texture = new RGBELoader().load('1k.hdr', (t) => {
    const map = pmremGenerator.fromEquirectangular(t).texture;
    pmremGenerator.dispose();
    return map;
});
texture.mapping = THREE.EquirectangularReflectionMapping;
```

**PMREM**（Prefiltered Mipmapped Radiance Environment Map）把 HDR 全景预滤波为各级 mip，供 PBR 材质按粗糙度采样反射。

### 手动 envMap

本案例未设 `scene.environment`，而是 traverse 给每个 mesh：

```js
gltf.scene.traverse(obj => {
    if (obj.isMesh) obj.material.envMap = texture;
});
```

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { MeshoptDecoder } from "three/examples/jsm/libs/meshopt_decoder.module.js"
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 2, 3)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true })

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

// HDR
const pmremGenerator = new THREE.PMREMGenerator(renderer);

const texture = new RGBELoader().load(FILE_HOST + '/files/hdr/1k.hdr', (t) => {

    const map = pmremGenerator.fromEquirectangular(t).texture

    pmremGenerator.dispose()

    return map

})

texture.mapping = THREE.EquirectangularReflectionMapping

// GLTF
const loader = new GLTFLoader()

loader.setMeshoptDecoder(MeshoptDecoder)

loader.load(FILE_HOST + 'models/su7/sm_car.gltf', gltf => {

    scene.add(gltf.scene)

    gltf.scene.traverse(obj => {

        if (obj.isMesh) {

            obj.material.envMap = texture

        }

    })

})
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/basic/gltfOptLoader.js)

## 小结

- 本文提供 **Opt 解压（su7 模型）** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

