---
title: "Three.js 城市混合扫光教程"
description: "详解 Three.js 城市混合扫光：基于 WebGL 实现「城市混合扫光」可视化效果，附完整可运行源码，涵盖 onBeforeCompile、OrbitControls、FBXLoader 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,城市混合扫光,WebGL,源码,教程,在线案例,onBeforeCompile,shader注入,OrbitControls,相机控制,FBX,模型加载"
outline: deep
---

### 城市混合扫光 · *City Blend Light* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=cityBlendLight)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![城市混合扫光](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/cityBlendLight.jpg)

## 你将学到什么

- 封装 **modelBlendShader** 批量改模型材质
- 用 **模型空间距离** 做环形扫光带
- `mix(diff, color3, r)` 双色渐变 + `intensity` 增亮
- `model.render` 钩子驱动 uniform 动画

## 效果说明

FBX 城市场景上，一道青蓝→深蓝的光环从中心向外扩散，扫过建筑表面；可选 `isDisCard` 丢弃暗色像素（镂空楼体）。

## 核心概念

### 扫光逻辑（片元）

```glsl
float dis = length(v_position - center);
if (dis < (innerCircleWidth + circleWidth) && dis > innerCircleWidth) {
    float r = (dis - innerCircleWidth) / circleWidth;
    diffuseColor = vec4(mix(diff, color3, r) * intensity, opacity);
} else {
    if (isDisCard) discard;
    else diffuseColor = vec4(diffuse, opacity);
}
```

| uniform | 作用 |
|---------|------|
| `innerCircleWidth` | 环内缘半径（动画递增） |
| `circleWidth` | 环带宽度 |
| `circleMax` | 到达后重置为 0 |
| `center` | 扫光中心（模型空间 vec3） |

### 批量 onBeforeCompile

```js
model.traverse(c => c.isMesh && materials.push(c.material));
materials = [...new Set(materials)]; // 去重共享材质

materials.forEach(material => {
    material.onBeforeCompile = (shader) => {
        Object.keys(uniforms).forEach(key => shader.uniforms[key] = uniforms[key]);
        // 替换 void main / diffuseColor 行
    };
});

model.render = () => {
    if (uniforms.innerCircleWidth.value < uniforms.circleMax.value)
        uniforms.innerCircleWidth.value += uniforms.circleSpeed.value;
    else uniforms.innerCircleWidth.value = 0;
};
```

animate 里调用 `model.render?.()`。

## 实现步骤

1. FBXLoader 加载 city.FBX，scale/position 调整
2. `modelBlendShader(object3d)` 收集材质、注入 GLSL
3. rAF 更新 innerCircleWidth 并 render

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { FBXLoader } from 'three/examples/jsm/loaders/FBXLoader.js'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 100000)

camera.position.set(157, 545, -987)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

renderer.setClearColor(0x000000, 0)

renderer.setPixelRatio(window.devicePixelRatio * 2)

new OrbitControls(camera, renderer.domElement)

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

}

box.appendChild(renderer.domElement)

const dirLight = new THREE.DirectionalLight(0xffffff, 3.8)

dirLight.position.set(83, 61, -183)

dirLight.target.position.set(10, -11, -194)

scene.add(dirLight)

const pointLight = new THREE.PointLight(0xffffff, 2)

pointLight.position.set(-60, 182, -98)

scene.add(pointLight)

let model = null

// 加载模型
new FBXLoader().load(HOST + '/files/model/city.FBX', (object3d) => {

    scene.add(object3d)

    object3d.scale.set(0.04, 0.04, 0.04)

    object3d.position.set(224, -9, -49)

    model = object3d

    modelBlendShader(object3d, box)

})

// 渲染
animate()

function animate() {

    model && model.render?.()

    renderer.render(scene, camera)

    requestAnimationFrame(animate)

}

/* 混合着色 */
function modelBlendShader(model) {

    let materials = []

    model.traverse(c => c.isMesh && materials.push(c.material))

    materials = [... new Set(materials)]

    const uniforms = {

        innerCircleWidth: { value: 480, type: 'number', unit: 'float' },

        circleWidth: { value: 160, type: 'number', unit: 'float' },

        circleMax: { value: 940, type: 'number', unit: 'float' },

        circleSpeed: { value: 1.5, type: 'number', unit: 'float' },

        diff: { value: new THREE.Color(0x6edbe8), type: 'color', unit: 'vec3' },

        color3: { value: new THREE.Color(0x1919f9), type: 'color', unit: 'vec3' },

        center: { value: new THREE.Vector3(-1, 0, 0), type: 'position', unit: 'vec3' },

        intensity: { value: 4, type: 'number', unit: 'float' },

        isDisCard: { value: false, type: 'bool', unit: 'bool' },

    }

    const glslProps = {

        vertexHeader: `
            varying vec2 vUv;
            varying vec3 v_position;
            void main() {
                vUv = uv;
                v_position = position;
        `,

        fragHeader: Object.keys(uniforms).map(i => 'uniform ' + uniforms[i].unit + ' ' + i + ';').join('\n') + '\n' + 'varying vec3 v_position; varying vec2 vUv;\n',

        fragBody: `
            float dis = length(v_position - center);
            vec4 diffuseColor;
            if(dis < (innerCircleWidth + circleWidth) && dis > innerCircleWidth) {
                float r = (dis - innerCircleWidth) / circleWidth;
                #ifdef USE_MAP
                    vec3 textureColor = texture2D(map, vUv).rgb;
                    if(isDisCard && textureColor.r < 0.1 && textureColor.g < 0.1  && textureColor.b < 0.1 ) discard;
                #endif
                diffuseColor = vec4( mix(diff, color3, r) * vec3(intensity, intensity, intensity)  , opacity);
            }
            else {
                if(isDisCard)  discard ;
                else diffuseColor = vec4( diffuse, opacity );
            }
        `

    }

    materials.forEach(material => {

        material.onBeforeCompile = (shader) => {

            Object.keys(uniforms).forEach((key) => shader.uniforms[key] = uniforms[key])

            shader.vertexShader = shader.vertexShader.replace(`void main() {`, glslProps.vertexHeader)

            shader.fragmentShader = shader.fragmentShader.replace(/#include <common>/, glslProps.fragHeader + '\n#include <common>\n')

            shader.fragmentShader = shader.fragmentShader.replace('vec4 diffuseColor = vec4( diffuse, opacity );', glslProps.fragBody)

        }

        material.needsUpdate = true

    })

    model.render = () => uniforms.innerCircleWidth.value < uniforms.circleMax.value ? uniforms.innerCircleWidth.value += uniforms.circleSpeed.value : uniforms.innerCircleWidth.value = 0

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/cityBlendLight.js)

## 小结

- 本文提供 **城市混合扫光** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

