---
title: "Three.js 模型混合着色器教程"
description: "详解 Three.js 模型混合着色器：基于 WebGL 实现「模型混合着色器」可视化效果，附完整可运行源码，涵盖 onBeforeCompile、OrbitControls、glTF/Draco 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,模型混合着色器,WebGL,源码,教程,在线案例,onBeforeCompile,shader注入,OrbitControls,相机控制,glTF,模型加载"
outline: deep
---
### 模型混合着色器 · *Model Blend* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=modelBlendShader)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![模型混合着色器](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/modelBlendShader.jpg)

## 你将学到什么

- glTF/FBX/OBJ 外部模型加载
- 自定义 ShaderMaterial / 修改内置 shader
- 相机交互控制器
- requestAnimationFrame 渲染循环

## 效果说明

本案例演示 **模型混合着色器** 效果：基于 WebGL 实现「模型混合着色器」可视化效果，附完整可运行源码；核心用到 onBeforeCompile、OrbitControls、glTF/Draco。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **Loader** 异步加载模型；glTF 返回 `gltf.scene`，加载后注意 `scale` 与坐标系。Draco 需配置 `DRACOLoader`。

- **ShaderMaterial** 完全自定义 GLSL；`onBeforeCompile` 可在内置材质 shader 中注入代码。关注 `uniforms` 与 rAF 更新。

- **OrbitControls** 轨道旋转缩放；开 `enableDamping` 时每帧需 `controls.update()`。

## 实现步骤

1. 搭建 Scene / Camera / Renderer 与 OrbitControls
2. Loader 异步加载模型/纹理资源
3. 定义材质/shader 与 uniforms，rAF 中更新
4. rAF 循环中 update 并 render

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(3, 3, 3)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

}

scene.add(new THREE.AmbientLight(0xffffff, 3))

scene.add(new THREE.AxesHelper(1000))

let car = null

const loader = new GLTFLoader()

loader.setDRACOLoader(new DRACOLoader().setDecoderPath(FILE_HOST + 'js/three/draco/'))

loader.load(

    HOST + '/files/model/car.glb',

    gltf => {

        car = gltf.scene

        scene.add(car)

        modelBlendShader(car, box)

    }

)

animate()

function animate() {

    requestAnimationFrame(animate)

    car?.render?.()

    renderer.render(scene, camera)

}

/* 混合着色 */
function modelBlendShader(model, DOM) {

    let materials = []

    model.traverse(c => c.isMesh && materials.push(c.material))

    materials = [... new Set(materials)]

    const uniforms = {

        iResolution: {
            type: 'v2',
            value: new THREE.Vector2(DOM.clientWidth, DOM.clientHeight)
        },

        iTime: {
            type: 'f',
            value: 1.0
        }

    }

    materials.forEach(material => {

        material.onBeforeCompile = shader => {

            shader.uniforms.iResolution = uniforms.iResolution

            shader.uniforms.iTime = uniforms.iTime

            shader.fragmentShader = shader.fragmentShader.replace(/#include <common>/, `
                uniform vec2 iResolution;
                uniform float iTime;
                #include <common> 
            `)

            shader.fragmentShader = shader.fragmentShader.replace('vec4 diffuseColor = vec4( diffuse, opacity );', `
                vec3 c;
                float l,z=iTime;
                for(int i=0;i<3;i++) {
                    vec2 uv,p=gl_FragCoord.xy/iResolution/2.0;
                    uv=p +  2.0;
                    p-=.5;
                    p.x*=iResolution.x/iResolution.y;
                    z+=.07;
                    l=length(p);
                    uv+=p/l*(sin(z)+1.)*abs(sin(l*9.-z-z));
                    c[i]=.01/length(mod(uv,1.)-.5);
                }
                vec4 diffuseColor = vec4( diffuse * c  * vec3(20.,20.,20.), opacity );
            `)

        }

        material.needsUpdate = true

    })

    model.render = () => uniforms.iTime.value += 0.02

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/modelBlendShader.js)

## 小结

- 本文提供 **模型混合着色器** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

