# space-blaster
Space Blaster is a lightweight HTML5 arcade shooter built with pure JavaScript and Canvas. Dodge and blast endless waves of enemies, track your score, and play instantly in any browser. Works offline, supports keyboard and touch, and is free under the MIT License.
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Space Blaster — Downloadable HTML5 Game</title>
  <style>
    :root{--bg:#0b1220;--fg:#e6f0ff;--accent:#4ee1a3}
    html,body{height:100%;margin:0;background:var(--bg);color:var(--fg);font-family:Inter, system-ui, Arial}
    #gameWrap{display:flex;align-items:center;justify-content:center;height:100%;}
    canvas{border-radius:12px;box-shadow:0 8px 30px rgba(0,0,0,.6);background:linear-gradient(180deg,#06101a 0%, #071420 100%)}
    .ui{position:absolute;left:16px;top:16px;color:var(--fg);font-size:14px}
    .bottomUi{position:absolute;left:16px;bottom:16px;color:var(--fg);font-size:13px}
    .btn{background:rgba(255,255,255,.06);padding:8px 10px;border-radius:8px;border:1px solid rgba(255,255,255,.04);cursor:pointer}
    a{color:var(--accent)}
    .controls{display:flex;gap:8px;align-items:center}
    #overlay{position:fixed;inset:0;display:flex;align-items:center;justify-content:center;pointer-events:none}
  </style>
</head>
<body>
  <div id="gameWrap">
    <canvas id="game"></canvas>
    <div class="ui" id="hud">Score: 0 &nbsp; Lives: 3 &nbsp; Wave: 1</div>
    <div class="bottomUi">Keys: ← → to move, Space to shoot, P pause/resume — Touch: tap left/right to move, tap top-right to shoot</div>
    <div id="overlay"></div>
  </div>

<script>
/*
  Space Blaster
  - Single-file downloadable HTML5 game
  - No external assets: everything drawn on canvas
  - Save highscore via localStorage
  - Simple arcade loop with enemies waves, player bullets, collisions
*/

const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
let DPR = Math.max(1, window.devicePixelRatio || 1);

// Resize canvas responsively
function resize(){
  const w = Math.min(window.innerWidth - 40, 900);
  const h = Math.min(window.innerHeight - 120, 700);
  canvas.style.width = w + 'px';
  canvas.style.height = h + 'px';
  canvas.width = Math.floor(w * DPR);
  canvas.height = Math.floor(h * DPR);
  ctx.setTransform(DPR,0,0,DPR,0,0);
}
window.addEventListener('resize', resize);
resize();

// Utility functions
const rand = (a,b)=> Math.random()*(b-a)+a;
const clamp = (v,a,b)=> Math.max(a, Math.min(b,v));
const dist2 = (a,b)=>{const dx=a.x-b.x,dy=a.y-b.y;return dx*dx+dy*dy}

// Game state
let state = {
  running:true,
  paused:false,
  score:0,
  highscore: parseInt(localStorage.getItem('sb_high')||0,10),
  lives:3,
  wave:1,
  time:0
};

// Player
const player = {
  x: 200, y: 520, w: 32, h: 28, speed: 260, cooldown:0
};

// Input
const input = {left:false,right:false,shoot:false};
window.addEventListener('keydown', e=>{
  if(e.key==='ArrowLeft'||e.key==='a') input.left=true;
  if(e.key==='ArrowRight'||e.key==='d') input.right=true;
  if(e.key===' '||e.key==='Spacebar') input.shoot=true;
  if(e.key==='p'||e.key==='P') togglePause();
});
window.addEventListener('keyup', e=>{
  if(e.key==='ArrowLeft'||e.key==='a') input.left=false;
  if(e.key==='ArrowRight'||e.key==='d') input.right=false;
  if(e.key===' '||e.key==='Spacebar') input.shoot=false;
});

// Touch controls
canvas.addEventListener('touchstart', e=>{
  e.preventDefault();
  const t = e.touches[0];
  const rect = canvas.getBoundingClientRect();
  const x = (t.clientX-rect.left);
  const y = (t.clientY-rect.top);
  if(x > rect.width * 0.85 && y < rect.height*0.3) input.shoot = true;
  else{
    // left or right half
    if(x < rect.width/2) { input.left = true; input.right=false; }
    else { input.right = true; input.left = false; }
  }
});
canvas.addEventListener('touchend', e=>{ e.preventDefault(); input.left=false; input.right=false; input.shoot=false; });

// Entities
let bullets = [];
let enemies = [];
let enemyBullets = [];

function spawnWave(n){
  const cols = Math.min(8, Math.ceil(n/2));
  const rows = Math.ceil(n/cols);
  const margin = 60;
  for(let r=0;r<rows;r++){
    for(let c=0;c<cols;c++){
      const ex = margin + c*( (canvas.width/DPR - margin*2)/(cols) ) + rand(-8,8);
      const ey = 40 + r*40 + rand(-6,6);
      enemies.push({x:ex,y:ey,w:28,h:22,spd:20 + state.wave*6,hp:1 + Math.floor(state.wave/3),dir:1});
    }
  }
}

function resetGame(){
  bullets=[]; enemies=[]; enemyBullets=[];
  state.score=0; state.lives=3; state.wave=1; state.time=0; state.paused=false; state.running=true;
  player.x = (canvas.width/DPR)/2; player.y = (canvas.height/DPR)-80;
  spawnWave(6 + state.wave*2);
  updateHud();
}

function nextWave(){
  state.wave++;
  spawnWave(6 + state.wave*2);
}

function togglePause(){ state.paused = !state.paused; updateHud(); }

// Game loop
let last = performance.now();
function loop(now){
  const dt = Math.min(0.05, (now - last)/1000);
  last = now;
  if(!state.paused && state.running) update(dt);
  render();
  requestAnimationFrame(loop);
}
requestAnimationFrame(loop);

function update(dt){
  state.time += dt;
  // Player movement
  if(input.left) player.x -= player.speed*dt;
  if(input.right) player.x += player.speed*dt;
  player.x = clamp(player.x, 24, (canvas.width/DPR)-24);

  // Shooting
  player.cooldown -= dt;
  if(input.shoot && player.cooldown <= 0){
    bullets.push({x:player.x,y:player.y-18,vy:-520,w:6,h:10});
    player.cooldown = 0.18; // fire rate
  }

  // bullets
  for(let i=bullets.length-1;i>=0;i--){
    bullets[i].y += bullets[i].vy*dt;
    if(bullets[i].y < -20) bullets.splice(i,1);
  }

  // enemies
  for(let i=enemies.length-1;i>=0;i--){
    const e = enemies[i];
    // simple side-to-side sweep
    e.x += Math.sin((state.time + i)*0.6)*e.spd*dt;
    // occasional shooting
    if(Math.random() < 0.006 + state.wave*0.0008){
      enemyBullets.push({x:e.x,y:e.y+14,vy:160 + state.wave*10,w:6,h:10});
    }
    // collision with player bullets
    for(let b=bullets.length-1;b>=0;b--){
      if(collidePointRect(bullets[b], e)){
        bullets.splice(b,1);
        e.hp--;
        if(e.hp <= 0){
          state.score += 10 + Math.floor(state.wave*2);
          enemies.splice(i,1);
          break;
        }
      }
    }
    // enemies reach bottom -> lose life
    if(e.y > (canvas.height/DPR) - 80){
      enemies.splice(i,1);
      state.lives--;
      updateHud();
      if(state.lives <= 0) gameOver();
    }
  }

  // enemy bullets
  for(let i=enemyBullets.length-1;i>=0;i--){
    enemyBullets[i].y += enemyBullets[i].vy*dt;
    if(collidePointRect(enemyBullets[i], player)){
      enemyBullets.splice(i,1);
      state.lives--;
      updateHud();
      if(state.lives <= 0) gameOver();
    } else if(enemyBullets[i].y > (canvas.height/DPR)+40) enemyBullets.splice(i,1);
  }

  // check if wave cleared
  if(enemies.length === 0){ nextWave(); updateHud(); }

  // update HUD occasionally
  if(Math.random() < dt*3) updateHud();
}

function collidePointRect(p, r){
  // p has x,y,w,h - treat as rect
  return !(p.x + (p.w||0) < r.x - r.w/2 || p.x > r.x + r.w/2 || p.y + (p.h||0) < r.y - r.h/2 || p.y > r.y + r.h/2);
}

function gameOver(){
  state.running = false;
  state.paused = true;
  if(state.score > state.highscore){
    state.highscore = state.score;
    localStorage.setItem('sb_high', String(state.highscore));
  }
  showOverlay(`<div style="background:rgba(10,14,20,.92);padding:18px;border-radius:12px;color:var(--fg);text-align:center">`+
              `<h2 style="margin:6px 0">Game Over</h2><p>Score: ${state.score} • Highscore: ${state.highscore}</p>`+
              `<div style="display:flex;gap:8px;justify-content:center;margin-top:10px">`+
              `<button class='btn' onclick='restart()'>Play Again</button>`+
              `<button class='btn' onclick='downloadBuild()'>Download Game (zip)</button>`+
              `</div></div>`);
}

function restart(){
  hideOverlay();
  resetGame();
}

function downloadBuild(){
  // Create a downloadable zip containing this HTML file. We'll synthesize a basic zip by creating a blob.
  // To keep it simple, we provide the single-file HTML for user to save.
  const html = document.documentElement.outerHTML;
  const blob = new Blob([html], {type:'text/html'});
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url; a.download = 'space-blaster.html'; a.click();
  URL.revokeObjectURL(url);
}

function showOverlay(html){
  const o = document.getElementById('overlay');
  o.style.pointerEvents='auto';
  o.innerHTML = html;
}
function hideOverlay(){ const o=document.getElementById('overlay'); o.style.pointerEvents='none'; o.innerHTML=''; }

// Rendering
function render(){
  const W = canvas.width/DPR, H = canvas.height/DPR;
  ctx.clearRect(0,0,canvas.width,canvas.height);

  // starfield background
  renderStars(ctx, W, H);

  // hull shadow
  roundRect(ctx, 10, 10, W-20, H-20, 12);

  // player
  drawPlayer(ctx, player.x, player.y);

  // bullets
  bullets.forEach(b=> drawBullet(ctx, b.x, b.y));
  enemyBullets.forEach(b=> drawEnemyBullet(ctx,b.x,b.y));

  // enemies
  enemies.forEach(e=> drawEnemy(ctx,e.x,e.y,e.hp));

  // HUD text
  // draw small score top-right
  ctx.save();
  ctx.font = '12px Inter, Arial'; ctx.fillStyle='rgba(230,240,255,.85)';
  ctx.textAlign='right'; ctx.fillText(`HS: ${state.highscore}`, W-18, 20);
  ctx.restore();
}

// Drawing helpers
function drawPlayer(ctx,x,y){
  ctx.save();
  ctx.translate(x,y);
  // ship body
  ctx.beginPath();
  ctx.moveTo(0,-16); ctx.lineTo(14,8); ctx.lineTo(8,10); ctx.lineTo(0,2); ctx.lineTo(-8,10); ctx.lineTo(-14,8); ctx.closePath();
  gradientFill(ctx, -16, -16, 32, 36);
  ctx.strokeStyle='rgba(255,255,255,.08)'; ctx.lineWidth=1; ctx.stroke();
  // cockpit
  ctx.beginPath(); ctx.ellipse(0,-4,6,4,0,0,Math.PI*2); ctx.fillStyle='rgba(10,18,28,.9)'; ctx.fill();
  ctx.restore();
}
function drawBullet(ctx,x,y){ ctx.save(); ctx.fillStyle='#bff5ff'; ctx.fillRect(x-3,y-8,6,12); ctx.restore(); }
function drawEnemyBullet(ctx,x,y){ ctx.save(); ctx.fillStyle='#ffb2b2'; ctx.fillRect(x-3,y-2,6,12); ctx.restore(); }
function drawEnemy(ctx,x,y,hp){ ctx.save(); ctx.translate(x,y); ctx.beginPath(); ctx.rect(-14,-10,28,20); ctx.fillStyle='rgba(220,80,120,.95)'; ctx.fill(); ctx.fillStyle='rgba(0,0,0,.18)'; ctx.fillRect(-12,-6,24,4); // detail
  // health bar
  ctx.fillStyle='#222'; ctx.fillRect(-14, -16, 28, 4);
  ctx.fillStyle = hp>1? '#4ee1a3' : '#ffd76b'; ctx.fillRect(-14, -16, (28)*(hp>2?1:hp/2), 4);
  ctx.restore(); }

function gradientFill(ctx,x,y,w,h){
  const g = ctx.createLinearGradient(x,y,x+w,y+h);
  g.addColorStop(0,'#7be5ff'); g.addColorStop(1,'#6ad6ff');
  ctx.fillStyle = g; ctx.fill();
}

function roundRect(ctx,x,y,w,h,r){ ctx.save(); ctx.fillStyle='rgba(255,255,255,0.02)'; ctx.beginPath(); ctx.moveTo(x+r,y); ctx.arcTo(x+w,y,x+w,y+h,r); ctx.arcTo(x+w,y+h,x,y+h,r); ctx.arcTo(x,y+h,x,y,r); ctx.arcTo(x,y,x+w,y,r); ctx.closePath(); ctx.fill(); ctx.restore(); }

// Starfield
let stars = [];
function makeStars(){ stars = []; const W = canvas.width/DPR, H = canvas.height/DPR; for(let i=0;i<120;i++){ stars.push({x:rand(0,W), y:rand(0,H), s:rand(0.4,1.4)}) } }
makeStars(); window.addEventListener('resize', makeStars);
function renderStars(ctx,W,H){ ctx.save(); ctx.fillStyle='black'; ctx.fillRect(0,0,W,H); for(const s of stars){ ctx.fillStyle = `rgba(255,255,255,${0.06 + s.s*0.2})`; ctx.fillRect(s.x, s.y + Math.sin(state.time + s.x*0.01)*2, s.s, s.s); } ctx.restore(); }

// HUD updater
function updateHud(){ document.getElementById('hud').innerHTML = `Score: ${state.score} &nbsp; Lives: ${state.lives} &nbsp; Wave: ${state.wave}`; }
updateHud();

// Simple menu on first load
showOverlay(`<div style="background:rgba(10,14,20,.92);padding:18px;border-radius:12px;color:var(--fg);text-align:center">`+
            `<h1 style="margin:6px 0">Space Blaster</h1><p style='margin:6px 0 10px 0'>A small, single-file HTML5 arcade shooter — download and play anywhere.</p>`+
            `<div style="display:flex;gap:8px;justify-content:center">`+
            `<button class='btn' onclick='start()'>Start Game</button>`+
            `<button class='btn' onclick='showHow()'>How to Download</button>`+
            `</div></div>`);

function start(){ hideOverlay(); resetGame(); }
function showHow(){ hideOverlay(); showOverlay(`<div style="background:rgba(10,14,20,.92);padding:18px;border-radius:12px;color:var(--fg);text-align:left;max-width:520px">`+
  `<h3 style='margin:4px 0'>How to download</h3>`+
  `<ol><li>Click <b>Download Game (zip)</b> in the Game Over screen or use the button below to save <code>space-blaster.html</code>.</li>`+
  `<li>Save the file and open it in any modern browser (Chrome, Edge, Firefox).</li>`+
  `<li>To make a desktop app: use <code>Electron</code> or <code>nativefier</code> — see instructions below.</li></ol>`+
  `<pre style='background:#04121a;padding:8px;border-radius:8px;color:#bfe8d7;overflow:auto'>`+
  `# Quick Electron package (very short):\n`+
  `mkdir sb && cd sb\n`+
  `# copy space-blaster.html into this folder as index.html\n`+
  `npm init -y\n`+
  `npm install electron --save-dev\n`+
  `# create main.js with simple BrowserWindow to load index.html then run: npx electron .\n`+
  `# For building use electron-packager or electron-builder (platform-specific)\n`+
  `</pre>`+
  `<div style='display:flex;gap:8px;justify-content:center;margin-top:10px'>`+
  `<button class='btn' onclick='downloadBuild()'>Save HTML</button>`+
  `<button class='btn' onclick='hideOverlay()'>Close</button>`+
  `</div></div>`);
}

// Expose some functions for inline buttons
window.restart = restart; window.downloadBuild = downloadBuild; window.start = start; window.showHow = showHow; window.togglePause = togglePause;

// initialize
(function(){ player.x = (canvas.width/DPR)/2; player.y = (canvas.height/DPR)-80; })();

</script>
</body>
</html>
