---
title: "Three.js 骨骼动画教程"
description: "详解 Three.js 骨骼动画：基于 WebGL 实现「骨骼动画」可视化效果，附完整可运行源码，涵盖 OrbitControls、骨骼动画与 等关键实现，附完整源码与在线 Demo，适合 Three.js 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,骨骼动画,WebGL,源码,教程,在线案例,OrbitControls,相机控制"
outline: deep
---

### 骨骼动画 · *Skeleton Bone* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=basic&id=skeletonBone)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

## 你将学到什么

- **Bone → Skeleton → SkinnedMesh** 底层结构（非 glTF 导入）
- **skinIndex / skinWeight** 顶点绑骨
- **SkeletonHelper** 显示骨骼线框

## 效果说明

Three.js 官方骨骼示例简化版：手工创建骨骼层级与蒙皮网格，GUI 可开关 **animateBones** 旋转关节。

## 核心概念

```
Bone (parent-child 链)
  ↓
Skeleton(bones, boneInverses)
  ↓
SkinnedMesh(geometry, material) + bind(skeleton)
  ↓
geometry.attributes.skinIndex / skinWeight
```

理解此结构后，glTF 骨骼动画（[modelAnimation](/examples/three/basic/modelAnimation)）的 `AnimationMixer` 只是在驱动这些 Bone 的 matrix。

## 代码要点

```js
import {
    Bone,
    Color,
    CylinderGeometry,
    BoxGeometry,
    DirectionalLight,
    DoubleSide,
    Float32BufferAttribute,
    MeshPhongMaterial,
    PerspectiveCamera,
    Scene,
    SkinnedMesh,
    Skeleton,
    SkeletonHelper,
    Vector3,
    Uint16BufferAttribute,
    WebGLRenderer,
} from "three";

import { GUI } from "three/addons/libs/lil-gui.module.min.js";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";

let gui, scene, camera, renderer, orbit, lights, mesh, bones, skeletonHelper;

const state = {
    animateBones: false,
};

function initScene() {
    gui = new GUI();

    scene = new Scene();
    scene.background = new Color(0x444444);

    camera = new PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 200);
    camera.position.z = 30;
    camera.position.y = 30;

    renderer = new WebGLRenderer({ antialias: true });
    renderer.setPixelRatio(window.devicePixelRatio);
    renderer.setSize(window.innerWidth, window.innerHeight);
    document.body.appendChild(renderer.domElement);

    orbit = new OrbitControls(camera, renderer.domElement);
    orbit.enableZoom = false;

    lights = [];
    lights[0] = new DirectionalLight(0xffffff, 3);
    lights[1] = new DirectionalLight(0xffffff, 3);
    lights[2] = new DirectionalLight(0xffffff, 3);

    lights[0].position.set(0, 200, 0);
    lights[1].position.set(100, 200, 100);
    lights[2].position.set(-100, -200, -100);

    scene.add(lights[0]);
    scene.add(lights[1]);
    scene.add(lights[2]);

    window.addEventListener(
        "resize",
        function () {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();

            renderer.setSize(window.innerWidth, window.innerHeight);
        },
        false
    );

    initBones();
    setupDatGui();
}

function createGeometry(sizing) {
    // const geometry = new CylinderGeometry(
    // 	5, // radiusTop
    // 	5, // radiusBottom
    // 	sizing.height, // height
    // 	8, // radiusSegments
    // 	sizing.segmentCount * 3, // heightSegments
    // 	true // openEnded
    // );

    const geometry = new BoxGeometry(5,sizing.height,5,1,sizing.segmentCount*10)

    const position = geometry.attributes.position;

    const vertex = new Vector3();

    const skinIndices = [];
    const skinWeights = [];

    for (let i = 0; i < position.count; i++) {
        vertex.fromBufferAttribute(position, i);

        const y = vertex.y + sizing.halfHeight;

        const skinIndex = Math.floor(y / sizing.segmentHeight);
        const skinWeight = (y % sizing.segmentHeight) / sizing.segmentHeight;

        skinIndices.push(skinIndex, skinIndex + 1, 0, 0);
        skinWeights.push(1 - skinWeight, skinWeight, 0, 0);
    }

    geometry.setAttribute("skinIndex", new Uint16BufferAttribute(skinIndices, 4));
    geometry.setAttribute("skinWeight", new Float32BufferAttribute(skinWeights, 4));

    return geometry;
}

function createBones(sizing) {
    bones = [];

    let prevBone = new Bone();
    bones.push(prevBone);
    prevBone.position.y = -sizing.halfHeight;

    for (let i = 0; i < sizing.segmentCount; i++) {
        const bone = new Bone();
        bone.position.y = sizing.segmentHeight;
        bones.push(bone);
        prevBone.add(bone);
        prevBone = bone;
    }

    return bones;
}

function createMesh(geometry, bones) {
    const material = new MeshPhongMaterial({
        color: 0x156289,
        emissive: 0x072534,
        side: DoubleSide,
        flatShading: true,
    });

    const mesh = new SkinnedMesh(geometry, material);
    const skeleton = new Skeleton(bones);

    mesh.add(bones[0]);

    mesh.bind(skeleton);

    skeletonHelper = new SkeletonHelper(mesh);
    skeletonHelper.material.linewidth = 2;
    scene.add(skeletonHelper);

    return mesh;
}

function setupDatGui() {
    let folder = gui.addFolder("General Options");

    folder.add(state, "animateBones");
    folder.controllers[0].name("Animate Bones");

    folder.add(mesh, "pose");
    folder.controllers[1].name(".pose()");

    const bones = mesh.skeleton.bones;

    for ( let i = 0; i < bones.length; i ++ ) {

        const bone = bones[ i ];

        folder = gui.addFolder( 'Bone ' + i );

        folder.add( bone.position, 'x', - 10 + bone.position.x, 10 + bone.position.x );
        folder.add( bone.position, 'y', - 10 + bone.position.y, 10 + bone.position.y );
        folder.add( bone.position, 'z', - 10 + bone.position.z, 10 + bone.position.z );

        folder.add( bone.rotation, 'x', - Math.PI * 0.5, Math.PI * 0.5 );
        folder.add( bone.rotation, 'y', - Math.PI * 0.5, Math.PI * 0.5 );
        folder.add( bone.rotation, 'z', - Math.PI * 0.5, Math.PI * 0.5 );

        folder.add( bone.scale, 'x', 0, 2 );
        folder.add( bone.scale, 'y', 0, 2 );
        folder.add( bone.scale, 'z', 0, 2 );

        folder.controllers[ 0 ].name( 'position.x' );
        folder.controllers[ 1 ].name( 'position.y' );
        folder.controllers[ 2 ].name( 'position.z' );

        folder.controllers[ 3 ].name( 'rotation.x' );
        folder.controllers[ 4 ].name( 'rotation.y' );
        folder.controllers[ 5 ].name( 'rotation.z' );

        folder.controllers[ 6 ].name( 'scale.x' );
        folder.controllers[ 7 ].name( 'scale.y' );
        folder.controllers[ 8 ].name( 'scale.z' );

    }
}

function initBones() {
    const segmentHeight = 8;
    const segmentCount = 4;
    const height = segmentHeight * segmentCount;
    const halfHeight = height * 0.5;

    const sizing = {
        segmentHeight,
        segmentCount,
        height,
        halfHeight,
    };

    const geometry = createGeometry(sizing);
    const bones = createBones(sizing);
    mesh = createMesh(geometry, bones);

    mesh.scale.multiplyScalar(1);
    scene.add(mesh);
}

function render() {
    requestAnimationFrame(render);

    const time = Date.now() * 0.001;

    //Wiggle the bones
    if (state.animateBones) {
        for (let i = 0; i < mesh.skeleton.bones.length; i++) {
            mesh.skeleton.bones[i].rotation.z = (Math.sin(time) * 2) / mesh.skeleton.bones.length;
        }
    }

    renderer.render(scene, camera);
}

initScene();
render();
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/basic/skeletonBone.js)

## 小结
