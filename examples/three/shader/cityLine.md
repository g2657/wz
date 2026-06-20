---
title: "Three.js 城市线条教程"
description: "详解 Three.js 城市线条：基于 WebGL 实现「城市线条」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls、FBXLoader 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,城市线条,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制,FBX"
outline: deep
---

### 城市线条 · *City Line* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=cityLine)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![城市线条](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/cityLine.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- FBXLoader 加载 FBX 城市/角色模型
- EdgesGeometry 模型边线提取
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **城市线条** 效果：基于 WebGL 实现「城市线条」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls、FBXLoader。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 异步加载模型 / 3D Tiles / GeoJSON 等资源并加入 scene 或 entities
3. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
4. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
5. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { FBXLoader } from 'three/examples/jsm/loaders/FBXLoader.js'

class City {

    constructor() {
        this.fbxLoader = new FBXLoader();
        this.group = new THREE.Group();
        this.clock = new THREE.Clock()
        this.surroundLineMaterial = null;// 定义包围线材质属性
        this.time = { value: 0 };
        this.startTime = { value: 0 };
        this.startLength = { value: 2 }
        this.isStart = false;

        this.fbxLoader.load(FILE_HOST + 'models/fbx/shanghai.FBX', (group) => {

            this.group.add(group);
            group.traverse((child) => {
                // 设置城市建筑（mesh物体），材质基本颜色
                if (child.name == 'CITY_UNTRIANGULATED') {
                    const materials = Array.isArray(child.material) ? child.material : [child.material]
                    materials.forEach((material) => {
                        material.transparent = true;
                        material.color.setStyle("#9370DB");
                    })
                }

                // 设置城市地面（mesh物体），材质基本颜色
                if (child.name == 'LANDMASS') {
                    const materials = Array.isArray(child.material) ? child.material : [child.material]
                    materials.forEach((material) => {
                        material.transparent = true;
                        material.color.setStyle("#040912");
                    })
                }
            })

            this.init();
        });
    }

    // 初始化城市类的实例
    init() {
        this.isStart = true; // 城市渲染启动
        this.clock.start()
        this.surroundLine();
    }

    // 创建包围线条效果
    surroundLine() {
        let cityBuildings // 城市建筑群
        this.group.traverse(child => {
            if (child.name !== 'CITY_UNTRIANGULATED') return
            cityBuildings = child
        })

        const geometry = new THREE.EdgesGeometry(cityBuildings.geometry);
        const surroundLineMaterial = new THREE.ShaderMaterial({
            transparent: true,
            uniforms: {
                uColor: {
                    value: new THREE.Color('#FFF')
                }
            },
            vertexShader: `
         void main() {
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
          }
        `,
            fragmentShader: ` 
          uniform vec3 uColor;
          void main() {
            gl_FragColor = vec4(uColor, 1);
          }
        `
        });


        const line = new THREE.LineSegments(geometry, surroundLineMaterial);
        line.name = 'surroundLine';
        line.applyMatrix4(cityBuildings.matrix.clone())
        cityBuildings.parent.add(line)
    }

    updateData = () => {
        if (!this.isStart) return false;
        const dt = this.clock.getDelta();
        this.time.value += dt;
        this.startTime.value += dt;
        if (this.startTime.value >= this.startLength.value) {
            this.startTime.value = this.startLength.value;
        }
    }
}

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000000)
camera.position.set(600, 750, -1221)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true })
renderer.setSize(box.clientWidth, box.clientHeight)
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.setClearColor(new THREE.Color('#32373E'), 1);

const controls = new OrbitControls(camera, renderer.domElement)
controls.enableDamping = true

window.onresize = () => {
    renderer.setSize(box.clientWidth, box.clientHeight)
    camera.aspect = box.clientWidth / box.clientHeight
    camera.updateProjectionMatrix()
}

box.appendChild(renderer.domElement)

// 坐标轴
scene.add(new THREE.AxesHelper(100000))
const light = new THREE.AmbientLight(0xadadad, 3); // 环境光，soft white light
const directionalLight = new THREE.DirectionalLight(0xffffff, 1.5); // 方向光
directionalLight.position.set(100, 100, 0);
scene.add(light);
scene.add(directionalLight);

// 实例化
const city = new City({});
scene.add(city.group);

// 渲染
animate()
function animate() {
    city.updateData();
    controls.update()
    renderer.render(scene, camera)
    requestAnimationFrame(animate)
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/cityLine.js)

## 小结

- 本文提供 **城市线条** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

