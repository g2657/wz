---
title: "Three.js 粒子火焰教程"
description: "详解 Three.js 粒子火焰：基于 WebGL 实现「粒子火焰」可视化效果，附完整可运行源码，涵盖 OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 粒子 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,粒子火焰,WebGL,源码,教程,在线案例,OrbitControls,相机控制"
outline: deep
---
### 粒子火焰 · *Fire Particles* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=particle&id=fireParticles)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![粒子火焰](https://z2586300277.github.io/three-cesium-examples/threeExamples/particle/fireParticles.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **粒子火焰** 效果：基于 WebGL 实现「粒子火焰」可视化效果，附完整可运行源码；核心用到 OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **OrbitControls** 轨道旋转缩放；开 `enableDamping` 时每帧需 `controls.update()`。

## 实现步骤

1. 搭建 Scene / Camera / Renderer 与 OrbitControls
2. rAF 循环中 update 并 render

## 代码要点

```js
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

var camera, renderer, scene, particleSystem, baseParticle, mouse;
mouse = [window.innerWidth / 2, window.innerHeight / 2];
renderer = new THREE.WebGLRenderer({ antialias: true });
scene = new THREE.Scene();
camera = new THREE.PerspectiveCamera(20, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.z = 50;
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);
new OrbitControls(camera, renderer.domElement);

baseParticle = new THREE.BoxGeometry(0.4, 0.4, 0.4);
baseParticle.rotateZ(Math.PI / 4);

baseParticle = new THREE.Mesh(
	baseParticle,
	new THREE.MeshBasicMaterial({ color: 0xffffff })
);

particleSystem = new ParticleSystem(99);
render();

function randomFloat(a, b) {
	var r = (Math.random() * (b - a) + a);
	return r;
}

function partToHex(part) {
	var h = part.toString(16);
	return h.length == 1 ? "0" + h : h;
}

var color;
function FireParticle() {
	this.direction;
	this.scaleSpeed;
	this.curAge;

	this.parent;

	this.obj;
	this.colorRamp = [[255, 255, 0], [255, 136, 34], [255, 17, 68], [153, 136, 136]];

	this.update = function () {
		if (Math.abs(this.parent.pos.x - this.obj.position.x) > 10 || Math.abs(-this.parent.pos.y - this.obj.position.y) > 10) {
			this.obj.scale.x *= .8;
			this.obj.scale.y *= .8;
			this.obj.scale.z *= .8;
		}

		var point = (this.curAge / 40);
		var pointRem = point % 1;

		if (Math.round(point) >= this.colorRamp.length - 1) {
			color = this.colorRamp[this.colorRamp.length - 1];
		} else {
			color = [Math.floor(this.colorRamp[Math.floor(point)][0] * (1 - pointRem) + this.colorRamp[Math.floor(point) + 1][0] * pointRem), Math.floor(this.colorRamp[Math.floor(point)][1] * (1 - pointRem) + this.colorRamp[Math.floor(point) + 1][1] * pointRem), Math.floor(this.colorRamp[Math.floor(point)][2] * (1 - pointRem) + this.colorRamp[Math.floor(point) + 1][2] * pointRem)];
		}

		color = partToHex(color[0]) + partToHex(color[1]) + partToHex(color[2])
		color = parseInt(color, 16);

		this.obj.material.color.setHex(color);

		this.curAge++;

		if (this.obj.scale.x < .01) {
			this.init();
		}

		this.obj.position.x += this.direction.x;
		this.obj.position.y += this.direction.y;
		this.obj.position.z += this.direction.z;

		this.obj.scale.x *= this.scaleSpeed;
		this.obj.scale.y *= this.scaleSpeed;
		this.obj.scale.z *= this.scaleSpeed;
	}

	this.init = function () {
		this.direction = new THREE.Vector3(randomFloat(-.01, .01), randomFloat(.01, .1), randomFloat(-.01, .01));
		this.scaleSpeed = randomFloat(.8, .99);
		this.curAge = 0;

		if (this.obj != undefined) {
			scene.remove(this.obj);
		}

		this.obj = baseParticle.clone();
		this.obj.position.set(this.parent.obj.position.x + randomFloat(-.2, .2), this.parent.obj.position.y, this.parent.obj.position.z + randomFloat(-.2, .2));
		this.obj.scale.set(1, 2, 1);
		this.obj.material = this.obj.material.clone();

		for (var i = 0; i < randomFloat(0, 100); i++) {
			this.update();
		}

		scene.add(this.obj);
	}




}

function ParticleSystem(size) {
	this.particles = [];
	this.obj = new THREE.Group();

	this.p = new THREE.Vector3();
	this.d;
	this.dis;
	this.pos = new THREE.Vector3(0, 0, 0);

	this.init = function () {
		for (var i = 0; i < size; i++) {
			this.particles.push(new FireParticle());
			this.particles[i].parent = this;
			this.particles[i].init();
		}

		scene.add(this.obj);
	}
	this.init();

	this.update = function () {
		this.p.set((mouse[0] / window.innerWidth) * 2 - 1, (mouse[1] / window.innerHeight) * 2 - 1, .5);
		this.p.unproject(camera);
		this.d = this.p.sub(camera.position).normalize();
		this.dis = -camera.position.z / this.d.z;
		this.pos = camera.position.clone().add(this.d.multiplyScalar(this.dis));

		this.obj.position.x = this.pos.x;
		this.obj.position.y = -this.pos.y;

		for (var i = 0; i < this.particles.length; i++) {
			this.particles[i].update();
		}

		this.obj.rotation.y += .03;
	}
}

function render() {
	requestAnimationFrame(render);
	renderer.render(scene, camera);
	particleSystem.update();
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/particle/fireParticles.js)

## 小结

- 本文提供 **粒子火焰** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

