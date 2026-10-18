<!DOCTYPE html>
<html lang="th">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Zombie FPS 3D</title>
<style>
  *{box-sizing:border-box}
  body{margin:0;overflow:hidden;background:#090d0c;color:#fff;font-family:Tahoma,Arial,sans-serif}
  #hud{position:fixed;inset:0;pointer-events:none;text-shadow:0 2px 4px #000}
  #top{position:absolute;top:14px;left:14px;right:14px;display:flex;justify-content:space-between;gap:12px;flex-wrap:wrap}
  .panel{background:#07110dcc;border:1px solid #7b9b8a;padding:10px 14px;border-radius:8px;font-size:16px}
  #crosshair{position:absolute;left:50%;top:50%;transform:translate(-50%,-50%);font-size:28px;font-weight:bold;color:#fff}
  #message{position:absolute;left:50%;top:66%;transform:translate(-50%,-50%);text-align:center;font-size:17px;line-height:1.7;width:min(90%,620px)}
  #overlay{position:fixed;inset:0;background:linear-gradient(#07100bdc,#020504f2);display:flex;align-items:center;justify-content:center;text-align:center;padding:20px;z-index:5}
  #card{max-width:650px;background:#101b18;border:1px solid #688578;border-radius:16px;padding:28px;box-shadow:0 18px 60px #000}
  h1{margin:0 0 8px;color:#b7ffb0;font-size:clamp(28px,5vw,48px)}
  p{line-height:1.7;color:#dce8df}
  button{pointer-events:auto;border:0;border-radius:8px;padding:13px 24px;background:#a7f28c;color:#10200e;font-size:18px;font-weight:bold;cursor:pointer}
  button:hover{background:#d0ffbe}
  .small{font-size:13px;color:#aebdb4}
  #hit{position:absolute;left:50%;top:44%;transform:translate(-50%,-50%);font-size:36px;color:#ff4848;opacity:0}
  #healthbar{display:inline-block;width:120px;height:10px;background:#432;vertical-align:middle;border-radius:10px;overflow:hidden;margin-left:5px}
  #healthfill{height:100%;width:100%;background:#6dff72}
  @media(max-width:600px){.panel{font-size:13px;padding:8px}#message{top:72%}}
</style>
</head>
<body>
<div id="hud">
  <div id="top">
    <div class="panel">❤️ พลังชีวิต <span id="health">100</span><span id="healthbar"><span id="healthfill" style="display:block"></span></span></div>
    <div class="panel">🎯 คะแนน: <span id="score">0</span>　🧟 ซอมบี้: <span id="count">0</span></div>
    <div class="panel">🔫 กระสุน: <span id="ammo">12</span> / ∞</div>
  </div>
  <div id="crosshair">+</div><div id="hit">✕</div>
  <div id="message"></div>
</div>
<div id="overlay">
  <div id="card">
    <h1>🧟 ZOMBIE FPS 3D</h1>
    <p>เอาชีวิตรอดจากฝูงซอมบี้ในมุมมองบุคคลที่หนึ่ง<br>เล็งแล้วยิง เก็บคะแนน และอย่าให้ซอมบี้เข้าถึงตัว!</p>
    <p><b>W A S D</b> เดิน　 <b>เมาส์</b> มองรอบตัว　 <b>คลิกซ้าย</b> ยิง<br><b>R</b> เติมกระสุน　 <b>Shift</b> วิ่ง　 <b>Esc</b> ปลดเมาส์</p>
    <button id="start">เริ่มเกม</button>
    <p class="small">เกมนี้ใช้ Three.js ผ่านอินเทอร์เน็ต กรุณาเปิดด้วย Chrome หรือ Edge และเชื่อมต่ออินเทอร์เน็ต</p>
  </div>
</div>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x87968b);
scene.fog = new THREE.Fog(0x87968b, 22, 75);
const camera = new THREE.PerspectiveCamera(75, innerWidth/innerHeight, 0.1, 120);
camera.position.set(0,1.7,6);
const renderer = new THREE.WebGLRenderer({antialias:true});
renderer.setSize(innerWidth,innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio,2));
renderer.shadowMap.enabled = true;
renderer.outputColorSpace = THREE.SRGBColorSpace;
document.body.appendChild(renderer.domElement);

scene.add(new THREE.HemisphereLight(0xc8e5ff,0x263023,2.0));
const sun = new THREE.DirectionalLight(0xfff1cf,2.2);
sun.position.set(-8,18,8); sun.castShadow=true; scene.add(sun);

const mat=(color,roughness=0.9)=>new THREE.MeshStandardMaterial({color,roughness});
const groundMat=mat(0x4a5548), wallMat=mat(0x59635a), crateMat=mat(0x65513b);
const ground=new THREE.Mesh(new THREE.PlaneGeometry(180,180),groundMat);
ground.rotation.x=-Math.PI/2; ground.position.y=0; ground.receiveShadow=true; scene.add(ground);

// ลานร้างและสิ่งกีดขวาง
const obstacles=[];
function box(x,y,z,w,h,d,material,collide=true){
 const m=new THREE.Mesh(new THREE.BoxGeometry(w,h,d),material);
 m.position.set(x,y,z);m.castShadow=true;m.receiveShadow=true;scene.add(m);
 if(collide) obstacles.push({x,z,hw:w/2+0.5,hd:d/2+0.5});
 return m;
}
box(0,2,-35,75,4,1.5,wallMat);box(0,2,35,75,4,1.5,wallMat);
box(-37,2,0,1.5,4,70,wallMat);box(37,2,0,1.5,4,70,wallMat);
for(let i=0;i<20;i++){
 const x=(Math.random()-.5)*62,z=(Math.random()-.5)*58;
 if(Math.abs(x)<5&&z>0&&z<12)continue;
 if(Math.abs(z)<3&&Math.abs(x)<8)continue;
 if(Math.random()<.55) box(x,.7,z,1.8+Math.random()*2,1.4,1.8+Math.random()*2,crateMat);
 else {
   const trunk= new THREE.Mesh(new THREE.CylinderGeometry(.18,.28,2,7),mat(0x453b2c));
   trunk.position.set(x,1,z);scene.add(trunk);
   const leaves=new THREE.Mesh(new THREE.ConeGeometry(1.3,3,7),mat(0x293f2b));
   leaves.position.set(x,3,z);leaves.castShadow=true;scene.add(leaves);
   obstacles.push({x,z,hw:0.8,hd:0.8});
 }
}
// พื้นมีเส้นตาราง
const grid=new THREE.GridHelper(140,70,0x687568,0x687568);grid.position.y=.015;scene.add(grid);

// ปืนแบบง่ายที่ติดกล้อง
const gun=new THREE.Group();
const gunBody=new THREE.Mesh(new THREE.BoxGeometry(.22,.2,.65),mat(0x252b2b));
gunBody.position.set(.34,-.28,-.58);gun.add(gunBody);
const barrel=new THREE.Mesh(new THREE.CylinderGeometry(.045,.055,.38,10),mat(0x111515));
barrel.rotation.x=Math.PI/2;barrel.position.set(.34,-.24,-.98);gun.add(barrel);
const grip=new THREE.Mesh(new THREE.BoxGeometry(.12,.25,.16),mat(0x3d3027));
grip.position.set(.34,-.43,-.43);grip.rotation.x=-.2;gun.add(grip);
camera.add(gun);scene.add(camera);

let zombies=[], score=0, health=100, ammo=12, reloading=false, playing=false, gameOver=false;
let yaw=0,pitch=0,keys={},lastShot=0,lastSpawn=0,damageFlash=0;
const $=id=>document.getElementById(id);
const overlay=$('overlay'), message=$('message');
function updateHUD(){
 $('health').textContent=Math.max(0,Math.ceil(health));$('healthfill').style.width=Math.max(0,health)+'%';
 $('healthfill').style.background=health<30?'#ff3838':'#6dff72';
 $('score').textContent=score;$('count').textContent=zombies.length;$('ammo').textContent=reloading?'...':ammo;
}
function makeZombie(x,z){
 const group=new THREE.Group();
 const skin=mat(0x718b5a), dark=mat(0x344638), shirt=mat([0x554c3e,0x384f42,0x5d3434,0x464b56][Math.floor(Math.random()*4)]);
 const torso=new THREE.Mesh(new THREE.BoxGeometry(.65,.85,.35),shirt);torso.position.y=1.2;group.add(torso);
 const head=new THREE.Mesh(new THREE.SphereGeometry(.27,12,10),skin);head.position.y=1.85;group.add(head);
 const eyeMat=new THREE.MeshBasicMaterial({color:0xff2828});
 for(const ex of [-.1,.1]){const eye=new THREE.Mesh(new THREE.SphereGeometry(.045,8,6),eyeMat);eye.position.set(ex,1.88,-.235);group.add(eye);}
 const arms=[],legs=[];
 for(const side of [-1,1]){
  const arm=new THREE.Mesh(new THREE.BoxGeometry(.18,.72,.2),skin);arm.position.set(side*.43,1.18,0);arm.rotation.z=side*-.2;group.add(arm);arms.push(arm);
  const leg=new THREE.Mesh(new THREE.BoxGeometry(.23,.65,.25),dark);leg.position.set(side*.19,.42,0);group.add(leg);legs.push(leg);
 }
 group.position.set(x,0,z);group.traverse(o=>{if(o.isMesh){o.castShadow=true;o.receiveShadow=true}});
 scene.add(group);
 return {group,hp:2,speed:1.1+Math.random()*.55,attack:0,phase:Math.random()*6,arms,legs};
}
function spawnZombie(){
 let x,z,tries=0;
 do {const a=Math.random()*Math.PI*2,r=18+Math.random()*17;x=camera.position.x+Math.sin(a)*r;z=camera.position.z+Math.cos(a)*r;tries++;}
 while((Math.abs(x)>33||Math.abs(z)>31||obstacles.some(o=>Math.abs(x-o.x)<o.hw&&Math.abs(z-o.z)<o.hd))&&tries<30);
 if(Math.abs(x)<34&&Math.abs(z)<32)zombies.push(makeZombie(x,z));
 updateHUD();
}
function blocked(x,z){return Math.abs(x)>34||Math.abs(z)>32||obstacles.some(o=>Math.abs(x-o.x)<o.hw&&Math.abs(z-o.z)<o.hd);}
function shoot(){
 if(!playing||gameOver||document.pointerLockElement!==renderer.domElement)return;
 const now=performance.now();if(now-lastShot<220||reloading)return;lastShot=now;
 if(ammo<=0){message.textContent='กระสุนหมด! กด R เพื่อเติมกระสุน';setTimeout(()=>{if(playing)message.textContent='';},900);return;}
 ammo--;gun.position.z=.08;setTimeout(()=>gun.position.z=0,70);
 // raycast ยิงซอมบี้ที่อยู่กลางจอ
 const ray=new THREE.Raycaster();ray.setFromCamera(new THREE.Vector2(0,0),camera);
 const meshes=[];zombies.forEach(z=>z.group.traverse(o=>{if(o.isMesh){o.userData.zombie=z;meshes.push(o)}}));
 const hits=ray.intersectObjects(meshes,false);
 if(hits.length&&hits[0].distance<32){
  const z=hits[0].object.userData.zombie;z.hp--;
  $('hit').style.opacity=1;setTimeout(()=>$('hit').style.opacity=0,100);
  if(z.hp<=0){scene.remove(z.group);zombies=zombies.filter(a=>a!==z);score+=10;message.textContent='+10 คะแนน';setTimeout(()=>{if(playing)message.textContent='';},500);}
 }
 updateHUD();
}
function reload(){
 if(!playing||reloading||ammo===12)return;
 reloading=true;updateHUD();message.textContent='กำลังเติมกระสุน...';
 setTimeout(()=>{ammo=12;reloading=false;message.textContent='';updateHUD();},1100);
}
function endGame(){
 gameOver=true;playing=false;document.exitPointerLock?.();
 overlay.style.display='flex';
 $('card').innerHTML=`<h1>เกมจบแล้ว!</h1><p>คะแนนของคุณ: <b>${score}</b><br>คุณกำจัดซอมบี้ได้ ${score/10} ตัว</p><button id="start">เล่นอีกครั้ง</button><p class="small">กดเริ่มเกมเพื่อเริ่มการเอาชีวิตรอดรอบใหม่</p>`;
 $('start').addEventListener('click',startGame);
}
function startGame(){
 zombies.forEach(z=>scene.remove(z.group));zombies=[];
 score=0;health=100;ammo=12;reloading=false;gameOver=false;playing=true;
 camera.position.set(0,1.7,6);yaw=0;pitch=0;camera.rotation.set(0,0,0);keys={};
 overlay.style.display='none';message.textContent='';updateHUD();
 renderer.domElement.requestPointerLock?.();
 for(let i=0;i<5;i++)spawnZombie();
}
$('start').addEventListener('click',startGame);
renderer.domElement.addEventListener('click',()=>{if(playing&&!gameOver&&document.pointerLockElement!==renderer.domElement)renderer.domElement.requestPointerLock?.();else shoot();});
document.addEventListener('pointerlockchange',()=>{if(playing&&document.pointerLockElement!==renderer.domElement)message.textContent='คลิกหน้าจอเพื่อกลับมาเล่น';else message.textContent='';});
document.addEventListener('mousemove',e=>{
 if(document.pointerLockElement!==renderer.domElement||!playing)return;
 yaw-=e.movementX*.0022;pitch-=e.movementY*.0022;
 pitch=Math.max(-1.35,Math.min(1.35,pitch));
 camera.rotation.order='YXZ';camera.rotation.y=yaw;camera.rotation.x=pitch;
});
document.addEventListener('keydown',e=>{keys[e.code]=true;if(e.code==='KeyR')reload();});
document.addEventListener('keyup',e=>keys[e.code]=false);
document.addEventListener('mousedown',e=>{if(e.button===0)shoot();});
document.addEventListener('contextmenu',e=>e.preventDefault());
let prev=performance.now();
function animate(now){
 requestAnimationFrame(animate);
 const dt=Math.min((now-prev)/1000,.05);prev=now;
 if(playing&&!gameOver){
  // เคลื่อนที่ตามทิศที่มอง
  let forward=0,side=0;
  if(keys.KeyW||keys.ArrowUp)forward+=1;if(keys.KeyS||keys.ArrowDown)forward-=1;
  if(keys.KeyD||keys.ArrowRight)side+=1;if(keys.KeyA||keys.ArrowLeft)side-=1;
  const len=Math.hypot(forward,side)||1;forward/=len;side/=len;
  const speed=(keys.ShiftLeft||keys.ShiftRight?9:5)*dt;
  const dx=(-Math.sin(yaw)*forward+Math.cos(yaw)*side)*speed;
  const dz=(-Math.cos(yaw)*forward-Math.sin(yaw)*side)*speed;
  if(!blocked(camera.position.x+dx,camera.position.z))camera.position.x+=dx;
  if(!blocked(camera.position.x,camera.position.z+dz))camera.position.z+=dz;
  camera.position.y=1.7;
  lastSpawn+=dt;
  if(lastSpawn>5&&zombies.length<14){spawnZombie();lastSpawn=0;}
  for(const z of zombies){
   const dxp=camera.position.x-z.group.position.x,dzp=camera.position.z-z.group.position.z;
   const dist=Math.hypot(dxp,dzp);
   z.phase+=dt*5;
   z.arms.forEach((a,i)=>a.rotation.x=Math.sin(z.phase+i*Math.PI)*.65);
   z.legs.forEach((l,i)=>l.rotation.x=Math.sin(z.phase+i*Math.PI)*.45);
   if(dist>1.45){
    const step=z.speed*dt;const nx=z.group.position.x+dxp/(dist||1)*step,nz=z.group.position.z+dzp/(dist||1)*step;
    if(!blocked(nx,nz)){z.group.position.x=nx;z.group.position.z=nz;}
    z.group.rotation.y=Math.atan2(dxp,dzp);
   }else{
    z.attack+=dt;
    if(z.attack>0.8){health-=8;z.attack=0;damageFlash=.15;updateHUD();if(health<=0){health=0;updateHUD();endGame();break;}}
   }
  }
  if(damageFlash>0){damageFlash-=dt;renderer.domElement.style.filter='brightness(1.2) sepia(.4) hue-rotate(-40deg)';}
  else renderer.domElement.style.filter='none';
 }
 renderer.render(scene,camera);
}
requestAnimationFrame(animate);
addEventListener('resize',()=>{camera.aspect=innerWidth/innerHeight;camera.updateProjectionMatrix();renderer.setSize(innerWidth,innerHeight);});
updateHUD();
</script>
</body>
</html>
