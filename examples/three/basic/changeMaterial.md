---
title: "Three.js 材质修改动画教程"
description: "详解 Three.js 材质修改动画：基于 WebGL 实现「材质修改动画」可视化效果，附完整可运行源码，涵盖 OrbitControls、glTF/Draco、GSAP 等关键实现，附完整源码与在线 Demo，适合 Three.js 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,材质修改动画,WebGL,源码,教程,在线案例,OrbitControls,相机控制,glTF,模型加载,GSAP,动画"
outline: deep
---

### 材质修改动画 · *Material Tweak* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=basic&id=changeMaterial)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![材质修改动画](https://z2586300277.github.io/three-cesium-examples/threeExamples/basic/changeMaterial.jpg)

## 你将学到什么

- `group.traverse` 按 **name** 找到特定部件材质
- **MeshStandardMaterial** 的 wireframe / opacity / metalness / roughness
- **gsap.to(material.color)** 颜色过渡动画

## 效果说明

加载汽车 glb，针对名为 `网格138_3` 的车身部件，GUI 调节线框、透明、金属度等；改颜色时用 **gsap 1.5 秒插值**，避免突变。

## 核心概念

```js
group.traverse(child => {
    if (child.isMesh && child.name === '网格138_3') {
        material = child.material;
        material.envMap = textureCube;  // 天空盒反射
    }
});

// gsap 动画改颜色
folder.addColor({ color: material.color.clone() }, 'color').onChange(c => {
    gsap.to(material.color, { ...c, duration: 1.5 });
});
```

::: tip
glTF 子节点 name 来自建模软件，加载后 `console.log` 或编辑器查看再 traverse。
:::

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js'
import { GUI } from "three/addons/libs/lil-gui.module.min.js"
import gsap from 'gsap'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(1, 2, 2)

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

animate()

function animate() {

    requestAnimationFrame(animate)

    controls.update()

    renderer.render(scene, camera)

}

scene.add(new THREE.AmbientLight(0xffffff, 0.2))

const pointLight = new THREE.PointLight(0xffffff, 1.5, 0, 2)

pointLight.position.set(5, 5, 5)

scene.add(pointLight)

const directionalLight = new THREE.DirectionalLight(0xffffff, 2)

directionalLight.position.set(-5, 5, -5)

scene.add(directionalLight)

let group

// 加载模型 gltf/ glb  draco解码器
const loader = new GLTFLoader()

loader.setDRACOLoader(new DRACOLoader().setDecoderPath(FILE_HOST + 'js/three/draco/'))

loader.load(

    HOST + '/files/model/car.glb',

    gltf => {

        group = gltf.scene

        scene.add(group)

        changeMaterial()

    }

)

// 文件地址
const urls = [0, 1, 2, 3, 4, 5].map(k => (FILE_HOST + 'files/sky/skyBox0/' + (k + 1) + '.png'));

const textureCube = new THREE.CubeTextureLoader().load(urls);

function changeMaterial() {

    let material

    if (!group) return

    group.traverse(child => {

        if (child.isMesh && child.name === '网格138_3') {

            material = child.material

            material.envMap = textureCube

        }

    })

    const folder = new GUI()

    folder.add(material, 'wireframe').name('线框').listen()

    folder.add(material, 'transparent').name('透明').listen()

    folder.add(material, 'opacity').min(0).max(1).name('透明度').listen()

    folder.addColor({ color: material.color.clone() }, 'color').name('颜色').listen().onChange(c => {

        gsap.to(material.color, { ...c, duration: 1.5 })

    })

    folder.add(material, 'metalness').min(0).max(1).name('金属度').listen()

    folder.add(material, 'roughness').min(0).max(1).name('粗糙度').listen()

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/basic/changeMaterial.js)

## 小结

- 本文提供 **材质修改动画** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

