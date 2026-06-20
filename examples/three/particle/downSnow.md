---
title: "Three.js 下雪教程"
description: "详解 Three.js 下雪：基于 WebGL 实现「下雪」可视化效果，附完整可运行源码，涵盖 onBeforeCompile、OrbitControls、THREE.Points 等关键实现，附完整源码与在线 Demo，适合 Three.js 粒子 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,下雪,WebGL,源码,教程,在线案例,onBeforeCompile,shader注入,OrbitControls,相机控制,粒子特效,Points"
outline: deep
---
### 下雪 · *Snow* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=particle&id=downSnow)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![下雪](https://z2586300277.github.io/three-cesium-examples/threeExamples/particle/downSnow.jpg)

## 你将学到什么

- 自定义 ShaderMaterial / 修改内置 shader
- 相机交互控制器
- 点云 / 粒子 / 实例化渲染
- requestAnimationFrame 渲染循环

## 效果说明

本案例演示 **下雪** 效果：基于 WebGL 实现「下雪」可视化效果，附完整可运行源码；核心用到 onBeforeCompile、OrbitControls、THREE.Points。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **ShaderMaterial** 完全自定义 GLSL；`onBeforeCompile` 可在内置材质 shader 中注入代码。关注 `uniforms` 与 rAF 更新。

- **OrbitControls** 轨道旋转缩放；开 `enableDamping` 时每帧需 `controls.update()`。

- **Points** 大量顶点用点精灵渲染；**InstancedMesh** 相同几何体批量绘制，降低 draw call。

## 实现步骤

1. 搭建 Scene / Camera / Renderer 与 OrbitControls
2. 定义材质/shader 与 uniforms，rAF 中更新
3. rAF 循环中 update 并 render

## 代码要点

```js
import * as THREE from "three";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
let renderer, scene, camera, stats, controls;

let mesh, uniforms, geometry;
let snowMaterial;
const particles = 1000;

init();
animate();

function init() {
	camera = new THREE.PerspectiveCamera(
		40,
		window.innerWidth / window.innerHeight,
		1,
		10000
	);


	scene = new THREE.Scene();

	snowMaterial = new THREE.PointsMaterial({
		size: 5,
		blending: THREE.AdditiveBlending,
		transparent: true,
	});

	geometry = new THREE.BufferGeometry();

	let positions = new Float32Array(particles * 3);

	for (let i = 0; i < particles * 3; i += 3) {
		positions[i] = Math.random() * 200 - 100;
		positions[i + 1] = Math.random() * 200 - 100;
		positions[i + 2] = Math.random() * 200 - 100;
	}
	geometry.setAttribute(
		"position",
		new THREE.Float32BufferAttribute(positions, 3)
	);

	snowMaterial.onBeforeCompile = (shader) => {
		shader.uniforms.uColor = {
			value: new THREE.Color(0xffffff),
		};

		shader.fragmentShader = shader.fragmentShader.replace(
			`#include <common>`,
			`
                        #include <common>
                            uniform vec3 uColor;
                        `
		);

		shader.fragmentShader = shader.fragmentShader.replace(
			`#include <premultiplied_alpha_fragment>`,
			`
                            #include <premultiplied_alpha_fragment>
                                float distanceLen = distance(gl_PointCoord,vec2(0.5));
                                distanceLen = 1.0 - distanceLen;
                                distanceLen = pow(distanceLen,10.0);
                                vec4 color = vec4(uColor,distanceLen);
                                // if(color.a<0.1)
                                //     discard;
                                gl_FragColor = color;
                            `
		);
	};

	mesh = new THREE.Points(geometry, snowMaterial);
	scene.add(mesh);

	renderer = new THREE.WebGLRenderer();
	renderer.setPixelRatio(window.devicePixelRatio);
	renderer.setSize(window.innerWidth, window.innerHeight);

	controls = new OrbitControls(camera, renderer.domElement);

	const container = document.getElementById("box");
	container.appendChild(renderer.domElement);

	window.addEventListener("resize", onWindowResize);
}

function onWindowResize() {
	camera.aspect = window.innerWidth / window.innerHeight;
	camera.updateProjectionMatrix();

	renderer.setSize(window.innerWidth, window.innerHeight);
}

function animate() {
	requestAnimationFrame(animate);

	render();
	controls.update();
}

function update(time, position) {
	if (snowMaterial) {
		const positions = mesh.geometry.getAttribute("position").array;
		for (let i = 0; i < positions.length; i += 3) {
			positions[i + 1] -= 0.1;
			positions[i] -= Math.sin(i) * 0.1;
			positions[i + 2] -= Math.sin(i) * 0.1;
			if (positions[i + 1] < -100) {
				positions[i + 1] = 100;
			}
		}

		mesh.geometry.getAttribute("position").needsUpdate = true;
	}
}

function render() {
	const time = Date.now() * 0.005;
	update();
	// mesh.rotation.z = 0.01 * time;

	renderer.render(scene, camera);
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/particle/downSnow.js)

## 小结

- 本文提供 **下雪** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

