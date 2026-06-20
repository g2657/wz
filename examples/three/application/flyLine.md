---
title: "Three.js 飞线效果教程"
description: "详解 Three.js 飞线效果：基于 WebGL 实现「飞线效果」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls、THREE.Points 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,飞线效果,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制,粒子特效"
outline: deep
---

### 飞线效果 · *Fly Line* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=flyLine)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![飞线效果](https://z2586300277.github.io/3d-file-server/threeExamples/application/flyLine/colorful.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- THREE.Points 粒子点渲染
- CubicBezierCurve3 三次贝塞尔曲线
- BufferGeometry 自定义顶点/索引数据
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **飞线效果** 效果：基于 WebGL 实现「飞线效果」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls、THREE.Points。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- **THREE.Points** 将每个顶点渲染为可控大小的粒子；可用自定义 attribute（如 `u_index`）驱动片元/顶点动画。
- 曲线类 `getPoints(n)` 将贝塞尔/样条离散为路径点，再写入 BufferGeometry 驱动飞线或路径动画。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
3. 用曲线离散点构建 BufferGeometry，写入自定义 attribute 驱动动画
4. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
5. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import Stats from 'three/examples/jsm/libs/stats.module.js';

var renderer,clock,scene,camera;
    
function initRender() {
    clock = new THREE.Clock();
    renderer = new THREE.WebGLRenderer({antialias: true,alpha:true});
    renderer.setSize(window.innerWidth, window.innerHeight);
    document.body.appendChild(renderer.domElement);
}

function initCamera() {
    camera = new THREE.PerspectiveCamera(85, window.innerWidth / window.innerHeight, 0.1, 1000);
    camera.position.set(-200, 250, 350);
    camera.lookAt(new THREE.Vector3(0, 0, 0));
}

function initScene() {
    scene = new THREE.Scene();

}

function initLight() {
    var hemisphereLight1 = new THREE.HemisphereLight(0xffffff, 0x444444, 2);
    hemisphereLight1.position.set(0, 200, 0);
    scene.add(hemisphereLight1);
}

var linegroup = [];
function addflyline(minx,maxx,colorf,colort){
    var colorf = colorf||{
        r:0.0,
        g:0.0,
        b:0.0
    };
    var colort = colort||{
        r:1.0,
        g:1.0,
        b:1.0
    };
    var curve = new THREE.CubicBezierCurve3(
        new THREE.Vector3( minx, 0, minx ),
        new THREE.Vector3( minx/2, maxx % 70 + 100, maxx/2 ),
        new THREE.Vector3( maxx/2, maxx % 70 + 70, maxx/2 ),
        new THREE.Vector3( maxx, 0, maxx )
    );
    var points = curve.getPoints( (maxx - minx) * 5  );
    var geometry = new THREE.BufferGeometry().setFromPoints( points );
    var material = createMaterial();
    var flyline = new THREE.Points( geometry, material );
    flyline.material.uniforms.time.value = minx;
    flyline.material.uniforms.colorf = {
        type:'v3',
        value:new THREE.Vector3(colorf.r,colorf.g,colorf.b)
    };
    flyline.material.uniforms.colort = {
        type:'v3',
        value:new THREE.Vector3(colort.r,colort.g,colort.b)            
    };

    flyline.minx = minx;
    flyline.maxx = maxx;
    linegroup.push(flyline);
    scene.add(flyline);
}




// 添加地球
var globeMesh;
function addglobe() {
    var axesHelper = new THREE.AxesHelper( 400 );
    scene.add( axesHelper );
    var globeTextureLoader = new THREE.TextureLoader();
    globeTextureLoader.load(FILE_HOST + 'threeExamples/application/flyLine/earth.jpeg', function (texture1) {
        console.log(texture1)
        var globeGgeometry = new THREE.SphereGeometry(60, 100, 100);
        var globeMaterial = new THREE.MeshStandardMaterial({map: texture1});
        globeMesh = new THREE.Mesh(globeGgeometry, globeMaterial);
        scene.add(globeMesh);
    });
}

//创建ShaderMaterial纹理的函数
function createMaterial() {
    var vertShader = `  uniform float time;
    uniform float size;
    varying vec3 iPosition;

    void main(){
        iPosition = vec3(position);
        float pointsize = 1.;
        if(position.x > time && position.x < (time + size)){
            pointsize = (position.x - time) / size;
        }
        gl_PointSize = pointsize * 3.0;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position,1.0);
    }`
    var fragShader = `   uniform float time;
    uniform float size;
    uniform vec3 colorf;
    uniform vec3 colort;

    varying vec3 iPosition;

    void main( void ) {
        float end = time + size;
        vec4 color;
        if(iPosition.x > end || iPosition.x < time){
            discard;
            //color = vec4(0.213,0.424,0.634,0.3);
        }else if(iPosition.x > time && iPosition.x < end){
            float step = fract((iPosition.x - time)/size);

            float dr = abs(colort.x - colorf.x);
            float dg = abs(colort.y - colorf.y);
            float db = abs(colort.z - colorf.z);

            float r = colort.x > colorf.x?(dr*step+colorf.x):(colorf.x -dr*step);
            float g = colort.y > colorf.y?(dg*step+colorf.y):(colorf.y -dg*step);
            float b = colort.z > colorf.z?(db*step+colorf.z):(colorf.z -db*step);

            color = vec4(r,g,b,1.0);
        }
        float d = distance(gl_PointCoord, vec2(0.5, 0.5));
        if(abs(iPosition.x - end) < 0.2 || abs(iPosition.x - time) < 0.2){
            if(d > 0.5){
                discard;
            }
        }
        gl_FragColor = color;
    }`

    //配置着色器里面的attribute变量的值
    var attributes = {};
    //配置着色器里面的uniform变量的值
    var uniforms = {
        time: {type: 'f', value: -70.0},
        size:{type:'f',value:25.0},
    };

    var meshMaterial = new THREE.ShaderMaterial({
        uniforms: uniforms,
        defaultAttributeValues : attributes,
        vertexShader: vertShader,
        fragmentShader: fragShader,
        transparent: true
    });

    return meshMaterial;
}

//初始化性能插件
var stats;
function initStats() {
    stats = new Stats();
    document.body.appendChild(stats.dom);
}
//用户交互插件 鼠标左键按住旋转，右键按住平移，滚轮缩放
var controls;
function initControls() {
    controls = new OrbitControls(camera, renderer.domElement);
    // 如果使用animate方法时，将此函数删除
    //controls.addEventListener( 'change', render );
    // 使动画循环使用时阻尼或自转 意思是否有惯性
    controls.enableDamping = true;
    //动态阻尼系数 就是鼠标拖拽旋转灵敏度
    //controls.dampingFactor = 0.25;
    //是否可以缩放
    controls.enableZoom = true;
    //是否自动旋转
    controls.autoRotate = false;
    controls.autoRotateSpeed = 3;
    //设置相机距离原点的最远距离
    controls.minDistance = 1;
    //设置相机距离原点的最远距离
    controls.maxDistance = 200;
    //是否开启右键拖拽
    controls.enablePan = true;
}
function render() {
    var delta = clock.getDelta();
    if(globeMesh){
        globeMesh.rotation.x += 0.01;
        globeMesh.rotation.y += 0.02;
    }

    if(linegroup.length){
        for(var i = 0;i<linegroup.length;i++){
            var flyline = linegroup[i];
            if(flyline && flyline.material.uniforms){
                var time = flyline.material.uniforms.time.value;
                var size = flyline.material.uniforms.size.value;
                if(time > flyline.maxx){
                    flyline.material.uniforms.time.value = flyline.minx - size;
                }
                flyline.material.uniforms.time.value += 1.0;
            }
        }
    }
    
    renderer.render(scene, camera);
}
function animate() {
    //更新控制器
    render();
    //更新性能插件
    stats.update();
    controls.update();
    requestAnimationFrame(animate);
}

function randomNum(minNum,maxNum){ 
    switch(arguments.length){ 
        case 1: 
            return parseInt(Math.random()*minNum+1,10); 
        break; 
        case 2: 
            return parseInt(Math.random()*(maxNum-minNum+1)+minNum,10); 
        break; 
            default: 
                return 0; 
            break; 
    } 
} 
function draw() {
    initScene();
    initCamera();
    initLight();
    initRender();
    initControls();
    initStats();
    for(var j = 0;j< 200;j++){
        var start = randomNum(-3000,-700)/10;
        var end = randomNum(700,3000)/10;

        var fr = randomNum(100,1000)/1000;
        var fg = randomNum(100,1000)/1000;
        var fb = randomNum(100,1000)/1000;

        var tr = randomNum(100,1000)/1000;
        var tg = randomNum(100,1000)/1000;
        var tb = randomNum(100,1000)/1000;

        
        addflyline(start,end,{r:fr,g:fg,b:fb},{r:tr,g:tg,b:tb});
    }
    addglobe();

    animate();
}
draw();
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/flyLine.js)

## 小结

- 本文提供 **飞线效果** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

