---
title: "Three.js 魔法阵教程"
description: "详解 Three.js 魔法阵：自定义圆柱几何体, 只有圆柱侧面, 没有顶和底，涵盖 OrbitControls、THREE.Points、BufferGeometry 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,魔法阵,WebGL,源码,教程,在线案例,OrbitControls,相机控制,粒子特效,Points,BufferGeometry"
outline: deep
---

### 魔法阵 · *Magic Circle* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=magicCircle)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![魔法阵](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/magicCircle.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- THREE.Points 粒子点渲染
- BufferGeometry 自定义顶点/索引数据
- 监听窗口 `resize` 同步更新 camera 与 renderer

## 效果说明

本案例演示 **魔法阵** 效果：基于 WebGL 实现「魔法阵」可视化效果，附完整可运行源码；核心用到 OrbitControls、THREE.Points、BufferGeometry。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- **THREE.Points** 将每个顶点渲染为可控大小的粒子；可用自定义 attribute（如 `u_index`）驱动片元/顶点动画。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
3. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import Stats from 'three/examples/jsm/libs/stats.module.js';

class ThreeCore {
    dom;
    scene;
    camera;
    defaultCamera;
    renderer;
    clock;
    options;
    stats;
    // 要执行动画的对象集合, 子类可以把自己的动画写进 onRender 也可以 this.addAnimate() 添加到父类动画集合里
    animates;
    constructor(dom, options) {
        this.dom = dom;
        this.options = options;
        this.scene = new THREE.Scene();
        this.clock = new THREE.Clock();
        this.animates = {};
        const k = dom.clientWidth / dom.clientHeight;
        if ("fov" in options.cameraOptions) {
            this.defaultCamera = new THREE.PerspectiveCamera(options.cameraOptions.fov, k, options.cameraOptions.near, options.cameraOptions.far);
        }
        else {
            const s = options.cameraOptions.s;
            this.defaultCamera = new THREE.OrthographicCamera(-s * k, s * k, s, -s, options.cameraOptions.near, options.cameraOptions.far);
        }
        this.camera = this.defaultCamera;
        this.scene.add(this.camera);
        const rendererOptions = {
            // 抗锯齿
            antialias: true,
            alpha: true,
            // 深度缓冲, 解决模型重叠部分不停闪烁问题
            // 这个属性会导致精灵材质会被后面的物体遮挡
            // 只能出现问题的时候, 在那个场景 new ThreeCore继承类的时候, 传入rendererOptions参数, 将此参数改为 false
            logarithmicDepthBuffer: true
        };
        const renderer = new THREE.WebGLRenderer(Object.assign({}, rendererOptions, options.rendererOptions));
        renderer.setSize(dom.clientWidth, dom.clientHeight);
        renderer.setPixelRatio(window.devicePixelRatio);
        dom.appendChild(renderer.domElement);
        this.renderer = renderer;
        this.stats = new Stats();
        dom.appendChild(this.stats.dom);
        window.addEventListener("resize", this.onResize, false);
        //利用 setTimeout 宏任务最后执行特性, 使js执行过程要等所有微任务和同步代码执行完再执行, 否则 this.init() 可能会在场景未搭建完毕就执行报错或没有生产对象

        this.init();
        this.animate();

    }
    // 提供给子类覆写
    onRenderer() { }
    // 提供给子类覆写
    onDestroy() { }
    animate() {
        this.renderer.setAnimationLoop(() => {
            this.onRenderer();
            this.stats.update();
            // 执行动画
            for (const key in this.animates) {
                this.animates[key](this.clock.getDelta());
            }
            this.renderer.render(this.scene, this.camera);
        });
    }
    addAnimate(name, func) {
        this.animates[name] = func;
    }
    removeAnimate(name) {
        delete this.animates[name];
    }
    onResize = () => {
        const width = this.dom.clientWidth;
        const height = this.dom.clientHeight;
        const k = width / height;
        // 更新相机
        if (this.camera instanceof THREE.PerspectiveCamera) {
            this.camera.aspect = k;
        }
        else {
            const s = this.options.cameraOptions.s;
            this.camera.left = -s * k;
            this.camera.right = s * k;
            this.camera.top = s;
            this.camera.bottom = -s;
        }
        this.camera.updateProjectionMatrix();
        // 更新renderer
        this.renderer.setSize(width, height);
        this.renderer.setPixelRatio(window.devicePixelRatio);
    };
    destroyed() {
        // 需要手动移除掉 gui, 否则刷新页面时会出现多个gui
        document.querySelector(".dg.main.a")?.remove();
        window.removeEventListener("resize", this.onResize.bind(this));
        this.onDestroy();
        this.renderer.setAnimationLoop(null);
        this.renderer.renderLists.dispose();
        this.renderer.dispose();
        this.renderer.forceContextLoss();
        this.renderer.domElement.innerHTML = "";
        this.scene.clear();
        THREE.Cache.clear();
    }
}

class ThreeProject extends ThreeCore {
    orbit;
    // 法阵高度
    height = 2;
    // 圆形底
    circle = [];
    circleRadius = 1;
    circleRotateSpeed = 0.02;
    // 两个旋转光环
    ring1 = [];
    ring2 = [];
    ringRadius = 0.5;
    ringScaleOffset = 0.01;
    ringRotateSpeed = 0.01;
    // 粒子
    particles = [];
    particlesMinSize = 0.04;
    particlesMaxSize = 0.15;
    particlesRangeRadius = 0.8;
    particlesFloatSpeed = 0.01;
    constructor(dom) {
        super(dom, {
            cameraOptions: {
                fov: 45,
                near: 0.001,
                far: 100
            }
        });
        this.camera.position.set(0, 2, 4);
        this.scene.background = new THREE.Color(0x000000);
        this.renderer.shadowMap.enabled = true;
        this.orbit = new OrbitControls(this.camera, this.renderer.domElement);
        this.orbit.minDistance = 0.001;
        this.orbit.target.y = 1;
        // const axesHelper = new THREE.AxesHelper(10)
        // this.scene.add(axesHelper)
        const gridHelper = new THREE.GridHelper(2);
        this.scene.add(gridHelper);
        const ambientLight = new THREE.AmbientLight(0xffffff, 1);
        this.scene.add(ambientLight);
        const directionalLight = new THREE.DirectionalLight(0xffffff, 2);
        directionalLight.position.set(60, 30, -30);
        directionalLight.castShadow = true;
        this.scene.add(directionalLight);
        const directionalLight2 = new THREE.DirectionalLight(0xffffff, 4);
        directionalLight2.position.set(60, 30, 30);
        directionalLight2.castShadow = true;
        this.scene.add(directionalLight2);
        const _url = FILE_HOST + 'threeExamples/application/magicCircle/';
        const point1Texture = new THREE.TextureLoader().load(_url + 'point1.png');
        const point2Texture = new THREE.TextureLoader().load(_url + 'point2.png');
        const point3Texture = new THREE.TextureLoader().load(_url + 'point3.png');
        const point4Texture = new THREE.TextureLoader().load(_url + 'point4.png');
        const magicTexture = new THREE.TextureLoader().load(_url + 'magic.png');
        const guangyunTexture = new THREE.TextureLoader().load(_url + 'guangyun.png');
        const pointTextures = [
            point1Texture,
            point2Texture,
            point3Texture,
            point4Texture,
        ];
        const { height, circleRadius, ringRadius, ringScaleOffset, particlesMaxSize, particlesMinSize, particlesRangeRadius } = this;
        const group = new THREE.Group();
        const circleGeo = new THREE.CircleGeometry(circleRadius, 64);
        const circleMat = new THREE.MeshBasicMaterial({
            map: magicTexture,
            transparent: true,
            side: THREE.DoubleSide,
            depthWrite: false
        });
        const circle = new THREE.Mesh(circleGeo, circleMat);
        circle.rotateX(-Math.PI / 2);
        this.circle.push(circle);
        group.add(circle);
        const ringGeo = this.getCylinderGeo(ringRadius, height);
        const ringMat = new THREE.MeshBasicMaterial({
            map: guangyunTexture,
            transparent: true,
            side: THREE.DoubleSide,
            wireframe: false,
            depthWrite: false
        });
        const ring1 = new THREE.Mesh(ringGeo, ringMat);
        this.ring1.push(ring1);
        group.add(ring1);
        const ring2 = ring1.clone();
        ring2.userData.ringScaleOffset = ringScaleOffset;
        group.add(ring2);
        this.ring2.push(ring2);
        // 有几种粒子材质图片, 每种图片做一次点云
        for (let j = 0; j < 10; j++) {
            for (let i = 0; i < pointTextures.length; i++) {
                const particles = this.getParticles(particlesRangeRadius, height, pointTextures[i], particlesMinSize, particlesMaxSize, this.particlesFloatSpeed);
                this.particles.push(particles);
                group.add(particles);
            }
        }
        this.scene.add(group);
    }
    getParticles(radius, height, texture, pointMinSize, pointMaxSize, pointFloatSpeed) {
        const geometry = new THREE.BufferGeometry();
        geometry.setAttribute('position', new THREE.BufferAttribute(new Float32Array([0, 0, 0]), 3));
        const material = new THREE.PointsMaterial({
            size: Math.random() * (pointMaxSize - pointMinSize) + pointMinSize,
            map: texture,
            blending: THREE.AdditiveBlending,
            depthTest: false,
            transparent: true,
            opacity: 0.2 + Math.random() * 0.8
        });
        const particles = new THREE.Points(geometry, material);
        particles.userData.floatSpeed = 0.001 + Math.random() * pointFloatSpeed;
        particles.userData.radius = radius;
        particles.position.x = Math.random() * radius * 2 - radius;
        particles.position.y = Math.random() * height;
        particles.position.z = Math.random() * radius * 2 - radius;
        return particles;
    }
    /**
     * 自定义圆柱几何体, 只有圆柱侧面, 没有顶和底
     */
    getCylinderGeo(radius = 1, height = 1, segment = 64) {
        const bottomPos = [];
        const topPos = [];
        const bottomUvs = [];
        const topUvs = [];
        const angleOffset = (Math.PI * 2) / segment;
        const uvOffset = 1 / (segment - 1);
        for (let i = 0; i < segment; i++) {
            const x = Math.cos(angleOffset * i) * radius;
            const z = Math.sin(angleOffset * i) * radius;
            bottomPos.push(x, 0, z);
            topPos.push(x, height, z);
            bottomUvs.push(i * uvOffset, 0);
            topUvs.push(i * uvOffset, 1);
        }
        const positions = bottomPos.concat(topPos);
        const uvs = bottomUvs.concat(topUvs);
        const index = [];
        for (let i = 0; i < segment; i++) {
            if (i != segment - 1) {
                index.push(i + segment + 1, i, i + segment);
                index.push(i, i + segment + 1, i + 1);
            }
            else {
                index.push(segment, i, i + segment);
                index.push(i, segment, 0);
            }
        }
        const geo = new THREE.BufferGeometry();
        geo.setAttribute('position', new THREE.BufferAttribute(new Float32Array(positions), 3));
        geo.setAttribute('uv', new THREE.BufferAttribute(new Float32Array(uvs), 2));
        geo.setIndex(new THREE.BufferAttribute(new Uint16Array(index), 1));
        return geo;
    }
    init() {
    }
    onRenderer() {
        this.orbit.update();
        this.updateCircle();
        this.updateRing();
        this.updatePartical();
    }
    updateCircle() {
        for (let i = 0; i < this.circle.length; i++) {
            this.circle[i].rotateZ(this.circleRotateSpeed);
        }
    }
    updateRing() {
        for (let i = 0; i < this.ring1.length; i++) {
            this.ring1[i].rotateY(this.ringRotateSpeed);
        }
        for (let i = 0; i < this.ring2.length; i++) {
            this.ring2[i].rotateY(-this.ringRotateSpeed);
            if (this.ring2[i].scale.x < 0.9 || this.ring2[i].scale.x > 1.4) {
                this.ring2[i].userData.ringScaleOffset *= -1;
            }
            this.ring2[i].scale.x -= this.ring2[i].userData.ringScaleOffset;
            this.ring2[i].scale.z -= this.ring2[i].userData.ringScaleOffset;
        }
    }
    updatePartical() {
        for (let i = 0; i < this.particles.length; i++) {
            this.particles[i].position.y += this.particles[i].userData.floatSpeed;
            if (this.particles[i].position.y >= this.height) {
                this.particles[i].position.y = 0;
                this.particles[i].position.x = Math.random() * this.particles[i].userData.radius * 2 - this.particles[i].userData.radius;
                this.particles[i].position.z = Math.random() * this.particles[i].userData.radius * 2 - this.particles[i].userData.radius;
                this.particles[i].userData.floatSpeed = 0.001 + Math.random() * this.particlesFloatSpeed;
            }
        }
    }
}

new ThreeProject(document.getElementById("box"));
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/magicCircle.js)

## 小结

- 本文提供 **魔法阵** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

