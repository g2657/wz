---
title: "Three.js 圆泡吸附教程"
description: "详解 Three.js 圆泡吸附：基于 WebGL 实现「圆泡吸附」可视化效果，附完整可运行源码，涵盖 GSAP、场景雾效增强纵深 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,圆泡吸附,WebGL,源码,教程,在线案例,GSAP,动画,雾效"
outline: deep
---
### 圆泡吸附 · *Smoke Circle* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=smokeCircle)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![圆泡吸附](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/smokeCircle.jpg)

## 你将学到什么

- GSAP 时间轴与补间动画
- 场景雾效增强纵深
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **圆泡吸附** 效果：基于 WebGL 实现「圆泡吸附」可视化效果，附完整可运行源码；核心用到 GSAP、场景雾效增强纵深。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- 阴影四步：`renderer.shadowMap.enabled`、光源 `castShadow`、物体 `castShadow`、地面 `receiveShadow`。

- **CubeTexture** 六面贴图作 `scene.background`；`scene.environment` 供 PBR 材质反射。

- 时间线库驱动 position/rotation/uniform，与 rAF 渲染循环配合。

## 实现步骤

1. 搭建 Scene / Camera / Renderer 与 OrbitControls
2. rAF 循环中 update 并 render

## 代码要点

```js
import * as THREE from 'three';
import gsap from 'gsap';

// reference Victor Vergara https://codepen.io/vcomics/pen/KBMyjE
window.addEventListener('load', init, false);
function init() {
  createWorld();
  createLights();
  createPrimitive();
  createParticleWord();
  animation();
}
var scene, camera, renderer;
var world = new THREE.Object3D();
var _width, _height;
function createWorld() {
  _width = window.innerWidth;
  _height= window.innerHeight;
  scene = new THREE.Scene();
  scene.fog = new THREE.Fog(0x000000, 4, 12);
  //scene.background = new THREE.Color(0xF00000);
  camera = new THREE.PerspectiveCamera(35, _width/_height, 1, 1000);
  camera.position.set(0,0,8);

  renderer = new THREE.WebGLRenderer({antialias:true, alpha:false});
  renderer.setSize(_width, _height);
  renderer.shadowMap.enabled = true;
  document.body.appendChild(renderer.domElement);

  window.addEventListener('resize', onWindowResize, false);
}
function onWindowResize() {
  _width = window.innerWidth;
  _height = window.innerHeight;
  renderer.setSize(_width, _height);
  camera.aspect = _width / _height;
  camera.updateProjectionMatrix();
  console.log('- resize -');
}
var _ambientLights, _lights;
function createLights() {
  _ambientLights = new THREE.HemisphereLight(0x111111, 0x000000, 5);
  _lights = new THREE.PointLight(0xF00555, 0.5);
  _lights.position.set(0, 0, 3);
  _lights.castShadow = true;
  scene.add(_ambientLights);
  scene.add(_lights);
}
var createParticleWord = function() {
  var geometry = new THREE.IcosahedronGeometry(0.7, 3);
  var circle_start = 10;
  
  for (var i = 0; i<120; i++) {
    var material = new THREE.MeshBasicMaterial( {
      color: 0xFFFF
   } );
    var circle = new THREE.Mesh( geometry, material );
    circle.castShadow  = true;
    circle.receivedShadow = true;
    
    circle.position.x = -Math.random()* circle_start + Math.random()* circle_start;
    circle.position.z = -Math.random()* circle_start + Math.random()* circle_start;
    circle.position.y = -Math.random()* circle_start + Math.random()* circle_start;
    var circle_scale = Math.random()* 1;
    var circle_random = Math.random() * 1;
    circle.scale.set(circle_scale, circle_scale, circle_scale);

    var object_pos = world.children[ 0 ];
    var object_pos_range = 0;
    setInterval(function(){  
      object_pos.position.x = -Math.random() * object_pos_range + Math.random() * object_pos_range;
      object_pos.position.y = -Math.random() * object_pos_range + Math.random() * object_pos_range;
      object_pos.position.z = -Math.random() * object_pos_range + Math.random() * object_pos_range;
    }, 1000);
    world.add( circle );
  }
  scene.add(world);
}
var primitiveElement = function() {
  this.mesh = new THREE.Object3D();
  var geo = new THREE.IcosahedronGeometry();
  var mat = new THREE.MeshBasicMaterial({color:0x500000});
  var mesh = new THREE.Mesh(geo, mat);

  //this.mesh.add(mesh);

}
var _primitive;
function createPrimitive() {
  _primitive = new primitiveElement();
  _primitive.mesh.scale.set(1,1,1);
  scene.add(_primitive.mesh);
}
function createGrid() {
  var gridHelper = new THREE.GridHelper(20, 20);
  gridHelper.position.y = -1;
  scene.add(gridHelper);
}

function animation() {
  var time = Date.now() * 0.003;
  _primitive.mesh.rotation.y += 0.003;
  world.rotation.y = Math.sin(time) * Math.PI / 180;
  world.rotation.z = Math.cos(time) * Math.PI / 180;
  var object_place = world.children[ 0 ];
  object_place.visible = false;

  for ( let i = 0, l = world.children.length; i < l; i ++ ) {
    var object = world.children[ i ];
    var object_left = world.children[ i-1 ];
    if (i>1) {

      gsap.to(object.position, 2, {
        x:Math.cos(object_left.position.x * Math.PI) * 1,
        y:Math.sin(object_left.position.y * Math.PI) * 1,
        z:Math.cos(object_left.position.z * Math.PI) * 1,
      });
    
    }
  }

  var object_speed = 0.6;
  var object_guide = world.children[ 1 ];
  object_guide.position.x += Math.sin(object_place.position.x) - object_guide.position.x * object_speed;
  object_guide.position.y += Math.cos(object_place.position.y) - object_guide.position.y * object_speed;
  object_guide.position.z += object_place.position.z - object_guide.position.z * object_speed;

  camera.lookAt(scene.position);

  requestAnimationFrame(animation);
  renderer.render(scene, camera);
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/smokeCircle.js)

## 小结

- 本文提供 **圆泡吸附** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

