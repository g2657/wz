---
title: "Three.js 模型反射效果教程"
description: "详解 Three.js 模型反射效果：基于 WebGL 实现「模型反射效果」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、onBeforeCompile、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 扩展 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,模型反射效果,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,onBeforeCompile,shader注入,OrbitControls"
outline: deep
---

### 模型反射效果 · *Model Blend* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=expand&id=modelBlendReflector)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![模型反射效果](https://z2586300277.github.io/three-cesium-examples/threeExamples/expand/modelBlendReflector.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- onBeforeCompile 注入 GLSL 改造内置材质
- OrbitControls 相机轨道交互
- glTF/Draco 模型加载与优化
- 水面反射/镜像材质
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **模型反射效果** 效果：基于 WebGL 实现「模型反射效果」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、onBeforeCompile、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **onBeforeCompile** 在 Three 拼好内置 shader 后替换 `#include <xxx>` 片段，适合在 PBR 材质上叠加大屏特效。
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
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(2, 1, 3)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

new GLTFLoader().setDRACOLoader(new DRACOLoader().setDecoderPath(FILE_HOST + 'js/three/draco/')).load(HOST + '/files/model/car.glb', gltf => scene.add(gltf.scene))

const pointLight = new THREE.PointLight(0xffffff, 0.5, 0, 0)

pointLight.position.set(0, 10, 0)

scene.add(pointLight)

const AmbientLight = new THREE.AmbientLight(0xffffff, 0.5)

scene.add(AmbientLight)

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

import {
    Color,
    Matrix4,
    Mesh,
    PerspectiveCamera,
    Plane,
    ShaderMaterial,
    UniformsUtils,
    Vector3,
    Vector4,
    WebGLRenderTarget,
    HalfFloatType,
} from "three";

class Reflector extends Mesh {
    constructor(geometry, options = {}) {
        super(geometry);

        this.isReflector = true;

        this.type = "Reflector";
        this.camera = new PerspectiveCamera();

        const scope = this;

        const color =
            options.color !== undefined
                ? new Color(options.color)
                : new Color(0x7f7f7f);
        const textureWidth = options.textureWidth || 512;
        const textureHeight = options.textureHeight || 512;
        const clipBias = options.clipBias || 0;
        const shader = options.shader || Reflector.ReflectorShader;
        const multisample =
            options.multisample !== undefined ? options.multisample : 4;

        //

        const reflectorPlane = new Plane();
        const normal = new Vector3();
        const reflectorWorldPosition = new Vector3();
        const cameraWorldPosition = new Vector3();
        const rotationMatrix = new Matrix4();
        const lookAtPosition = new Vector3(0, 0, -1);
        const clipPlane = new Vector4();

        const view = new Vector3();
        const target = new Vector3();
        const q = new Vector4();

        const textureMatrix = new Matrix4();
        const virtualCamera = this.camera;

        const renderTarget = new WebGLRenderTarget(
            textureWidth,
            textureHeight,
            { samples: multisample, type: HalfFloatType }
        );

        const material = new ShaderMaterial({
            name: shader.name !== undefined ? shader.name : "unspecified",
            uniforms: UniformsUtils.clone(shader.uniforms),
            fragmentShader: shader.fragmentShader,
            vertexShader: shader.vertexShader,
            transparent: true,
        });

        material.uniforms["tDiffuse"].value = renderTarget.texture;
        material.uniforms["_opacity"].value = options.opacity || 1;
        material.uniforms["color"].value = color;
        material.uniforms["textureMatrix"].value = textureMatrix;

        this.material = material;
        this.count = 0;
        this.onBeforeRender = (renderer, scene, camera) => {
            this.count++;

            // if (this.count % 4 === 0) {
            //     return;
            // }

            reflectorWorldPosition.setFromMatrixPosition(scope.matrixWorld);
            cameraWorldPosition.setFromMatrixPosition(camera.matrixWorld);

            rotationMatrix.extractRotation(scope.matrixWorld);

            normal.set(0, 0, 1);
            normal.applyMatrix4(rotationMatrix);

            view.subVectors(reflectorWorldPosition, cameraWorldPosition);

            // Avoid rendering when reflector is facing away

            if (view.dot(normal) > 0) return;

            view.reflect(normal).negate();
            view.add(reflectorWorldPosition);

            rotationMatrix.extractRotation(camera.matrixWorld);

            lookAtPosition.set(0, 0, -1);
            lookAtPosition.applyMatrix4(rotationMatrix);
            lookAtPosition.add(cameraWorldPosition);

            target.subVectors(reflectorWorldPosition, lookAtPosition);
            target.reflect(normal).negate();
            target.add(reflectorWorldPosition);

            virtualCamera.position.copy(view);
            virtualCamera.up.set(0, 1, 0);
            virtualCamera.up.applyMatrix4(rotationMatrix);
            virtualCamera.up.reflect(normal);
            virtualCamera.lookAt(target);

            virtualCamera.far = camera.far; // Used in WebGLBackground

            virtualCamera.updateMatrixWorld();
            virtualCamera.projectionMatrix.copy(camera.projectionMatrix);

            // Update the texture matrix
            textureMatrix.set(
                0.5,
                0.0,
                0.0,
                0.5,
                0.0,
                0.5,
                0.0,
                0.5,
                0.0,
                0.0,
                0.5,
                0.5,
                0.0,
                0.0,
                0.0,
                1.0
            );
            textureMatrix.multiply(virtualCamera.projectionMatrix);
            textureMatrix.multiply(virtualCamera.matrixWorldInverse);
            textureMatrix.multiply(scope.matrixWorld);

            // Now update projection matrix with new clip plane, implementing code from: http://www.terathon.com/code/oblique.html
            // Paper explaining this technique: http://www.terathon.com/lengyel/Lengyel-Oblique.pdf
            reflectorPlane.setFromNormalAndCoplanarPoint(
                normal,
                reflectorWorldPosition
            );
            reflectorPlane.applyMatrix4(virtualCamera.matrixWorldInverse);

            clipPlane.set(
                reflectorPlane.normal.x,
                reflectorPlane.normal.y,
                reflectorPlane.normal.z,
                reflectorPlane.constant
            );

            const projectionMatrix = virtualCamera.projectionMatrix;

            q.x =
                (Math.sign(clipPlane.x) + projectionMatrix.elements[8]) /
                projectionMatrix.elements[0];
            q.y =
                (Math.sign(clipPlane.y) + projectionMatrix.elements[9]) /
                projectionMatrix.elements[5];
            q.z = -1.0;
            q.w =
                (1.0 + projectionMatrix.elements[10]) /
                projectionMatrix.elements[14];

            // Calculate the scaled plane vector
            clipPlane.multiplyScalar(2.0 / clipPlane.dot(q));

            // Replacing the third row of the projection matrix
            projectionMatrix.elements[2] = clipPlane.x;
            projectionMatrix.elements[6] = clipPlane.y;
            projectionMatrix.elements[10] = clipPlane.z + 1.0 - clipBias;
            projectionMatrix.elements[14] = clipPlane.w;

            // Render
            scope.visible = false;

            const currentRenderTarget = renderer.getRenderTarget();

            const currentXrEnabled = renderer.xr.enabled;
            const currentShadowAutoUpdate = renderer.shadowMap.autoUpdate;

            renderer.xr.enabled = false; // Avoid camera modification
            renderer.shadowMap.autoUpdate = false; // Avoid re-computing shadows

            renderer.setRenderTarget(renderTarget);

            renderer.state.buffers.depth.setMask(true); // make sure the depth buffer is writable so it can be properly cleared, see #18897

            if (renderer.autoClear === false) renderer.clear();

            // filter

            options.filter.forEach((name) => {
                const mesh = scene.getObjectByName(name);
                mesh.visible = false;
            });

            renderer.render(scene, virtualCamera);

            options.filter.forEach((name) => {
                const mesh = scene.getObjectByName(name);
                mesh.visible = true;
            });

            renderer.xr.enabled = currentXrEnabled;
            renderer.shadowMap.autoUpdate = currentShadowAutoUpdate;

            renderer.setRenderTarget(currentRenderTarget);

            // Restore viewport

            const viewport = camera.viewport;

            if (viewport !== undefined) {
                renderer.state.viewport(viewport);
            }

            scope.visible = true;
        };

        this.getRenderTarget = function () {
            return renderTarget;
        };

        this.dispose = function () {
            renderTarget.dispose();
            scope.material.dispose();
        };
    }
}

Reflector.ReflectorShader = {
    name: "ReflectorShader",

    uniforms: {
        color: {
            value: null,
        },

        tDiffuse: {
            value: null,
        },

        textureMatrix: {
            value: null,
        },
        _opacity: {
            value: null,
        },
    },

    vertexShader: /* glsl */ `
		uniform mat4 textureMatrix;
		varying vec4 vUv;

		#include <common>
		#include <logdepthbuf_pars_vertex>

		void main() {

			vUv = textureMatrix * vec4( position, 1.0 );

			gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );

			#include <logdepthbuf_vertex>

		}`,

    fragmentShader: /* glsl */ `
		uniform vec3 color;
		uniform float _opacity;
		uniform sampler2D tDiffuse;
		varying vec4 vUv;

		#include <logdepthbuf_pars_fragment>

		float blendOverlay( float base, float blend ) {

			return( base < 0.5 ? ( 2.0 * base * blend ) : ( 1.0 - 2.0 * ( 1.0 - base ) * ( 1.0 - blend ) ) );

		}

		vec3 blendOverlay( vec3 base, vec3 blend ) {

			return vec3( blendOverlay( base.r, blend.r ), blendOverlay( base.g, blend.g ), blendOverlay( base.b, blend.b ) );

		}

		void main() {

			#include <logdepthbuf_fragment>

			vec4 base = texture2DProj( tDiffuse, vUv );
			gl_FragColor = vec4( base.rgb , 0.1 );

			#include <tonemapping_fragment>
			#include <colorspace_fragment>

		}`,
};

/**
 * @description: 为mesh的材质增加反光效果
 * @param {*} mesh
 * @return {*}
 */
function addReflectorEffect(mesh, options = { filter: [] }) {
    const material = mesh.material;

    // material.isReflector = true;

    // material.type = "Reflector";
    const camera = new PerspectiveCamera();

    const textureWidth = options.textureWidth || 512;
    const textureHeight = options.textureHeight || 512;
    const clipBias = options.clipBias || 0;
    const shader = options.shader || Reflector.ReflectorShader;
    const multisample =
        options.multisample !== undefined ? options.multisample : 4;

    const reflectorPlane = new Plane();
    const normal = new Vector3();
    const reflectorWorldPosition = new Vector3();
    const cameraWorldPosition = new Vector3();
    const rotationMatrix = new Matrix4();
    const lookAtPosition = new Vector3(0, 0, -1);
    const clipPlane = new Vector4();

    const view = new Vector3();
    const target = new Vector3();
    const q = new Vector4();

    const textureMatrix = new Matrix4();
    const virtualCamera = camera;

    const renderTarget = new WebGLRenderTarget(textureWidth, textureHeight, {
        samples: multisample,
        type: HalfFloatType,
    });

    const appendUniforms = {
        refDiffuse: { value: renderTarget.texture },
        // refOpacity: { value: options.opacity || 1 },
        refTextureMatrix: { value: textureMatrix },
    };

    material.onBeforeCompile = (shader) => {

        Object.assign(shader.uniforms, appendUniforms);
        shader.vertexShader = shader.vertexShader.replace(
            "#include <common>",
            `
            #include <common>
            uniform mat4 refTextureMatrix;
            varying vec4 refUv;
        `
        );
        shader.fragmentShader = shader.fragmentShader.replace(
            "#include <common>",
            `
            #include <common>
            uniform sampler2D refDiffuse;
            varying vec4 refUv;
        `
        );
        shader.vertexShader = shader.vertexShader.replace(
            "#include <begin_vertex>",
            `
            #include <begin_vertex>
            refUv = refTextureMatrix * vec4( position, 1.0 );
        `
        );
        shader.fragmentShader = shader.fragmentShader.replace(
            "#include <dithering_fragment>",
            `
            #include <dithering_fragment>
            
            gl_FragColor.rgb += texture2DProj( refDiffuse, refUv ).rgb;
			gl_FragColor.a =  ${options.opacity || "1.0"};

        `
        );
        // uniform sampler2D refDiffuse;
        // varying vec4 vUv;
        // console.log(shader.fragmentShader);
    };

    mesh.material.onBeforeRender = (renderer, scene, camera) => {
        reflectorWorldPosition.setFromMatrixPosition(mesh.matrixWorld);
        cameraWorldPosition.setFromMatrixPosition(camera.matrixWorld);

        rotationMatrix.extractRotation(mesh.matrixWorld);

        normal.set(0, 0, 1);
        normal.applyMatrix4(rotationMatrix);

        view.subVectors(reflectorWorldPosition, cameraWorldPosition);

        // Avoid rendering when reflector is facing away

        if (view.dot(normal) > 0) return;

        view.reflect(normal).negate();
        view.add(reflectorWorldPosition);

        rotationMatrix.extractRotation(camera.matrixWorld);

        lookAtPosition.set(0, 0, -1);
        lookAtPosition.applyMatrix4(rotationMatrix);
        lookAtPosition.add(cameraWorldPosition);

        target.subVectors(reflectorWorldPosition, lookAtPosition);
        target.reflect(normal).negate();
        target.add(reflectorWorldPosition);

        virtualCamera.position.copy(view);
        virtualCamera.up.set(0, 1, 0);
        virtualCamera.up.applyMatrix4(rotationMatrix);
        virtualCamera.up.reflect(normal);
        virtualCamera.lookAt(target);

        virtualCamera.far = camera.far; // Used in WebGLBackground

        virtualCamera.updateMatrixWorld();
        virtualCamera.projectionMatrix.copy(camera.projectionMatrix);

        // Update the texture matrix
        textureMatrix.set(
            0.5,
            0.0,
            0.0,
            0.5,
            0.0,
            0.5,
            0.0,
            0.5,
            0.0,
            0.0,
            0.5,
            0.5,
            0.0,
            0.0,
            0.0,
            1.0
        );
        textureMatrix.multiply(virtualCamera.projectionMatrix);
        textureMatrix.multiply(virtualCamera.matrixWorldInverse);
        textureMatrix.multiply(mesh.matrixWorld);

        // Now update projection matrix with new clip plane, implementing code from: http://www.terathon.com/code/oblique.html
        // Paper explaining this technique: http://www.terathon.com/lengyel/Lengyel-Oblique.pdf
        reflectorPlane.setFromNormalAndCoplanarPoint(
            normal,
            reflectorWorldPosition
        );
        reflectorPlane.applyMatrix4(virtualCamera.matrixWorldInverse);

        clipPlane.set(
            reflectorPlane.normal.x,
            reflectorPlane.normal.y,
            reflectorPlane.normal.z,
            reflectorPlane.constant
        );

        const projectionMatrix = virtualCamera.projectionMatrix;

        q.x =
            (Math.sign(clipPlane.x) + projectionMatrix.elements[8]) /
            projectionMatrix.elements[0];
        q.y =
            (Math.sign(clipPlane.y) + projectionMatrix.elements[9]) /
            projectionMatrix.elements[5];
        q.z = -1.0;
        q.w =
            (1.0 + projectionMatrix.elements[10]) /
            projectionMatrix.elements[14];

        // Calculate the scaled plane vector
        clipPlane.multiplyScalar(2.0 / clipPlane.dot(q));

        // Replacing the third row of the projection matrix
        projectionMatrix.elements[2] = clipPlane.x;
        projectionMatrix.elements[6] = clipPlane.y;
        projectionMatrix.elements[10] = clipPlane.z + 1.0 - clipBias;
        projectionMatrix.elements[14] = clipPlane.w;

        // Render
        // TODO : 于一体的反光 不能将自己隐去 只是不显示反射纹理
        mesh.visible = false;

        const currentRenderTarget = renderer.getRenderTarget();

        const currentXrEnabled = renderer.xr.enabled;
        const currentShadowAutoUpdate = renderer.shadowMap.autoUpdate;

        renderer.xr.enabled = false; // Avoid camera modification
        renderer.shadowMap.autoUpdate = false; // Avoid re-computing shadows

        renderer.setRenderTarget(renderTarget);

        renderer.state.buffers.depth.setMask(true); // make sure the depth buffer is writable so it can be properly cleared, see #18897

        if (renderer.autoClear === false) renderer.clear();

        // filter

        options.filter.forEach((name) => {
            const mesh = scene.getObjectByName(name);
            mesh.visible = false;
        });

        renderer.render(scene, virtualCamera);

        options.filter.forEach((name) => {
            const mesh = scene.getObjectByName(name);
            mesh.visible = true;
        });

        renderer.xr.enabled = currentXrEnabled;
        renderer.shadowMap.autoUpdate = currentShadowAutoUpdate;

        renderer.setRenderTarget(currentRenderTarget);

        // Restore viewport

        const viewport = camera.viewport;

        if (viewport !== undefined) {
            renderer.state.viewport(viewport);
        }

        mesh.visible = true;
    };
}

new GLTFLoader().load(FILE_HOST + '/files/model/floor.glb', gltf => {

    scene.add(gltf.scene)

    gltf.scene.traverse(child => {

        if (child.isMesh) {

            addReflectorEffect(child, {

                color: 0xffffff,

                clipBias: 0.003,

                filter: []

            })

        }

    })

})
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/expand/modelBlendReflector.js)

## 小结

- 本文提供 **模型反射效果** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

