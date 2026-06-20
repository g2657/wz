---
title: "Three.js 几何合并教程"
description: "详解 Three.js 几何合并：基于 WebGL 实现「几何合并」可视化效果，附完整可运行源码，涵盖 OrbitControls、glTF/Draco 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,几何合并,WebGL,源码,教程,在线案例,OrbitControls,相机控制,glTF,模型加载"
outline: deep
---
### 几何合并 · *Geometry Merge* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=geometryMerge)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![几何合并](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/geometryMerge.jpg)

## 你将学到什么

- glTF/FBX/OBJ 外部模型加载
- 相机交互控制器
- 实时阴影 ShadowMap
- 天空盒与环境贴图
- requestAnimationFrame 渲染循环

## 效果说明

本案例演示 **几何合并** 效果：基于 WebGL 实现「几何合并」可视化效果，附完整可运行源码；核心用到 OrbitControls、glTF/Draco。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **Loader** 异步加载模型；glTF 返回 `gltf.scene`，加载后注意 `scale` 与坐标系。Draco 需配置 `DRACOLoader`。

- **OrbitControls** 轨道旋转缩放；开 `enableDamping` 时每帧需 `controls.update()`。

- 阴影四步：`renderer.shadowMap.enabled`、光源 `castShadow`、物体 `castShadow`、地面 `receiveShadow`。

- **CubeTexture** 六面贴图作 `scene.background`；`scene.environment` 供 PBR 材质反射。

## 实现步骤

1. 搭建 Scene / Camera / Renderer 与 OrbitControls
2. Loader 异步加载模型/纹理资源
3. rAF 循环中 update 并 render

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js'
import * as BufferGeometryUtils from 'three/examples/jsm/utils/BufferGeometryUtils.js'

// 基础场景设置
const box = document.getElementById('box')
const scene = new THREE.Scene()
const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000000)
camera.position.set(5, 5, 5)

// 渲染器设置
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })
renderer.setSize(box.clientWidth, box.clientHeight)
renderer.setPixelRatio(window.devicePixelRatio * 2)
renderer.shadowMap.enabled = true
renderer.shadowMap.type = THREE.PCFSoftShadowMap
box.appendChild(renderer.domElement)

// 控制器
const controls = new OrbitControls(camera, renderer.domElement)
controls.enableDamping = true

// 光源
scene.add(new THREE.AmbientLight(0xffffff, 3))

// 响应式调整
window.onresize = () => {
    renderer.setSize(box.clientWidth, box.clientHeight)
    camera.aspect = box.clientWidth / box.clientHeight
    camera.updateProjectionMatrix()
}

// 动画循环
function animate() {
    renderer.render(scene, camera)
    controls.update()
    requestAnimationFrame(animate)
}
animate()

// 几何体合并函数
function mergeModelGeometries(model, options = {}) {
    const { 
        material = new THREE.MeshStandardMaterial({ 
            color: 0x666666, metalness: 0.5, roughness: 0.5,
            polygonOffset: true, polygonOffsetFactor: 1.0, polygonOffsetUnits: 1.0
        }), 
        ignoreHidden = true,
        mergeThreshold = 1e-4
    } = options;
    
    model.updateWorldMatrix(true, true);
    
    // 收集网格
    const meshes = [];
    model.traverse(object => {
        if (object.isMesh && object.geometry && (!ignoreHidden || object.visible)) {
            meshes.push(object);
        }
    });
    
    if (meshes.length === 0) return null;
    
    // 找出共有属性
    const commonAttribs = new Set();
    let refGeom = null;
    
    meshes.forEach(mesh => {
        const geom = mesh.geometry;
        if (!refGeom) {
            refGeom = geom;
            Object.keys(geom.attributes).forEach(name => commonAttribs.add(name));
        } else {
            for (const name of [...commonAttribs]) {
                if (!geom.attributes[name]) commonAttribs.delete(name);
            }
        }
    });
    
    if (!commonAttribs.has('position')) commonAttribs.add('position');
    
    try {
        // 预处理几何体
        const geometries = meshes.map(mesh => {
            const geom = mesh.geometry.clone();
            
            commonAttribs.forEach(name => {
                if (!geom.attributes[name]) {
                    if (name === 'normal') {
                        geom.computeVertexNormals();
                    } else if (name !== 'position') {
                        const itemSize = refGeom.attributes[name].itemSize;
                        geom.setAttribute(name, new THREE.BufferAttribute(
                            new Float32Array(geom.attributes.position.count * itemSize), itemSize
                        ));
                    }
                }
            });
            
            Object.keys(geom.attributes).forEach(name => {
                if (!commonAttribs.has(name)) geom.deleteAttribute(name);
            });
            
            geom.applyMatrix4(mesh.matrixWorld);
            return geom;
        });
        
        const mergedGeometry = BufferGeometryUtils.mergeGeometries(geometries);
        
        if (mergeThreshold > 0 && mergedGeometry.mergeVertices) {
            mergedGeometry.mergeVertices(mergeThreshold);
        }
        
        return new THREE.Mesh(mergedGeometry, material);
        
    } catch (error) {
        console.error('模型合并失败:', error);
        return null;
    }
}

// 文件地址
const urls = [0, 1, 2, 3, 4, 5].map(k => (FILE_HOST + 'files/sky/skyBox0/' + (k + 1) + '.png'));

const textureCube = new THREE.CubeTextureLoader().load(urls);

new GLTFLoader()

    .setDRACOLoader(new DRACOLoader().setDecoderPath(FILE_HOST + 'js/three/draco/'))

    .load(FILE_HOST + "files/model/LittlestTokyo.glb", gltf => {

        const mergedModel = mergeModelGeometries(gltf.scene, {
            material: new THREE.MeshStandardMaterial({
                metalness: 1, roughness: 0, envMap: textureCube,
                polygonOffset: true, polygonOffsetFactor: 1.0, polygonOffsetUnits: 1.0
            })
        })
        
        if (mergedModel) {
            scene.add(mergedModel);

            console.log('模型合并成功:', mergedModel);        
            // 调整相机视角
            const box = new THREE.Box3().setFromObject(mergedModel);
            const size = box.getSize(new THREE.Vector3()).length();
            const center = box.getCenter(new THREE.Vector3());
            
            camera.near = size * 0.01;
            camera.far = size * 100;
            camera.position.set(center.x, center.y, center.z + size * 1.5);
            camera.lookAt(center);
            controls.target.copy(center);
            camera.updateProjectionMatrix();
        }
    });
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/geometryMerge.js)

## 小结

- 本文提供 **几何合并** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

