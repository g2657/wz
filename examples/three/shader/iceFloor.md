---
title: "Three.js 冰面教程"
description: "详解 Three.js 冰面：基于 WebGL 实现「冰面」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,冰面,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 冰面 · *Ice Floor* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=iceFloor)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![冰面](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/iceFloor.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **冰面** 效果：基于 WebGL 实现「冰面」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
3. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
4. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 100000)

camera.position.set(5, 5, 5)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)
controls.enableDamping = true

// https://github.com/rock-biter/ice-trails
const textureLoader = new THREE.TextureLoader()
const crackMap = textureLoader.load(FILE_HOST + 'images/shader/cracks.png')
crackMap.wrapS = THREE.RepeatWrapping
crackMap.wrapT = THREE.RepeatWrapping
const perlinMap = textureLoader.load(FILE_HOST + 'images/shader/smokeMap.png')
perlinMap.wrapS = THREE.RepeatWrapping
perlinMap.wrapT = THREE.RepeatWrapping
const groundMaterial = new THREE.ShaderMaterial({
	vertexShader: `
		uniform float uParallaxDistance;

		varying vec2 vParallax;
		varying vec2 vUv;

		void main() {

		vUv = uv;

		vec3 pos = position;
		vec4 wPos = modelMatrix * vec4(pos, 1.0);

		mat3 tbn = mat3(vec3(1.,0,0), vec3(0,0.,-1.), vec3(0.,1.,0.));
		tbn = transpose(tbn);

		vec3 viewDir = normalize(wPos.xyz - cameraPosition);
		vec3 tbnViewDir = tbn * viewDir;

		vParallax = tbnViewDir.xy;
		vParallax *= uParallaxDistance / dot(-tbnViewDir, vec3(0.0,0.0,1.0));

		gl_Position = projectionMatrix * viewMatrix * wPos;

	}`,
	fragmentShader: `uniform sampler2D uCracksMap;
		uniform sampler2D uTrailMap;
		uniform sampler2D uPerlin;

		varying vec2 vParallax;
		varying vec2 vUv;

		void main() {

		float perlin = texture(uPerlin, vUv).r;
		float perlin2 = texture(uPerlin, vUv * 10.).r;
		vec3 trail = texture(uTrailMap, vUv).rgb;
		float cracks = texture(uCracksMap, vUv * 4.).r;
		float nomalization = 1.0;

		vec3 colorBlue = vec3(0.0,0.2,0.25);
		vec3 colorDeepBlue = vec3(0.0,0.01,0.03);
		vec3 colorGreen = vec3(0.1,0.2,0.35);

		float accumulateFrosted = 0.;

		for (int i = 0; i < 50; i++) {
			float aplitude = float(70 - i) / 1.;
			vec2 uv = vUv * 4. + vParallax * 0.002 * float(i + 1);

			float currCrack = (1. - texture(uCracksMap, uv ).r) * aplitude;

			float currTrail = texture(uTrailMap, vUv + vParallax * 0.0025 * float(i + 1)).r;

			currCrack = currCrack * step(0.7, 1. - pow(currTrail,0.7));

			cracks += currCrack;
			nomalization += aplitude;

			accumulateFrosted += currTrail * aplitude;
		}
		cracks /= nomalization;
		accumulateFrosted /= nomalization;
		cracks += pow(1. - texture(uCracksMap, vUv * 4.).r, 3.) * 3. * step(0.92, 1. - pow(trail.r,0.6));
		
		vec3 cracksParallax = texture(uCracksMap, vUv * 2. + vParallax * 0.1).rgb;
		
		// color += 1. - cracks;
		// color += 1.0 - cracksParallax;

		vec3 frosted = colorBlue * 3. + perlin * 0.6 + perlin2 * 0.6;
		vec3 cracksColor = mix(colorBlue, colorGreen, pow(cracks,1.) * 1.);
		cracksColor += pow(cracks,1.) * 2.;
		cracksColor *= perlin * 8. * colorBlue;
		cracksColor += pow(cracks,1.) * 0.5;
		// cracksColor *= perlin2 * 4.;

		vec3 prxCracksColor = mix(colorDeepBlue, colorBlue, pow(1. - cracksParallax.r,3.) * 10.);
		prxCracksColor *= perlin;
		
		cracksColor = mix(cracksColor, prxCracksColor, 0.3);

		// vec3 color = mix(cracksColor, frosted, pow(trail.r,0.5));
		// cracksColor = mix(cracksColor, vec3(0.1,0.7,0.7), pow(accumulateFrosted,1.5));
		vec3 deepColor = mix(vec3(0.1,0.7,0.7),vec3(0., 0.3, 1.), 1. - pow(accumulateFrosted,1.5));
		cracksColor = mix(cracksColor, deepColor, pow(accumulateFrosted,1.5));
		vec3 color = mix(cracksColor, frosted, pow(trail.r,0.5) );
		// color = mix( color, colorBlue * frosted, pow(trail.r,3.));

		vec2 uv = vUv - 0.5;
		uv *= 2.0;
		color = mix(color, vec3(0.0, 0.01, 0.02), smoothstep(0.2,1.,length(pow(abs(uv), vec2(1.)))));

		// 添加边缘透明度处理，剔除边缘纯色部分
		float edgeDistance = length(uv);
		float alpha = 1.0 - smoothstep(0.8, 1.0, edgeDistance);

		gl_FragColor = vec4(color, alpha);

		#include <tonemapping_fragment>
		#include <colorspace_fragment>
		}`,
	transparent: true,
	side: THREE.DoubleSide,
	uniforms: {
		uTrailMap: new THREE.Uniform(),
		uCracksMap: new THREE.Uniform(crackMap),
		uPerlin: new THREE.Uniform(perlinMap),
		uParallaxDistance: new THREE.Uniform(1),
	},
})

const groundGeometry = new THREE.PlaneGeometry(15, 15)
groundGeometry.rotateX(-Math.PI * 0.5)
const ground = new THREE.Mesh(groundGeometry, groundMaterial)
scene.add(ground)


animate()

function animate() {

	controls.update()
    requestAnimationFrame(animate)
    renderer.render(scene, camera)

}

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)
    camera.aspect = box.clientWidth / box.clientHeight
    camera.updateProjectionMatrix()

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/iceFloor.js)

## 小结

- 本文提供 **冰面** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

