---
title: "Three.js 红玫瑰教程"
description: "详解 Three.js 红玫瑰：基于 WebGL 实现「红玫瑰」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,红玫瑰,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 红玫瑰 · *Red Rose* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=redRose)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![红玫瑰](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/redRose.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **红玫瑰** 效果：基于 WebGL 实现「红玫瑰」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。

## 实现步骤

1. 搭建灯光与环境（如有）
2. requestAnimationFrame 循环 update + render

## 代码要点

```js
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";

const flowerVs=`

uniform float uTime;
varying vec2 vUv;
void main(){
    gl_Position=vec4(position,1.);
    vUv=uv;
}
`;
const flowerFs=`


varying vec2 vUv;
uniform float uTime;
uniform float uAspectRatio;

float flower(vec3 p,  float r)
 {     
     vec3 n=normalize(p);
     
     float q=length(p);
     
     float rho=atan(length(vec2(n.x,n.z)),n.y)*15.0+q*10.01-uTime*4.;//vertical part of  cartesian to polar with some q warp

     float theta=atan(n.x,n.z)*5.0+p.y*3.0+rho*2.0-uTime ;//horizontal part plus some warp by z(bend up) and by rho(twist)
 
     return length(p) -(r+sin(theta)*0.5*(1.5-abs(dot(n,vec3(0,1,0)) )) //the 1-abs(dot()) is limiting the warp effect at poles
                        +sin(rho)*0.3  *(1.5-abs(dot(n,vec3(0,1,0)) )) );// 1.3-abs(dot()means putting some back in 
 }

vec2 map( in vec3 pos )
{
      
    return vec2( flower(pos, 0.750), 5.1 + (sin(uTime)/2.)) ;
    
}

vec2 castRay( in vec3 ro, in vec3 rd )
{
    float tmin = 1.0;
    float tmax = 20.0;
    
#if 0
    float tp1 = (0.0-ro.y)/rd.y; if( tp1>0.0 ) tmax = min( tmax, tp1 );
    float tp2 = (1.6-ro.y)/rd.y; if( tp2>0.0 ) { if( ro.y>1.6 ) tmin = max( tmin, tp2 );
                                                 else           tmax = min( tmax, tp2 ); }
#endif
    
	float precis = 0.01;
    float t = tmin;
    float m = -1.0;
    for( int i=0; i<400; i++ )
    {
	    vec2 res = map( ro+rd*t );
        if( res.x<precis || t>tmax ) break;
        t += res.x*0.05;
	    m = res.y;
    }

    if( t>tmax ) m=-1.0;
    return vec2( t, m );
}

vec3 calcNormal( in vec3 pos )
{
	vec3 eps = vec3( 0.001, 0.0, 0.0 );
	vec3 nor = vec3(
	    map(pos+eps.xyy).x - map(pos-eps.xyy).x,
	    map(pos+eps.yxy).x - map(pos-eps.yxy).x,
	    map(pos+eps.yyx).x - map(pos-eps.yyx).x );
	return normalize(nor);
}

float calcAO( in vec3 pos, in vec3 nor )
{
	float occ = 0.0;
    float sca = 1.0;
    for( int i=0; i<15; i++ )
    {
        float hr = 0.05 + 0.12*float(i)/4.0;
        vec3 aopos =  nor * hr + pos;
        float dd = map( aopos ).x;
        occ += -(dd-hr)*sca;
        sca *= 0.95;
    }
    return clamp( 1.0 - 3.0*occ, 0.0, 1.0 );    
}

vec3 render( in vec3 ro, in vec3 rd )
{ 
    vec3 col = vec3(0.85, 0.8, .9) +rd.y*0.9;
    vec2 res = castRay(ro,rd);
    float t = res.x;
	float m = res.y;
    if( m>-0.5 )
    {
        vec3 pos = ro + t*rd;
        vec3 nor = calcNormal( pos );
        vec3 ref = reflect( rd, nor );
        
        // material        
        col =0.60+ vec3(1.0,0.0,0.0);
		
        if( m<1.5 )
        {
            
            float f = mod( floor(5.0*pos.z) + floor(5.0*pos.x), 2.0);
            col = 0.4 + 0.1*f*vec3(1.0);
        }

        // lighitng        
        float occ = calcAO( pos, nor ) ;
		vec3  lig =  normalize( vec3(-0.6, 0.7, -0.5) );
		float amb =0.0;// clamp( 0.5+0.5*nor.y, 0.0, 1.0 );
        float dif  = clamp( dot( nor, lig ), 0.0, 1.0 );
        float bac =0.0;// clamp( dot( nor, normalize(vec3(-lig.x,0.0,-lig.z))), 0.0, 1.0 )*clamp( 1.0-pos.y,0.0,1.0);
        float dom = smoothstep( -0.1, 0.1, ref.y );
        float fre = 0.750;//pow( clamp(1.0+dot(nor,rd),0.0,1.0), 2.0 );
		float spe = 0.0;//pow(clamp( dot( ref, lig ), 0.0, 1.0 ),16.0);

		vec3 lin = vec3(0.0);
        lin += 1.20*dif*vec3(1.00,0.85,0.55);
		lin += 1.20*spe*vec3(1.00,0.85,0.55)*dif;
        lin += 0.20*amb*vec3(0.50,0.70,1.00)*occ;
        lin += 0.30*dom*vec3(0.50,0.70,1.00)*occ;
        lin += 0.30*bac*vec3(0.25,0.25,0.25)*occ;
        lin += 0.40*fre*vec3(1.00,1.00,1.00)*occ;
		col = col*lin;

    	col = mix( col, vec3(0.7,0.4,.3), 1.0-exp( -0.01*t*t ) );

    }

	return vec3( clamp(col,0.0,1.0) );
}

mat3 setCamera( in vec3 ro, in vec3 ta, float cr )
{
	vec3 cw = normalize(ta-ro);
	vec3 cp = vec3(sin(cr), cos(cr),0.0);
	vec3 cu = normalize( cross(cw,cp) );
	vec3 cv = normalize( cross(cu,cw) );
    return mat3( cu, cv, cw );
}

void main(){
    vec2 q = vUv;
    vec2 p = -1.0+2.0*q;
	p.x *=  uAspectRatio;
    // vec2 mo = iMouse.xy/iResolution.xy;
		 
	float time = 15.0 + uTime*3.0;

	// camera	
    vec3 ro = vec3(0.0,4.0,4.0);
  
 	vec3 ta = vec3( -0.0, 0.0, 0.0 );
	
	// camera-to-world transformation
    mat3 ca = setCamera( ro, ta, 0.0 );
    
    // ray direction
	vec3 rd = ca * normalize( vec3(p.xy,3.0) );

    // render	
    vec3 col = render( ro, rd );

	//col = pow( col, vec3(0.7, 1., .9) );

    gl_FragColor=vec4( col, 1.0 );
}
`

// Debug
const scene = new THREE.Scene();

/**
 * Sizes
 */
const sizes = {
  width: window.innerWidth,
  height: window.innerHeight,
  resolution: null,
  pixelRatio: Math.min(window.devicePixelRatio, 2),
};
sizes.resolution = new THREE.Vector2(
  window.innerWidth * sizes.pixelRatio,
  window.innerHeight * sizes.pixelRatio
);
/**
 * Camera
 */
const camera = new THREE.PerspectiveCamera(
  75,
  sizes.width / sizes.height,
  0.1,
  100
);
camera.position.set(0, 10, 0);
scene.add(camera);
/**
 * Renderer
 */
var renderer = new THREE.WebGLRenderer({
  antialias: window.devicePixelRatio < 2,
});
renderer.outputEncoding = THREE.sRGBEncoding;
renderer.setSize(sizes.width, sizes.height);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
document.getElementById("box").appendChild(renderer.domElement);
// Controls
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;


const plane=new THREE.PlaneGeometry(2,2,1,1);
const planeMaterial=new THREE.ShaderMaterial({
  vertexShader: flowerVs,
  fragmentShader: flowerFs,
  uniforms:{
    uTime:{value:0},
    uAspectRatio:{value:sizes.resolution.x/sizes.resolution.y}
  }
})
const planeMesh=new THREE.Mesh(plane,planeMaterial);
planeMesh.rotation.x=-Math.PI/2;
scene.add(planeMesh);

/**
 * Debugger
 */
const debugObject = {
  clearColor: "#1a1414",
};

/**
 * Animate
 */
const clock = new THREE.Clock();
let time = 0;
const tick = () => {
  const elapsedTime = clock.getElapsedTime();
  time = elapsedTime;
  controls.update();

  planeMaterial.uniforms.uTime.value=elapsedTime;

  renderer.render(scene, camera);
  window.requestAnimationFrame(tick);
};
tick();
window.addEventListener("resize", () => {
  // Update sizes
  sizes.width = window.innerWidth;
  sizes.height = window.innerHeight;
  sizes.pixelRatio = Math.min(window.devicePixelRatio, 2);
  sizes.resolution.set(
    window.innerWidth * sizes.pixelRatio,
    window.innerHeight * sizes.pixelRatio
  );
  camera.aspect = sizes.width / sizes.height;
  planeMaterial.uniforms.uAspectRatio.value=sizes.resolution.x/sizes.resolution.y;
  camera.updateProjectionMatrix();
  renderer.setSize(sizes.width, sizes.height);
  renderer.setPixelRatio(sizes.pixelRatio);
});
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/redRose.js)

## 小结

- 本文提供 **红玫瑰** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

