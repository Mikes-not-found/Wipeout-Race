<script setup>
import { ref, onMounted } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import * as CANNON from 'cannon-es';
import CannonDebugger from 'cannon-es-debugger';

defineProps({
  msg: String,
})

const scene = new THREE.Scene();

const world = new CANNON.World({
  gravity: new CANNON.Vec3(0, -9.82, 0)
});
const cannonDebugger = new CannonDebugger(scene, world, {
  color: '#AEE2FF',
  scale: 1
});

const camera = new THREE.PerspectiveCamera( 75, window.innerWidth / window.innerHeight, 0.1, 1000 );
camera.position.z = 4.5;
camera.position.y = 1.5;


const renderer = new THREE.WebGLRenderer();
renderer.setSize( window.innerWidth, window.innerHeight );
renderer.setAnimationLoop( animate );
document.body.appendChild( renderer.domElement );


const randomRangeNum = (max, min) => {
  return Math.floor(Math.random() * (max - min + 1) + min);
}

  const moveObstacles = (arr, speed, maxX, minX, maxZ, minZ) => {
    arr.forEach((el) => {
      el.body.position.z += speed;
      if (el.body.position.z > camera.position.z) {
        el.body.position.x = randomRangeNum(maxX, minX);
        el.body.position.z = randomRangeNum(maxZ, minZ);
      }
      el.mesh.position.copy(el.body.position);
      el.mesh.quaternion.copy(el.body.quaternion);
  });
};

//CONTROLS
const controls = new OrbitControls(camera, renderer.domElement);
// const gridHelper = new THREE.GridHelper(30,30);
// scene.add(gridHelper);

//GROUND
const groundBody = new CANNON. Body({
  shape: new CANNON.Box(new CANNON.Vec3(15, 0.5, 15)),
});
groundBody.position.y = -1;
world.addBody(groundBody);

const ground = new THREE.Mesh( new THREE.BoxGeometry( 30, 1, 30), new THREE.MeshBasicMaterial( { color: 0x00ff00 } ));
ground.position.y = -1;
scene.add( ground );

//PLAYER
const playerBody = new CANNON. Body({
  mass: 1,
  shape: new CANNON.Box(new CANNON.Vec3(0.25, 0.25, 0.25)),
  fixedRotation: true,
});

world.addBody(playerBody);

const player = new THREE.Mesh( new THREE.BoxGeometry( 0.5,  0.5,  0.5), new THREE.MeshBasicMaterial( { color: 0xff0000 } ));
scene.add( player );

// POINTS
let points = 0;
let pointsUI;

onMounted(() => {
  pointsUI = document.querySelector("#pointsUI");
  playerBody.addEventListener("collide", (e) => {
    powerups.forEach((el) => {
      if (e.body === el.body) {
        el.body.position.x = randomRangeNum(8, -8);
        el.body.position.z = randomRangeNum(-5, -10);
        el.mesh.position.copy(el.body.position);
        el.mesh.quaternion.copy(el.body.quaternion);
        points += 1;
        pointsUI.textContent = points.toString();
      }
    });
  });
});

//POWERUP
const powerups = []
for (let i = 0; i < 10; i++) {
  const posX = randomRangeNum(8, -8);
  const posZ = randomRangeNum(-2, -5);
  const powerup = new THREE.Mesh(
    new THREE. TorusGeometry(1, 0.4, 16, 50),
    new THREE.MeshBasicMaterial({ color: 0xffff00 })
  );

  powerup.scale.set(0.1, 0.1, 0.1);
  powerup.position.x = posX;
  powerup.position.z = posZ;
  powerup. name = "powerup" + [i + 1];
  scene.add(powerup);

  const powerupBody = new CANNON.Body({
    shape: new CANNON.Sphere(0.2),
  });

  powerupBody.position.set(posX, 0, posZ);
  world.addBody(powerupBody);

  const powerupObject = {
    mesh: powerup,
    body: powerupBody
  };

  powerups.push(powerupObject);
}

//ENEMIES
const enemies = []
for (let i = 0; i < 5; i++) {
  const posX = randomRangeNum(8, -8);
  const posZ = randomRangeNum(-2, -5);
  const enemy = new THREE.Mesh(
    new THREE.BoxGeometry(1, 1, 1),
    new THREE.MeshBasicMaterial({ color: 0x0000ff })
  );

  enemy.position.x = posX;
  enemy.position.z = posZ;
  enemy.name = "enemy" + [i + 1];
  scene.add(enemy);

  const enemyBody = new CANNON.Body({
    shape: new CANNON.Box(new CANNON.Vec3(0.5, 0.5, 0.5))
  });
  enemyBody.position.set(posX, 0, posZ);
  world.addBody(enemyBody);

  const enemyObject = {
    mesh: enemy,
    body: enemyBody
  };

  enemies.push(enemyObject);
}




let keys = {
  ArrowUp: false,
  ArrowDown: false,
  ArrowLeft: false,
  ArrowRight: false,
  w: false,
  a: false,
  s: false,
  d: false,
  r: false,
  R: false,
  x: false
};

window.addEventListener("keydown", (e) => {
  if (e.key in keys) {
    keys[e.key] = true;
  }
});

window.addEventListener("keyup", (e) => {
  if (e.key in keys) {
    keys[e.key] = false;
  }
});

function update() {
  if (keys.ArrowUp || keys.w) {
    playerBody.position.z -= 0.1;
  }
  if (keys.ArrowDown || keys.s) {
    playerBody.position.z += 0.1;
  }
  if (keys.ArrowLeft || keys.a) {
    playerBody.position.x -= 0.1;
  }
  if (keys.ArrowRight || keys.d) {
    playerBody.position.x += 0.1;
  }
    
    if (keys.r || keys.R) {
      playerBody.position.x = 0;
      playerBody.position.y = 0;
      playerBody.position.z = 0;
      points = 0;
      pointsUI.textContent = points.toString();
    }
    if (keys.x) {
      playerBody.position.y = 0;
    }
}

function animate() {
  requestAnimationFrame(update);
  moveObstacles(powerups, 0.01, 8, -8, -2, -5);
  moveObstacles(enemies, 0.02, 8, -8, -2, -5);

  controls.update();
  world.fixedStep();
  player.position.copy(playerBody.position);
  player.quaternion.copy(playerBody.quaternion);

  cannonDebugger.update();
	renderer.render( scene, camera );

}

window.addEventListener("resize", () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  })


</script>

<template>
  <div id="ui">
    <h1>
      Points: <span id="pointsUI">00</span>
    </h1>
  </div>
</template>
<style scoped>

</style>
