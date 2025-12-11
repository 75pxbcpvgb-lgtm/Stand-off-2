# Stand-off-2
Gh
<!doctype html>
<html lang="ru">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover" />
<title>Mini FPS — мобильный прототип</title>
<style>
  html,body{height:100%;margin:0;overflow:hidden;background:#111}
  #gameCanvas{display:block;width:100%;height:100%}
  /* HUD */
  .hud {
    position: absolute; left: 12px; top: 12px;
    color: #fff; font-family: Arial, sans-serif; font-size: 14px;
    text-shadow: 0 1px 4px rgba(0,0,0,.8);
  }
  .hud .row{margin-bottom:6px}
  /* fire button */
  #fireBtn{
    position:absolute; right:18px; bottom:22px;
    width:84px; height:84px; border-radius:50%;
    background: rgba(255,60,60,0.9); color:#fff;
    display:flex; align-items:center; justify-content:center;
    font-weight:bold; font-size:18px; user-select:none;
    -webkit-user-select:none;
  }
  /* joystick */
  #leftStick {
    position:absolute; left:18px; bottom:22px;
    width:140px; height:140px; border-radius:50%;
    background: rgba(255,255,255,0.06);
    touch-action:none;
  }
  #leftThumb {
    position:absolute; left:50%; top:50%; transform:translate(-50%,-50%);
    width:56px; height:56px; border-radius:50%;
    background: rgba(255,255,255,0.18);
    transition: 0.06s;
    touch-action:none;
  }
  /* message */
  #msg {
    position:absolute; left:50%; top:12px; transform:translateX(-50%);
    color:#fff; font-family: Arial; font-size:13px; text-shadow:0 1px 3px #000;
    background: rgba(0,0,0,0.35); padding:6px 10px; border-radius:8px;
  }
  /* small crosshair */
  #cross {
    position:absolute; left:50%; top:50%; transform:translate(-50%,-50%);
    width:8px; height:8px; border-radius:2px; background:rgba(255,255,255,0.9); opacity:0.9;
  }
  /* control overlay for looking */
  #lookArea { position:absolute; right:0; left:50%; top:0; bottom:0; touch-action:none; }
  /* simple button style (reset) */
  button{border:0;outline:0}
</style>
</head>
<body>
<canvas id="gameCanvas"></canvas>
<div id="cross"></div>
<div class="hud">
  <div class="row">HP: <span id="hp">100</span></div>
  <div class="row">Ammo: <span id="ammo">30</span></div>
  <div class="row">Kills: <span id="kills">0</span></div>
</div>
<div id="msg">Левый джойстик — движение. Правая половина — смотреть. Кнопка — стрелять.</div>
<div id="leftStick"><div id="leftThumb"></div></div>
<div id="lookArea"></div>
<button id="fireBtn">FIRE</button>

<!-- Three.js r146 (non-module) -->
<script src="https://cdn.jsdelivr.net/npm/three@0.146.0/build/three.min.js"></script>

<script>
/* ========== Простая 3D FPS-логика ========== */
/* Настройки */
const SETTINGS = {
  moveSpeed: 3.4, runMultiplier: 1.6,
  lookSpeed: 0.15, // сенсор чувствительность
  gravity: -9.8, jumpForce: 5,
  maxHp: 100,
  weapon: {damage: 25, fireRate: 0.12, mag: 30, range: 80}
};

/* Сцена, камера, рендер */
const canvas = document.getElementById('gameCanvas');
const renderer = new THREE.WebGLRenderer({canvas, antialias:true});
renderer.setPixelRatio(window.devicePixelRatio);
renderer.shadowMap.enabled = true;

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x8fbfff);

const camera = new THREE.PerspectiveCamera(75, innerWidth/innerHeight, 0.1, 2000);
camera.position.set(0,1.6,0);

/* Light */
const hemi = new THREE.HemisphereLight(0xffffff, 0x444466, 0.9);
scene.add(hemi);
const sun = new THREE.DirectionalLight(0xffffff, 0.9);
sun.position.set(5,10,2);
sun.castShadow = true;
scene.add(sun);

/* Ground */
const groundMat = new THREE.MeshPhongMaterial({color:0x5a7a42});
const ground = new THREE.Mesh(new THREE.BoxGeometry(200,1,200), groundMat);
ground.position.y = -0.5;
ground.receiveShadow = true;
scene.add(ground);

/* Simple environment: boxes as cover */
const coverMat = new THREE.MeshStandardMaterial({color:0x6b6b6b});
for(let i=0;i<18;i++){
  const b = new THREE.Mesh(new THREE.BoxGeometry(2+Math.random()*4,1+Math.random()*2,2+Math.random()*4), coverMat);
  b.position.set((Math.random()-0.5)*60, 0.5 + Math.random()*2, (Math.random()-0.5)*60);
  b.castShadow = true; b.receiveShadow=true;
  scene.add(b);
}

/* Player state */
const player = {
  pos: new THREE.Vector3(0,1.6,5),
  vel: new THREE.Vector3(),
  yaw: 0, pitch: 0,
  hp: SETTINGS.maxHp,
  ammo: SETTINGS.weapon.mag,
  lastFire: -999,
  kills: 0
};

/* Camera pivot: sync with player.pos */
const cameraHolder = new THREE.Object3D();
cameraHolder.position.copy(player.pos);
cameraHolder.add(camera);
scene.add(cameraHolder);

/* Enemies list */
const enemies = [];
const enemyGeometry = new THREE.BoxGeometry(0.9,1.6,0.9);
const enemyMat = new THREE.MeshStandardMaterial({color:0xcc4444});

/* Spawn some enemies */
function spawnEnemy() {
  const e = new THREE.Mesh(enemyGeometry, enemyMat);
  e.position.set((Math.random()-0.5)*70, 0.8, (Math.random()-0.5)*70);
  e.userData = {hp: 60};
  scene.add(e);
  enemies.push(e);
}
for(let i=0;i<6;i++) spawnEnemy();

/* Utility: update HUD */
const elHp = document.getElementById('hp'), elAmmo = document.getElementById('ammo'), elKills = document.getElementById('kills');
function updateHUD(){ elHp.textContent = Math.max(0, Math.round(player.hp)); elAmmo.textContent = player.ammo; elKills.textContent = player.kills; }

/* Movement inputs (virtual joystick + look) */
let moveInput = {x:0,y:0};
let lookDelta = {x:0,y:0};
let isFiring=false;

/* Left joystick implementation */
const leftStick = document.getElementById('leftStick');
const leftThumb = document.getElementById('leftThumb');
let leftId = null, origin = {x:0,y:0}, radius = 56;
leftStick.addEventListener('touchstart', e=>{
  const t = e.changedTouches[0];
  leftId = t.identifier;
  const rect = leftStick.getBoundingClientRect();
  origin.x = rect.left + rect.width/2;
  origin.y = rect.top + rect.height/2;
  updateLeft(t.clientX, t.clientY);
  e.preventDefault();
});
leftStick.addEventListener('touchmove', e=>{
  for(const t of e.changedTouches) if(t.identifier===leftId){ updateLeft(t.clientX, t.clientY); e.preventDefault(); break; }
});
leftStick.addEventListener('touchend', e=>{
  for(const t of e.changedTouches) if(t.identifier===leftId){ leftId = null; moveInput.x=0; moveInput.y=0; leftThumb.style.transform='translate(-50%,-50%)'; e.preventDefault(); break;}
});
function updateLeft(x,y){
  const dx = x - origin.x, dy = y - origin.y;
  const dist = Math.min(radius, Math.hypot(dx,dy));
  const nx = dx/dist||0, ny = dy/dist||0;
  leftThumb.style.transform = `translate(calc(-50% + ${nx*dist}px), calc(-50% + ${ny*dist}px))`;
  moveInput.x = (dx / radius); moveInput.y = (-dy / radius); // forward is -y on screen
  // clamp
  if(Math.abs(moveInput.x) < 0.05) moveInput.x=0;
  if(Math.abs(moveInput.y) < 0.05) moveInput.y=0;
}

/* Look area (right half) */
const lookArea = document.getElementById('lookArea');
let lookId = null;
lookArea.addEventListener('touchstart', e=>{
  const t = e.changedTouches[0];
  lookId = t.identifier;
  e.preventDefault();
});
lookArea.addEventListener('touchmove', e=>{
  for(const t of e.changedTouches) if(t.identifier===lookId){
    // relative delta
    const dx = t.movementX || (t.clientX - (t._prevX||t.clientX));
    const dy = t.movementY || (t.clientY - (t._prevY||t.clientY));
    t._prevX = t.clientX; t._prevY = t.clientY;
    lookDelta.x += dx * SETTINGS.lookSpeed * 0.05;
    lookDelta.y += dy * SETTINGS.lookSpeed * 0.05;
    e.preventDefault(); break;
  }
});
lookArea.addEventListener('touchend', e=>{
  for(const t of e.changedTouches) if(t.identifier===lookId){ lookId = null; break; }
});

/* Fire button */
const fireBtn = document.getElementById('fireBtn');
fireBtn.addEventListener('touchstart', ()=>{ isFiring=true; });
fireBtn.addEventListener('touchend', ()=>{ isFiring=false; });

/* Desktop mouse support (so you can test on PC) */
let mouseDown=false;
window.addEventListener('mousedown', e=>{ if(e.button===0){ mouseDown=true; }});
window.addEventListener('mouseup', e=>{ if(e.button===0){ mouseDown=false; }});
window.addEventListener('mousemove', e=>{
  if(e.buttons & 1){
    lookDelta.x += e.movementX * 0.15;
    lookDelta.y += e.movementY * 0.15;
  }
});

/* Shooting / raycast */
const raycaster = new THREE.Raycaster();
function tryShoot(){
  const now = performance.now()/1000;
  if(player.ammo <= 0) return;
  if(now - player.lastFire < SETTINGS.weapon.fireRate) return;
  player.lastFire = now;
  player.ammo--;
  // ray from camera forward
  const origin = new THREE.Vector3();
  origin.copy(camera.getWorldPosition(new THREE.Vector3()));
  const dir = new THREE.Vector3(0,0,-1).applyQuaternion(camera.quaternion).normalize();
  raycaster.set(origin, dir);
  const hits = raycaster.intersectObjects(enemies, false);
  if(hits.length>0){
    const h = hits[0];
    const enemy = h.object;
    enemy.userData.hp -= SETTINGS.weapon.damage;
    // small flash
    enemy.material.emissive = new THREE.Color(0xff3333);
    setTimeout(()=>{ try{ enemy.material.emissive.setHex(0x000000); }catch(e){} }, 90);
    if(enemy.userData.hp <= 0){
      // kill
      scene.remove(enemy);
      const idx = enemies.indexOf(enemy);
      if(idx>=0) enemies.splice(idx,1);
      player.kills++;
      updateHUD();
      // spawn a new enemy after delay
      setTimeout(()=>spawnEnemy(), 1000 + Math.random()*3000);
    }
  } else {
    // miss — nothing
  }
  updateHUD();
}

/* Simple enemy AI: move toward player on ground */
function updateEnemies(dt){
  for(const e of enemies){
    const dir = new THREE.Vector3().subVectors(cameraHolder.position, e.position);
    dir.y = 0;
    if(dir.length() > 0.5){
      dir.normalize();
      e.position.addScaledVector(dir, dt * 1.4);
    }
    // check collision with player (damage)
    if(e.position.distanceTo(cameraHolder.position) < 1.4){
      // deal damage slowly
      if(!e.userData._lastHit || performance.now()-e.userData._lastHit > 800){
        player.hp -= 6;
        e.userData._lastHit = performance.now();
        updateHUD();
        if(player.hp <= 0){
          player.hp = 0;
          updateHUD();
          // respawn player after short delay
          setTimeout(()=>{ player.hp = SETTINGS.maxHp; player.ammo = SETTINGS.weapon.mag; cameraHolder.position.set(0,1.6,5); updateHUD(); }, 1200);
        }
      }
    }
  }
}

/* Resize */
function onResize(){
  camera.aspect = innerWidth/innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth, innerHeight);
}
addEventListener('resize', onResize);
onResize();

/* Main loop */
let prev = performance.now();
function animate(){
  const now = performance.now();
  const dt = Math.min(0.05, (now - prev) / 1000);
  prev = now;

  // apply look
  player.yaw -= lookDelta.x;
  player.pitch = Math.max(-85, Math.min(85, player.pitch - lookDelta.y));
  lookDelta.x = 0; lookDelta.y = 0;

  cameraHolder.rotation.y = THREE.MathUtils.degToRad(player.yaw);
  camera.rotation.x = THREE.MathUtils.degToRad(player.pitch);

  // movement (moveInput provided by joystick)
  const forward = new THREE.Vector3(0,0,-1).applyAxisAngle(new THREE.Vector3(0,1,0), cameraHolder.rotation.y);
  const right = new THREE.Vector3(1,0,0).applyAxisAngle(new THREE.Vector3(0,1,0), cameraHolder.rotation.y);
  const move = new THREE.Vector3();
  move.addScaledVector(forward, moveInput.y);
  move.addScaledVector(right, moveInput.x);
  if(move.lengthSq()>1) move.normalize();
  cameraHolder.position.addScaledVector(move, dt * SETTINGS.moveSpeed);

  // keep camera y
  cameraHolder.position.y = 1.6;

  // update cameraHolder to player.pos
  // (camera is child of holder)

  // shooting
  if(isFiring) tryShoot();

  // update enemies
  updateEnemies(dt);

  // small ambient camera bob based on movement (for feel)
  const bob = Math.sin(performance.now()/160) * 0.003 * move.length();
  camera.position.y = 1.6 + bob;

  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
updateHUD();
animate();

/* Prevent default gestures that break gameplay on mobile */
document.addEventListener('touchmove', function(e){ /* allow our handlers */ }, {passive:false});

/* Optional: handle reload friendly key for desktop testing */
window.addEventListener('keydown', e=>{
  if(e.key === 'r'){ // refill
    player.ammo = SETTINGS.weapon.mag; updateHUD();
  }
  if(e.key === 'k'){ // spawn extra enemy
    spawnEnemy();
  }
});
</script>
</body>
</html>
