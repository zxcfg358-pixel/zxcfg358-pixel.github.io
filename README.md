<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>PitBike Run — Угон от ГАИ</title>
<style>
  * { margin:0; padding:0; box-sizing:border-box; user-select:none; }
  html, body { width:100%; height:100%; overflow:hidden; background:#0b0f1a;
    font-family: "Segoe UI", system-ui, sans-serif; color:#e8ecf5; }
  canvas { display:block; position:absolute; inset:0; }

  /* ---------- OVERLAY / UI ---------- */
  .overlay {
    position:absolute; inset:0; display:flex; align-items:center; justify-content:center;
    backdrop-filter: blur(14px); background: rgba(10,14,25,.55);
    z-index:10; transition: opacity .35s ease;
  }
  .overlay.hidden { opacity:0; pointer-events:none; }

  .panel {
    width: min(900px, 92vw); max-height: 88vh; overflow:auto;
    background: rgba(25,30,48,.75);
    border: 1px solid rgba(255,255,255,.08);
    border-radius: 18px; padding: 28px 32px;
    box-shadow: 0 30px 80px rgba(0,0,0,.55), inset 0 1px 0 rgba(255,255,255,.06);
    animation: pop .35s cubic-bezier(.2,.9,.3,1.2);
  }
  @keyframes pop { from { transform: scale(.85) translateY(20px); opacity:0; } to { transform:none; opacity:1; } }

  /* ---------- 3D TITLE ---------- */
  .title3d {
    font-size: clamp(42px, 7vw, 84px); font-weight: 900; letter-spacing: 2px;
    text-align:center; margin-bottom: 24px;
    background: linear-gradient(180deg,#ffd86b 0%,#ff7a3d 55%,#b23a00 100%);
    -webkit-background-clip: text; background-clip:text; color:transparent;
    text-shadow:
      0 1px 0 #ffb26b, 0 2px 0 #e88a3a, 0 3px 0 #c66a22,
      0 4px 0 #9a4d12, 0 5px 0 #6e3408,
      0 8px 18px rgba(255,120,40,.55),
      0 0 40px rgba(255,180,60,.35);
    transform: perspective(600px) rotateX(18deg);
    filter: drop-shadow(0 10px 14px rgba(0,0,0,.6));
  }
  .subtitle { text-align:center; opacity:.75; margin-bottom: 28px; font-size:14px; letter-spacing:3px; }

  /* ---------- BUTTONS (Win11 style) ---------- */
  .btn {
    display:block; width: 100%; max-width: 360px; margin: 12px auto;
    padding: 14px 22px; font-size: 18px; font-weight: 600;
    color:#f2f5fb; background: rgba(255,255,255,.06);
    border: 1px solid rgba(255,255,255,.12);
    border-radius: 12px; cursor:pointer;
    backdrop-filter: blur(10px);
    transition: transform .15s ease, background .2s ease, box-shadow .2s ease;
  }
  .btn:hover { background: rgba(255,255,255,.12); transform: translateY(-2px);
    box-shadow: 0 10px 24px rgba(0,0,0,.35); }
  .btn:active { transform: translateY(0); }
  .btn.primary { background: linear-gradient(135deg,#ff8a3d,#ff5a2c); border-color: transparent; }
  .btn.primary:hover { background: linear-gradient(135deg,#ff9a55,#ff6a3c); }

  .gear {
    position:absolute; top:18px; right:18px; width:46px; height:46px;
    border-radius:50%; background: rgba(255,255,255,.08);
    border:1px solid rgba(255,255,255,.15); display:flex; align-items:center; justify-content:center;
    cursor:pointer; font-size:22px; transition: transform .4s ease, background .2s;
  }
  .gear:hover { transform: rotate(90deg); background: rgba(255,255,255,.15); }

  /* ---------- HUD ---------- */
  #hud {
    position:absolute; top:12px; left:12px; z-index:5;
    background: rgba(15,20,35,.6); padding:10px 14px; border-radius:12px;
    font-size:14px; line-height:1.5; border:1px solid rgba(255,255,255,.08);
    backdrop-filter: blur(8px); min-width: 220px;
  }
  #hud b { color:#ffd86b; }

  /* ---------- SHOP / UPGRADES ---------- */
  .grid { display:grid; grid-template-columns: repeat(auto-fill, minmax(180px,1fr)); gap:14px; margin-top:18px; }
  .card {
    background: rgba(255,255,255,.05); border:1px solid rgba(255,255,255,.1);
    border-radius:12px; padding:14px; text-align:center; cursor:pointer;
    transition: transform .15s, background .2s;
  }
  .card:hover { transform: translateY(-3px); background: rgba(255,255,255,.1); }
  .card.owned { border-color:#4ade80; }
  .card.equipped { border-color:#ffd86b; box-shadow: 0 0 18px rgba(255,216,107,.35); }
  .card .icon { font-size: 38px; margin-bottom: 6px; }
  .card .name { font-weight:600; margin-bottom:4px; }
  .card .price { color:#ffd86b; font-size:13px; }
  .card .speed { color:#9ad; font-size:12px; }

  .tabs { display:flex; gap:8px; margin-bottom:12px; flex-wrap:wrap; }
  .tab { padding:8px 14px; border-radius:10px; background: rgba(255,255,255,.06);
    border:1px solid rgba(255,255,255,.1); cursor:pointer; font-size:14px; }
  .tab.active { background: linear-gradient(135deg,#ff8a3d,#ff5a2c); border-color: transparent; }

  .bar { height:8px; background: rgba(255,255,255,.1); border-radius:4px; overflow:hidden; margin-top:6px; }
  .bar > div { height:100%; background: linear-gradient(90deg,#4ade80,#ffd86b); transition: width .3s; }

  .row { display:flex; justify-content:space-between; align-items:center; padding:10px 0;
    border-bottom:1px solid rgba(255,255,255,.06); }
  .row:last-child { border:none; }
  .row input[type=range] { width:180px; }

  .close { position:absolute; top:14px; right:14px; width:34px; height:34px; border-radius:50%;
    background: rgba(255,255,255,.08); border:1px solid rgba(255,255,255,.15);
    display:flex; align-items:center; justify-content:center; cursor:pointer; font-size:18px; }
  .close:hover { background: rgba(255,80,80,.4); }

  .money { position:absolute; top:14px; left:50%; transform:translateX(-50%);
    background: rgba(15,20,35,.7); padding:8px 16px; border-radius:20px;
    font-weight:700; z-index:5; border:1px solid rgba(255,216,107,.3); }
  .money b { color:#ffd86b; }

  .hint { font-size:12px; opacity:.6; text-align:center; margin-top:12px; }
</style>
</head>
<body>

<canvas id="game"></canvas>

<!-- HUD -->
<div id="hud" style="display:none">
  <div>Скорость: <b id="hSpeed">0</b> км/ч</div>
  <div>Пройдено: <b id="hDist">0</b> м</div>
  <div>Заработано: <b id="hMoney">0.0</b> ₽</div>
  <div>Здоровье: <b id="hHp">100</b></div>
  <div style="margin-top:6px; font-size:12px; opacity:.7">ESC — меню · Space — тормоз</div>
</div>
<div class="money" id="moneyBox" style="display:none">💰 <b id="moneyVal">0.0</b> ₽</div>

<!-- MAIN MENU -->
<div class="overlay" id="menuOverlay">
  <div class="gear" onclick="openSettings()" title="Настройки">⚙️</div>
  <div class="panel" style="text-align:center">
    <div class="title3d">PITBIKE RUN</div>
    <div class="subtitle">УГОН · СКОРОСТЬ · ПОБЕГ</div>
    <button class="btn primary" onclick="openPlay()">🏁 Играть</button>
    <button class="btn" onclick="openShop()">🛒 Магазин</button>
    <button class="btn" onclick="openUpgrades()">⚡ Улучшения</button>
    <div class="hint">WASD — движение · Space — тормоз · Esc — меню</div>
  </div>
</div>

<!-- PLAY (difficulty) -->
<div class="overlay hidden" id="playOverlay">
  <div class="panel" style="position:relative">
    <div class="close" onclick="closeOverlay('playOverlay')">✕</div>
    <h2 style="margin-bottom:14px">Выберите сложность</h2>
    <button class="btn" onclick="startGame('easy')">🟢 Лёгкий — 0.2 ₽/м · пустая карта</button>
    <button class="btn" onclick="startGame('medium')">🟡 Средний — 0.5 ₽/м · здания + ГАИ</button>
    <button class="btn" onclick="startGame('hard')">🔴 Сложный — 1.0 ₽/м · люди, парки, быстрые копы</button>
  </div>
</div>

<!-- SHOP -->
<div class="overlay hidden" id="shopOverlay">
  <div class="panel" style="position:relative">
    <div class="close" onclick="closeOverlay('shopOverlay')">✕</div>
    <h2>🛒 Магазин</h2>
    <div class="tabs" id="shopTabs">
      <div class="tab active" data-tab="bikes">Пит-байки</div>
      <div class="tab" data-tab="helmets">Шлемы</div>
      <div class="tab" data-tab="trails">Следы шин</div>
    </div>
    <div class="grid" id="shopGrid"></div>
  </div>
</div>

<!-- UPGRADES -->
<div class="overlay hidden" id="upgOverlay">
  <div class="panel" style="position:relative">
    <div class="close" onclick="closeOverlay('upgOverlay')">✕</div>
    <h2>⚡ Улучшения</h2>
    <div id="upgList"></div>
  </div>
</div>

<!-- SETTINGS -->
<div class="overlay hidden" id="settingsOverlay">
  <div class="panel" style="position:relative">
    <div class="close" onclick="closeOverlay('settingsOverlay')">✕</div>
    <h2>⚙️ Настройки</h2>
    <div class="row"><span>Качество графики</span>
      <select id="setQuality"><option value="low">Низкое</option><option value="med" selected>Среднее</option><option value="high">Высокое</option></select></div>
    <div class="row"><span>Режим производительности (для слабых ПК)</span>
      <input type="checkbox" id="setPerf"></div>
    <div class="row"><span>Громкость мотора</span><input type="range" id="setEngine" min="0" max="100" value="60"></div>
    <div class="row"><span>Громкость шин</span><input type="range" id="setTires" min="0" max="100" value="50"></div>
    <div class="row"><span>Сбросить прогресс</span>
      <button class="btn" style="margin:0; max-width:160px; padding:8px" onclick="resetProgress()">Сбросить</button></div>
  </div>
</div>

<!-- GAME OVER -->
<div class="overlay hidden" id="overOverlay">
  <div class="panel" style="text-align:center">
    <h2 style="color:#ff6b6b; margin-bottom:12px">💥 Вас поймали!</h2>
    <p>Пройдено: <b id="overDist">0</b> м</p>
    <p>Заработано: <b id="overMoney" style="color:#ffd86b">0.0</b> ₽</p>
    <button class="btn primary" onclick="backToMenu()" style="margin-top:18px">В главное меню</button>
  </div>
</div>

<script>
/* =========================================================
   PITBIKE RUN — core
   ========================================================= */
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
let W=0,H=0;
function resize(){ W=canvas.width=innerWidth; H=canvas.height=innerHeight; }
addEventListener('resize', resize); resize();

/* ---------- SAVE ---------- */
const SAVE_KEY = 'pitbike_run_save_v1';
const defaultSave = {
  money: 0,
  bikes:   { green: {owned:true, equipped:true} },
  helmets: { none: {owned:true, equipped:true} },
  trails:  { default: {owned:true, equipped:true} },
  upgrades:{ tires:0, speed:0, brake:0, armor:0 }, // 0..15 each
  settings:{ quality:'med', perf:false, engine:60, tires:50 }
};
let save = loadSave();
function loadSave(){
  try { const s = JSON.parse(localStorage.getItem(SAVE_KEY));
    return s ? {...defaultSave, ...s, upgrades:{...defaultSave.upgrades,...(s.upgrades||{})},
      settings:{...defaultSave.settings,...(s.settings||{})}} : structuredClone(defaultSave);
  } catch(e){ return structuredClone(defaultSave); }
}
function persist(){ localStorage.setItem(SAVE_KEY, JSON.stringify(save)); }

/* ---------- CATALOG ---------- */
const CATALOG = {
  bikes: [
    { id:'green',   name:'Зелёный',    icon:'🏍️', color:'#4ade80', speed:1.00, price:0 },
    { id:'black',   name:'Чёрный',     icon:'🏍️', color:'#1f2937', speed:1.25, price:250 },
    { id:'orange',  name:'Оранжевый',  icon:'🏍️', color:'#fb923c', speed:0.85, price:120 },
    { id:'red',     name:'Красный',    icon:'🏍️', color:'#ef4444', speed:1.15, price:320 },
    { id:'blue',    name:'Синий',      icon:'🏍️', color:'#3b82f6', speed:1.10, price:280 },
    { id:'purple',  name:'Фиолетовый', icon:'🏍️', color:'#a855f7', speed:1.30, price:450 },
    { id:'gold',    name:'Золотой',    icon:'🏍️', color:'#fbbf24', speed:1.45, price:800 },
    { id:'white',   name:'Белый',      icon:'🏍️', color:'#e5e7eb', speed:0.95, price:180 },
  ],
  helmets: [
    { id:'none',   name:'Без шлема', icon:'👤', price:0 },
    { id:'basic',  name:'Обычный',   icon:'🪖', price:80 },
    { id:'sport',  name:'Спортивный',icon:'⛑️', price:220 },
    { id:'carbon', name:'Карбон',    icon:'🎩', price:500 },
  ],
  trails: [
    { id:'default', name:'Стандарт', icon:'〰️', color:'#555', price:0 },
    { id:'fire',    name:'Огонь',    icon:'🔥', color:'#ff5a2c', price:150 },
    { id:'ice',     name:'Лёд',      icon:'❄️', color:'#60a5fa', price:150 },
    { id:'neon',    name:'Неон',     icon:'✨', color:'#a855f7', price:260 },
    { id:'rainbow', name:'Радуга',   icon:'🌈', color:'rainbow', price:420 },
  ]
};

/* ---------- INPUT ---------- */
const keys = {};
addEventListener('keydown', e=>{
  keys[e.code] = true;
  if (e.code==='Escape') backToMenu();
});
addEventListener('keyup', e=> keys[e.code]=false);

/* camera rotation via middle mouse (wheel click) */
let camAngle = 0, camRotating=false, lastMX=0;
canvas.addEventListener('mousedown', e=>{ if(e.button===1){ camRotating=true; lastMX=e.clientX; e.preventDefault(); }});
addEventListener('mouseup', e=>{ if(e.button===1) camRotating=false; });
addEventListener('mousemove', e=>{ if(camRotating){ camAngle += (e.clientX-lastMX)*0.005; lastMX=e.clientX; }});

/* =========================================================
   GAME STATE
   ========================================================= */
let state = null; // current game state

function equippedBike(){
  const id = Object.keys(save.bikes).find(k=>save.bikes[k].equipped) || 'green';
  return CATALOG.bikes.find(b=>b.id===id);
}
function equippedTrail(){
  const id = Object.keys(save.trails).find(k=>save.trails[k].equipped) || 'default';
  return CATALOG.trails.find(t=>t.id===id);
}

function startGame(diff){
  closeOverlay('playOverlay');
  closeOverlay('menuOverlay');
  document.getElementById('hud').style.display='block';
  document.getElementById('moneyBox').style.display='block';

  const bike = equippedBike();
  const trail = equippedTrail();
  const upg = save.upgrades;

  // base stats with upgrades
  const maxSpeed = 210 * (1 + upg.speed*0.02);          // +~2% per level
  const accel    = 90  * bike.speed * (1 + upg.speed*0.03);
  const grip     = 2.2 + upg.tires*0.25;                 // tires → grip
  const brakePow = 120 + upg.brake*5;
  const maxHp    = 100 + upg.armor*3;

  const ratePerMeter = diff==='easy'?0.2 : diff==='medium'?0.5 : 1.0;

  state = {
    diff, ratePerMeter,
    bike, trail,
    x:0, y:0, angle:0, speed:0,
    maxSpeed, accel, grip, brakePow,
    hp: maxHp, maxHp,
    dist:0, money:0,
    cops:[], buildings:[], people:[], trees:[],
    trails:[],
    spawnTimer:0, copSpawnInterval: diff==='easy'?999 : diff==='medium'?8 : 4,
    maxCops: diff==='easy'?1 : diff==='medium'?5 : 8,
    time:0,
    over:false,
    camX:0, camY:0
  };

  // generate world
  generateWorld(diff);
  // spawn first cop
  if (diff!=='easy') spawnCop();

  requestAnimationFrame(loop);
}

function generateWorld(diff){
  const s = state;
  if (diff==='easy') return;

  // buildings
  const nBuild = diff==='medium'? 60 : 90;
  for (let i=0;i<nBuild;i++){
    const w = 60+Math.random()*120, h = 60+Math.random()*120;
    const x = (Math.random()-0.5)*6000;
    const y = (Math.random()-0.5)*6000;
    if (Math.hypot(x,y) < 200) continue; // keep spawn clear
    s.buildings.push({x,y,w,h, color:`hsl(${Math.random()*60+15},25%,${30+Math.random()*15}%)`});
  }
  // trees / parks (hard only)
  if (diff==='hard'){
    for (let i=0;i<140;i++){
      const x=(Math.random()-0.5)*6000, y=(Math.random()-0.5)*6000;
      if (Math.hypot(x,y)<200) continue;
      s.trees.push({x,y,r:14+Math.random()*10});
    }
    for (let i=0;i<40;i++){
      const x=(Math.random()-0.5)*6000, y=(Math.random()-0.5)*6000;
      s.people.push({x,y, vx:(Math.random()-0.5)*20, vy:(Math.random()-0.5)*20, t:Math.random()*10});
    }
  }
}

function spawnCop(){
  const s = state;
  if (s.cops.length >= s.maxCops) return;
  // spawn at random angle around player
  const a = Math.random()*Math.PI*2;
  const d = 700 + Math.random()*300;
  const baseSpeed = s.diff==='medium' ? 130 : 170;
  const speedBoost = Math.min(s.time/60, 1) * 40; // gets faster over time
  s.cops.push({
    x: s.x + Math.cos(a)*d,
    y: s.y + Math.sin(a)*d,
    angle:0, speed:0,
    maxSpeed: baseSpeed + speedBoost,
    accel: 70
  });
}

/* ---------- LOOP ---------- */
let lastT = performance.now();
function loop(t){
  const dt = Math.min(0.05, (t-lastT)/1000); lastT = t;
  if (!state || state.over) return;
  update(dt);
  render();
  requestAnimationFrame(loop);
}

function update(dt){
  const s = state;
  s.time += dt;

  // --- player controls ---
  let throttle = 0;
  if (keys['KeyW']) throttle += 1;
  if (keys['KeyS']) throttle -= 0.6;
  let steer = 0;
  if (keys['KeyA']) steer -= 1;
  if (keys['KeyD']) steer += 1;
  const braking = keys['Space'];

  // armor affects handling
  const hpRatio = s.hp/s.maxHp;
  let handling = 1;
  if (hpRatio < 0.5) handling = 0.75 + Math.random()*0.1;
  if (hpRatio < 0.35) handling = 0.4;

  // acceleration
  if (throttle>0) s.speed += s.accel * handling * dt;
  else if (throttle<0) s.speed += s.accel*0.5 * throttle * dt;

  // braking
  if (braking) s.speed -= s.brakePow * dt;
  // friction
  s.speed -= s.speed * 0.4 * dt;

  // armor: low hp causes random slowdowns / smoke-stops
  if (hpRatio < 0.35 && Math.random() < 0.003) s.speed *= 0.4;
  if (hpRatio < 0.5  && Math.random() < 0.002) s.speed *= 0.7;

  s.speed = Math.max(-s.maxSpeed*0.3, Math.min(s.maxSpeed, s.speed));

  // steering depends on speed & grip
  const steerFactor = Math.min(1, Math.abs(s.speed)/60);
  s.angle += steer * s.grip * steerFactor * handling * dt * (s.speed>=0?1:-1);

  // movement
  const vx = Math.cos(s.angle) * s.speed;
  const vy = Math.sin(s.angle) * s.speed;
  const nx = s.x + vx*dt;
  const ny = s.y + vy*dt;

  // collisions with buildings / trees / people
  let collided = false;
  for (const b of s.buildings){
    if (nx > b.x-b.w/2-14 && nx < b.x+b.w/2+14 &&
        ny > b.y-b.h/2-14 && ny < b.y+b.h/2+14){ collided = true; break; }
  }
  for (const tr of s.trees){
    if (Math.hypot(nx-tr.x, ny-tr.y) < tr.r+10) { collided = true; break; }
  }
  for (const p of s.people){
    if (Math.hypot(nx-p.x, ny-p.y) < 16) {
      // hit pedestrian → damage
      s.hp -= 8; p.hit=true; collided = true; break;
    }
  }
  if (!collided){ s.x = nx; s.y = ny; }
  else {
    s.hp -= Math.abs(s.speed)*0.05;
    s.speed *= -0.3;
  }

  // distance & money
  const moved = Math.hypot(vx*dt, vy*dt);
  s.dist += moved;
  s.money += moved * s.ratePerMeter / 100; // per meter

  // trail
  if (Math.abs(s.speed) > 5){
    s.trails.push({x:s.x, y:s.y, a:s.angle, life: 3, color:s.trail.color});
    if (s.trails.length>400) s.trails.shift();
  }
  for (const tr of s.trails) tr.life -= dt;
  s.trails = s.trails.filter(t=>t.life>0);

  // cops AI
  for (const c of s.cops){
    const dx = s.x-c.x, dy = s.y-c.y;
    const target = Math.atan2(dy,dx);
    let da = target - c.angle;
    while (da> Math.PI) da-=Math.PI*2;
    while (da<-Math.PI) da+=Math.PI*2;
    c.angle += Math.sign(da)*Math.min(Math.abs(da), 2.5*dt);
    c.speed += (c.speed<c.maxSpeed?1:-1) * c.accel * dt;
    c.speed = Math.max(0, Math.min(c.maxSpeed, c.speed));
    const cx = c.x + Math.cos(c.angle)*c.speed*dt;
    const cy = c.y + Math.sin(c.angle)*c.speed*dt;
    // cops avoid buildings roughly
    let blocked=false;
    for (const b of s.buildings){
      if (cx>b.x-b.w/2-12 && cx<b.x+b.w/2+12 && cy>b.y-b.h/2-12 && cy<b.y+b.h/2+12){ blocked=true; break; }
    }
    if (!blocked){ c.x=cx; c.y=cy; } else { c.angle += 0.6; }

    // catch?
    if (Math.hypot(s.x-c.x, s.y-c.y) < 22){
      gameOver(); return;
    }
  }

  // spawn more cops
  s.spawnTimer += dt;
  if (s.diff!=='easy' && s.spawnTimer > s.copSpawnInterval){
    s.spawnTimer = 0;
    spawnCop();
  }

  // people wander
  for (const p of s.people){
    p.t -= dt;
    if (p.t<=0){ p.vx=(Math.random()-0.5)*30; p.vy=(Math.random()-0.5)*30; p.t=2+Math.random()*4; }
    p.x += p.vx*dt; p.y += p.vy*dt;
  }

  // camera follows
  s.camX += (s.x - s.camX)*0.12;
  s.camY += (s.y - s.camY)*0.12;

  // HUD
  document.getElementById('hSpeed').textContent = Math.round(Math.abs(s.speed));
  document.getElementById('hDist').textContent  = Math.round(s.dist/10);
  document.getElementById('hMoney').textContent = s.money.toFixed(1);
  document.getElementById('hHp').textContent    = Math.round(s.hp);
  document.getElementById('moneyVal').textContent = (save.money + s.money).toFixed(1);

  // death by hp
  if (s.hp<=0){ gameOver(); }
}

function gameOver(){
  state.over = true;
  save.money += state.money;
  persist();
  document.getElementById('overDist').textContent = Math.round(state.dist/10);
  document.getElementById('overMoney').textContent = state.money.toFixed(1);
  document.getElementById('overOverlay').classList.remove('hidden');
  document.getElementById('hud').style.display='none';
  document.getElementById('moneyBox').style.display='none';
}

function backToMenu(){
  state = null;
  document.getElementById('overOverlay').classList.add('hidden');
  document.getElementById('hud').style.display='none';
  document.getElementById('moneyBox').style.display='none';
  document.getElementById('menuOverlay').classList.remove('hidden');
}

/* =========================================================
   RENDER (pseudo-3D top-down)
   ========================================================= */
function render(){
  const s = state;
  // background grass
  ctx.fillStyle = '#1b3a1f';
  ctx.fillRect(0,0,W,H);

  // world transform: translate to center, rotate by camAngle, then offset by -cam
  ctx.save();
  ctx.translate(W/2, H/2);
  ctx.rotate(-camAngle);
  ctx.translate(-s.camX, -s.camY);

  // grid
  const gs = 120;
  const x0 = Math.floor((s.camX-W)/gs)*gs;
  const y0 = Math.floor((s.camY-H)/gs)*gs;
  ctx.strokeStyle = 'rgba(255,255,255,.04)';
  ctx.lineWidth = 1;
  for (let x=x0; x<s.camX+W; x+=gs){
    ctx.beginPath(); ctx.moveTo(x, s.camY-H); ctx.lineTo(x, s.camY+H); ctx.stroke();
  }
  for (let y=y0; y<s.camY+H; y+=gs){
    ctx.beginPath(); ctx.moveTo(s.camX-W, y); ctx.lineTo(s.camX+W, y); ctx.stroke();
  }

  // trails
  for (const tr of s.trails){
    ctx.save();
    ctx.translate(tr.x, tr.y); ctx.rotate(tr.a);
    ctx.globalAlpha = Math.min(1, tr.life/3) * 0.6;
    ctx.fillStyle = tr.color==='rainbow'
      ? `hsl(${(performance.now()/10+tr.x)%360},90%,60%)` : tr.color;
    ctx.fillRect(-18,-3,6,6); ctx.fillRect(-18, -9, 6,6);
    ctx.restore();
  }
  ctx.globalAlpha = 1;

  // buildings (with 3D-ish shadow + side)
  for (const b of s.buildings){
    // shadow
    ctx.fillStyle = 'rgba(0,0,0,.35)';
    ctx.fillRect(b.x-b.w/2+8, b.y-b.h/2+10, b.w, b.h);
    // side (darker)
    ctx.fillStyle = shade(b.color,-25);
    ctx.fillRect(b.x-b.w/2, b.y-b.h/2+6, b.w, b.h);
    // top
    ctx.fillStyle = b.color;
    ctx.fillRect(b.x-b.w/2, b.y-b.h/2, b.w, b.h-6);
    // roof lines
    ctx.strokeStyle = 'rgba(0,0,0,.25)'; ctx.lineWidth=1;
    ctx.strokeRect(b.x-b.w/2, b.y-b.h/2, b.w, b.h-6);
  }

  // trees
  for (const t of s.trees){
    ctx.fillStyle='rgba(0,0,0,.35)';
    ctx.beginPath(); ctx.arc(t.x+4, t.y+5, t.r, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle='#2d5a2d';
    ctx.beginPath(); ctx.arc(t.x, t.y, t.r, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle='#3a7a3a';
    ctx.beginPath(); ctx.arc(t.x-3, t.y-3, t.r*0.7, 0, Math.PI*2); ctx.fill();
  }

  // people
  for (const p of s.people){
    ctx.fillStyle='rgba(0,0,0,.35)';
    ctx.beginPath(); ctx.arc(p.x+2, p.y+3, 6, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle = p.hit?'#ff6b6b':'#f5d0a9';
    ctx.beginPath(); ctx.arc(p.x, p.y, 6, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle='#3b82f6';
    ctx.fillRect(p.x-4, p.y+2, 8, 6);
  }

  // cops
  for (const c of s.cops){
    drawBike(c.x, c.y, c.angle, '#1e40af', '#60a5fa', true);
  }

  // player
  drawBike(s.x, s.y, s.angle, s.bike.color, '#222', false);

  // smoke if low hp
  const hpRatio = s.hp/s.maxHp;
  if (hpRatio < 0.5){
    for (let i=0;i<3;i++){
      ctx.fillStyle = `rgba(120,120,120,${0.3*(1-hpRatio)})`;
      ctx.beginPath();
      ctx.arc(s.x - Math.cos(s.angle)*18 + (Math.random()-0.5)*10,
              s.y - Math.sin(s.angle)*18 + (Math.random()-0.5)*10,
              4+Math.random()*6, 0, Math.PI*2);
      ctx.fill();
    }
  }

  ctx.restore();
}

function drawBike(x,y,angle,bodyC,darkC,isCop){
  ctx.save();
  ctx.translate(x,y); ctx.rotate(angle);
  // shadow
  ctx.fillStyle='rgba(0,0,0,.4)';
  ctx.fillRect(-16, -6+4, 32, 14);
  // body
  ctx.fillStyle = bodyC;
  ctx.fillRect(-16,-8,32,14);
  // side
  ctx.fillStyle = shade(bodyC,-30);
  ctx.fillRect(-16, 4, 32, 4);
  // wheels
  ctx.fillStyle = darkC;
  ctx.fillRect(-14,-10,6,4);
  ctx.fillRect(8,-10,6,4);
  ctx.fillRect(-14, 8, 6, 4);
  ctx.fillRect(8, 8, 6, 4);
  // rider
  ctx.fillStyle = '#111';
  ctx.beginPath(); ctx.arc(0,-2,5,0,Math.PI*2); ctx.fill();
  if (isCop){
    // flashing light
    ctx.fillStyle = (Math.floor(performance.now()/200)%2)?'#ef4444':'#3b82f6';
    ctx.fillRect(-3,-14,6,4);
  }
  ctx.restore();
}

function shade(hex, percent){
  // hex like #rrggbb or hsl
  if (hex.startsWith('hsl')) return hex;
  const n = parseInt(hex.slice(1),16);
  let r=(n>>16)&255, g=(n>>8)&255, b=n&255;
  r=Math.max(0,Math.min(255, r+percent));
  g=Math.max(0,Math.min(255, g+percent));
  b=Math.max(0,Math.min(255, b+percent));
  return '#'+((r<<16)|(g<<8)|b).toString(16).padStart(6,'0');
}

/* =========================================================
   UI LOGIC
   ========================================================= */
function openPlay(){ document.getElementById('playOverlay').classList.remove('hidden'); }
function openShop(){ renderShop('bikes'); document.getElementById('shopOverlay').classList.remove('hidden'); }
function openUpgrades(){ renderUpgrades(); document.getElementById('upgOverlay').classList.remove('hidden'); }
function openSettings(){
  document.getElementById('setQuality').value = save.settings.quality;
  document.getElementById('setPerf').checked  = save.settings.perf;
  document.getElementById('setEngine').value  = save.settings.engine;
  document.getElementById('setTires').value   = save.settings.tires;
  document.getElementById('settingsOverlay').classList.remove('hidden');
}
function closeOverlay(id){ document.getElementById(id).classList.add('hidden'); }

/* ---------- SHOP ---------- */
document.getElementById('shopTabs').addEventListener('click', e=>{
  if (!e.target.classList.contains('tab')) return;
  document.querySelectorAll('#shopTabs .tab').forEach(t=>t.classList.remove('active'));
  e.target.classList.add('active');
  renderShop(e.target.dataset.tab);
});

function renderShop(tab){
  const grid = document.getElementById('shopGrid');
  grid.innerHTML = '';
  CATALOG[tab].forEach(item=>{
    const inv = save[tab][item.id] || {owned:false, equipped:false};
    const card = document.createElement('div');
    card.className = 'card' + (inv.owned?' owned':'') + (inv.equipped?' equipped':'');
    let extra = '';
    if (tab==='bikes') extra = `<div class="speed">Скорость ×${item.speed.toFixed(2)}</div>`;
    card.innerHTML = `
      <div class="icon" style="color:${item.color||'#fff'}">${item.icon}</div>
      <div class="name">${item.name}</div>
      ${extra}
      <div class="price">${inv.owned ? (inv.equipped?'✅ Выбрано':'Нажми чтобы выбрать') : item.price+' ₽'}</div>
    `;
    card.onclick = ()=>{
      if (!inv.owned){
        const total = save.money + (state?state.money:0);
        if (total < item.price){ alert('Недостаточно средств!'); return; }
        // spend: if game running, from running money first
        let need = item.price;
        if (state){ const take = Math.min(state.money, need); state.money -= take; need -= take; }
        save.money -= need;
        save[tab][item.id] = {owned:true, equipped:false};
        // unequip others
        for (const k in save[tab]) if (k!==item.id) save[tab][k].equipped=false;
        save[tab][item.id].equipped = true;
        persist();
      } else {
        for (const k in save[tab]) save[tab][k].equipped=false;
        save[tab][item.id].equipped = true;
        persist();
      }
      renderShop(tab);
    };
    grid.appendChild(card);
  });
}

/* ---------- UPGRADES ---------- */
const UPG_META = {
  tires: { name:'Шины',     icon:'🛞', desc:'+1 к сцеплению за уровень', base:40 },
  speed: { name:'Скорость', icon:'⚡', desc:'+10 км/ч к макс. скорости', base:60 },
  brake: { name:'Торможение', icon:'🛑', desc:'+5 к мощности тормоза', base:50 },
  armor: { name:'Прочность',icon:'🛡️', desc:'+3 к максимальному здоровью', base:70 }
};
function upgPrice(key, lvl){
  // +0.1 каждый уровень? в ТЗ: "+0.1 каждый раз" — интерпретируем как +10% к базовой цене за уровень
  return Math.round(UPG_META[key].base * (1 + lvl*0.1));
}
function renderUpgrades(){
  const list = document.getElementById('upgList');
  list.innerHTML = '';
  for (const k in UPG_META){
    const m = UPG_META[k];
    const lvl = save.upgrades[k];
    const price = upgPrice(k, lvl);
    const max = lvl>=15;
    const row = document.createElement('div');
    row.className = 'row';
    row.innerHTML = `
      <div>
        <div style="font-size:18px">${m.icon} <b>${m.name}</b> — ур. ${lvl}/15</div>
        <div style="font-size:12px; opacity:.7">${m.desc}</div>
        <div class="bar" style="width:220px"><div style="width:${(lvl/15)*100}%"></div></div>
      </div>
      <div style="text-align:right">
        <div style="color:#ffd86b; margin-bottom:6px">${max?'MAX':price+' ₽'}</div>
        <button class="btn" style="margin:0; max-width:140px; padding:8px" ${max?'disabled':''}>
          ${max?'Максимум':'Улучшить'}
        </button>
      </div>
    `;
    row.querySelector('button').onclick = ()=>{
      if (max) return;
      const total = save.money + (state?state.money:0);
      if (total < price){ alert('Недостаточно средств!'); return; }
      let need = price;
      if (state){ const take = Math.min(state.money, need); state.money -= take; need -= take; }
      save.money -= need;
      save.upgrades[k]++;
      persist();
      renderUpgrades();
    };
    list.appendChild(row);
  }
}

/* ---------- SETTINGS ---------- */
['setQuality','setPerf','setEngine','setTires'].forEach(id=>{
  document.getElementById(id).addEventListener('change', ()=>{
    save.settings.quality = document.getElementById('setQuality').value;
    save.settings.perf    = document.getElementById('setPerf').checked;
    save.settings.engine  = +document.getElementById('setEngine').value;
    save.settings.tires   = +document.getElementById('setTires').value;
    persist();
  });
});

function resetProgress(){
  if (!confirm('Точно сбросить весь прогресс?')) return;
  localStorage.removeItem(SAVE_KEY);
  save = loadSave();
  alert('Прогресс сброшен.');
}

/* ---------- BG animation for menu (mini demo) ---------- */
let demo = { x:0, y:0, a:0, t:0 };
function menuBgLoop(){
  if (!document.getElementById('menuOverlay').classList.contains('hidden') && !state){
    ctx.fillStyle='#1b3a1f'; ctx.fillRect(0,0,W,H);
    demo.t += 0.016;
    demo.x = Math.cos(demo.t*0.5)*400;
    demo.y = Math.sin(demo.t*0.7)*300;
    demo.a = demo.t*1.2;
    // grid
    ctx.strokeStyle='rgba(255,255,255,.05)';
    for (let x=-W; x<W*2; x+=100){ ctx.beginPath(); ctx.moveTo(x,0); ctx.lineTo(x,H); ctx.stroke(); }
    for (let y=-H; y<H*2; y+=100){ ctx.beginPath(); ctx.moveTo(0,y); ctx.lineTo(W,y); ctx.stroke(); }
    // fake buildings
    for (let i=0;i<8;i++){
      const bx = (i*230-800 - demo.x*0.3)%W;
      ctx.fillStyle='rgba(0,0,0,.3)'; ctx.fillRect(bx+6,100+6,120,160);
      ctx.fillStyle='#3a4a3a'; ctx.fillRect(bx,100,120,160);
    }
    // bike
    drawBike(W/2+demo.x*0.2, H/2+demo.y*0.2, demo.a, '#ff7a3d', '#222', false);
    // cop chasing
    drawBike(W/2+demo.x*0.2 - Math.cos(demo.a)*80,
             H/2+demo.y*0.2 - Math.sin(demo.a)*80,
             demo.a, '#1e40af', '#60a5fa', true);
  }
  requestAnimationFrame(menuBgLoop);
}
menuBgLoop();
</script>
</body>
</html>
