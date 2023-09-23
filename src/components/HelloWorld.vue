<script setup>
import { ref } from 'vue'
import * as THREE from 'three';
import axios from 'axios';
import { TilesRenderer } from '3d-tiles-renderer';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { GUI } from 'three/addons/libs/lil-gui.module.min.js';
// https://rc-cdn.wzw.cn/project3D/3DTiles/wangu3DTiles/tileset.json
defineProps({
  msg: String,
})
let tilesRenderer;

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera( 75, window.innerWidth / window.innerHeight, 0.1, 100000 );

const renderer = new THREE.WebGLRenderer();
renderer.setSize( window.innerWidth, window.innerHeight );
document.body.appendChild( renderer.domElement );



var PlaneArr = [
					new THREE.Plane( new THREE.Vector3(  1, 0, 0 ), -20.0),
					new THREE.Plane( new THREE.Vector3( -1, 0, 0 ), -20.0),
					new THREE.Plane( new THREE.Vector3( 0, 0, - 1 ), -20.0),
					new THREE.Plane( new THREE.Vector3( 0, 0,  1 ), -20.0),
					new THREE.Plane( new THREE.Vector3( 0, 1,  0 ), -20.0),
					new THREE.Plane( new THREE.Vector3( 0, -1,  0 ), -20.0),
];
// renderer.clippingPlanes = PlaneArr;
// 开启模型对象的局部剪裁平面功能
// 如果不设置为true，设置剪裁平面的模型不会被剪裁
renderer.localClippingEnabled = true;

// 通过PlaneHelper辅助可视化显示剪裁平面Plane
const planeHelpers = PlaneArr.map( p => new THREE.PlaneHelper( p, 40, 0xffffff ) );
planeHelpers.forEach( ph => {

  ph.visible = true;
  scene.add( ph );

} );
// scene.add(helper1);
// scene.add(helper2);


const controls = new OrbitControls( camera, renderer.domElement );
	controls.minDistance = 1;

const geometry = new THREE.BoxGeometry( 10, 10, 10 );
const material = new THREE.MeshBasicMaterial( { color: 0x00ff00,   
  // 设置材质的剪裁平面的属性
  // clippingPlanes: PlaneArr,
  // //改变剪裁方式，剪裁所有平面要剪裁部分的交集
  // clipIntersection: false, 
} );
const cube = new THREE.Mesh( geometry, material );
scene.add( cube );
const ambientLight = new THREE.AmbientLight( 0xffffff, 0.5 );
scene.add( ambientLight );


const tilesParent = new THREE.Group();
tilesParent.rotation.set( Math.PI / 2, 0, 0 );
scene.add( tilesParent );

const tilesRendererArr = []
const qzpath = 'https://rc-cdn.wzw.cn/project3D/3DTiles/wangu3DTiles/'
axios.get('https://rc-cdn.wzw.cn/project3D/3DTiles/wangu3DTiles/tileset.json')
  .then(function (res) {
    // 处理成功情况
    console.log(res);
    const tilesetArr = res.data.root.children[0].children;
    for (const tilese of tilesetArr) {
      tilesRenderer = new TilesRenderer( qzpath + tilese.content.uri )
      tilesRenderer.setCamera( camera )
      tilesRenderer.setResolutionFromRenderer( camera, renderer )
      tilesRenderer.onLoadModel = ((s) => {
      }) 
      console.dir( tilesRenderer);
      const tilesObj = tilesRenderer.group
      tilesObj.rotation.set(-Math.PI / 2, 0, 0)
      // console.dir(tilesObj);
      scene.add( tilesObj )
      tilesRendererArr.push(tilesRenderer)
    }
    // console.dir(scene);
  })
  .catch(function (error) {
    // 处理错误情况
    console.log(error);
  })

camera.position.z = 150;
camera.position.y = 150;



// 自定义材质
function batchIdHighlightShaderMixin( shader ) {

const newShader = { ...shader };
newShader.uniforms = {
  highlightedBatchId: { value: - 1 },
  highlightColor: { value: new Color( 0xFFC107 ).convertSRGBToLinear() },
  ...UniformsUtils.clone( shader.uniforms ),
};
newShader.extensions = {
  derivatives: true,
};
newShader.lights = true;
newShader.vertexShader =
  `
    attribute float _batchid;
    varying float batchid;
  ` +
  newShader.vertexShader.replace(
    /#include <uv_vertex>/,
    `
    #include <uv_vertex>
    batchid = _batchid;
    `
  );
newShader.fragmentShader =
  `
    varying float batchid;
    uniform float highlightedBatchId;
    uniform vec3 highlightColor;
  ` +
  newShader.fragmentShader.replace(
    /vec4 diffuseColor = vec4\( diffuse, opacity \);/,
    `
    vec4 diffuseColor =
      abs( batchid - highlightedBatchId ) < 0.5 ?
      vec4( highlightColor, opacity ) :
      vec4( diffuse, opacity );
    `
  );

return newShader;

}

const topoShader = {
	extensions: {
		derivatives: true,
	},
	vertexShader: /* glsl */`
		varying vec3 wPosition;
		varying vec3 vViewPosition;
		void main() {

			#include <begin_vertex>
			#include <project_vertex>
			wPosition = ( modelMatrix * vec4( transformed, 1.0 ) ).xyz;
			vViewPosition = - mvPosition.xyz;

		}
	`,
	fragmentShader: /* glsl */`
		varying vec3 wPosition;
		varying vec3 vViewPosition;
		void main() {

			// lighting
			vec3 fdx = vec3( dFdx( wPosition.x ), dFdx( wPosition.y ), dFdx( wPosition.z ) );
			vec3 fdy = vec3( dFdy( wPosition.x ), dFdy( wPosition.y ), dFdy( wPosition.z ) );
			vec3 worldNormal = normalize( cross( fdx, fdy ) );

			float lighting =
				0.4 +
				clamp( dot( worldNormal, vec3( 1.0, 1.0, 1.0 ) ), 0.0, 1.0 ) * 0.5 +
				clamp( dot( worldNormal, vec3( - 1.0, 1.0, - 1.0 ) ), 0.0, 1.0 ) * 0.3;

			// thickness scale
			float upwardness = dot( worldNormal, vec3( 0.0, 1.0, 0.0 ) );
			float yInv = clamp( 1.0 - abs( upwardness ), 0.0, 1.0 );
			float thicknessScale = pow( yInv, 0.4 );
			thicknessScale *= 0.25 + 0.5 * ( vViewPosition.z + 1.0 ) / 2.0;

			// thickness
			float thickness = 0.01 * thicknessScale;
			float thickness2 = thickness / 2.0;
			float m = mod( wPosition.y, 3.0 );

			// soften edge
			float center = thickness2;
			float dist = clamp( abs( m - thickness2 ) / thickness2, 0.0, 1.0 );

			vec4 topoColor = vec4( 0.149, 0.196, 0.219, 1.0 ) * 0.5;
			gl_FragColor = mix( topoColor * lighting, vec4( lighting ), dist );

		}
	`,
  // 设置材质的剪裁平面的属性
  clippingPlanes: PlaneArr,
  //改变剪裁方式，剪裁所有平面要剪裁部分的交集
  clipIntersection: false,
};

  /**
  *  射线检测物体
  */
  const intersectables = [];

    /**
     *  射线检测
     */
    const raycaster = new THREE.Raycaster();
    const mouse = new THREE.Vector2();
    const point = new THREE.Vector3();
    const normal = new THREE.Vector3();
    function onMouseMove( event ) {
      console.log(event);
      console.log(scene);
    	const bounds = renderer.domElement.getBoundingClientRect();
      mouse.x = event.clientX - bounds.x;
      mouse.y = event.clientY - bounds.y;
      mouse.x = ( mouse.x / bounds.width ) * 2 - 1;
      mouse.y = - ( mouse.y / bounds.height ) * 2 + 1;
      raycaster.setFromCamera( mouse, camera );
    	const intersects = raycaster.intersectObject( scene, true ); // 射线检测
      console.log(intersects);
    	if ( intersects.length ) {

    		const { face, object } = intersects[ 0 ];
        console.log(face);
        console.log(object);
        if (object.isMesh) {
          // object.material.clippingPlanes = PlaneArr;

          // object.material = new THREE.ShaderMaterial( topoShader );
          object.material.clipIntersection = true;
          object.material.clippingPlanes = PlaneArr;
          // console.dir(object.material);
					// object.material.side = 2;
					// object.material.flatShading = true;
					// object.receiveShadow = false;
					// object.castShadow = false;
        }
      }

    }

    window.addEventListener( 'click', onMouseMove, false );
// 



const params = {

  animate: true,
  planeX: {

    constant: 0.2,
    negated: false,
    displayHelper: true

  },
  planeY: {

    constant: 0.2,
    negated: false,
    displayHelper: true

  },
  planeZ: {

    constant: 0.2,
    negated: false,
    displayHelper: true

  },
  planeH: {

    constant: 0.2,
    negated: false,
    displayHelper: true

  }


};


  // GUI
  const gui = new GUI();
  gui.add( params, 'animate' );

  const planeX = gui.addFolder( 'planeX' );
  planeX.add( params.planeX, 'displayHelper' ).onChange( v => planeHelpers[ 0 ].visible = v );
  planeX.add( params.planeX, 'constant' ).min( -100 ).max( 100 ).onChange( d => PlaneArr[ 0 ].constant = d );
  planeX.add( params.planeX, 'negated' ).onChange( () => {

    PlaneArr[ 0 ].negate();
    params.planeX.constant = PlaneArr[ 0 ].constant;

  } );
  planeX.open();

  const planeY = gui.addFolder( 'planeY' );
  planeY.add( params.planeY, 'displayHelper' ).onChange( v => planeHelpers[ 1 ].visible = v );
  planeY.add( params.planeY, 'constant' ).min( -100 ).max( 100 ).onChange( d => PlaneArr[ 1 ].constant = d );
  planeY.add( params.planeY, 'negated' ).onChange( () => {

    PlaneArr[ 1 ].negate();
    params.planeY.constant = PlaneArr[ 1 ].constant;

  } );
  planeY.open();



function animate() {
  requestAnimationFrame( animate );

  cube.rotation.x += 0.01;
  cube.rotation.y += 0.01;
  for (const tilesRenderer of tilesRendererArr) {
    // tilesRenderer.displayBoxBounds = true;
    tilesRenderer.update();
  }
  renderer.render( scene, camera );
}

animate();

</script>

<template>

</template>

<style scoped>
.read-the-docs {
  color: #888;
}
</style>
