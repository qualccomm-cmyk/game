<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>2D Purple Shadow Platformer</title>
  <style>
    * {
      box-sizing: border-box;
      margin: 0;
      padding: 0;
      user-select: none;
      -webkit-user-select: none;
      touch-action: none;
    }
    body {
      background-color: #08030f;
      color: #fff;
      font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      min-height: 100vh;
      overflow: hidden;
    }
    #game-container {
      position: relative;
      width: 100vw;
      max-width: 960px;
      height: 540px;
      background: #12061c;
      border: 3px solid #7209b7;
      box-shadow: 0 0 30px rgba(114, 9, 183, 0.7);
      overflow: hidden;
      border-radius: 8px;
    }
    canvas {
      display: block;
      width: 100%;
      height: 100%;
    }
    
    /* FIX: PERBAIKAN TOMBOL KONTROL HP (LEBIH BESAR & MUDAH DITEKAN) */
    #touch-controls {
      position: absolute;
      bottom: 15px;
      left: 0;
      width: 100%;
      display: flex;
      justify-content: space-between;
      padding: 0 20px;
      pointer-events: none;
      z-index: 10;
    }
    .btn-group {
      display: flex;
      gap: 12px;
      pointer-events: auto;
    }
    .btn {
      width: 75px;
      height: 75px;
      background: rgba(35, 8, 60, 0.85);
      border: 3px solid #b5179e;
      border-radius: 50%;
      color: #ffffff;
      font-weight: 900;
      font-size: 16px;
      display: flex;
      align-items: center;
      justify-content: center;
      box-shadow: 0 0 15px rgba(181, 23, 158, 0.6), inset 0 0 10px rgba(247, 37, 133, 0.4);
      backdrop-filter: blur(5px);
      text-shadow: 0 0 8px #f72585;
      transition: transform 0.05s ease;
    }
    .btn:active, .btn.pressed {
      background: rgba(247, 37, 133, 0.9);
      border-color: #4cc9f0;
      transform: scale(0.90);
      box-shadow: 0 0 25px rgba(247, 37, 133, 1);
    }
    .btn-action {
      background: rgba(114, 9, 183, 0.85);
      border-color: #f72585;
    }
    .btn-dash {
      background: rgba(76, 201, 240, 0.3);
      border-color: #4cc9f0;
    }
  </style>
</head>
<body>

<div id="game-container">
  <canvas id="gameCanvas" width="960" height="540"></canvas>
  
  <!-- Controller Layar Sentuh HP -->
  <div id="touch-controls">
    <div class="btn-group">
      <div class="btn" id="btn-left">◄</div>
      <div class="btn" id="btn-right">►</div>
    </div>
    <div class="btn-group">
      <div class="btn btn-dash" id="btn-dash">DASH</div>
      <div class="btn btn-action" id="btn-attack">ATK</div>
      <div class="btn btn-action" id="btn-jump">JUMP</div>
    </div>
  </div>
</div>

<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

// Memuat Gambar Asset
const assets = {
  heroine: new Image(),
  monsters: new Image(),
  towers: new Image(),
  platforms: new Image()
};

assets.heroine.src = 'heroine.png';
assets.monsters.src = 'monsters.png';
assets.towers.src = 'towers.png';
assets.platforms.src = 'platforms.png';

// Status Tombol
const keys = { left: false, right: false, jump: false };

// Kelas Animasi Sprite Sheet (Memotong Frame Gambar)
class SpriteAnimator {
  constructor(image, cols, rows) {
    this.image = image;
    this.cols = cols;
    this.rows = rows;
    this.frameIndex = 0;
    this.tickCount = 0;
  }

  drawFrame(ctx, anim, x, y, width, height, facingLeft) {
    if (!this.image.complete || this.image.naturalWidth === 0) {
      // Fallback jika gambar belum termuat
      ctx.fillStyle = '#b5179e';
      ctx.fillRect(x, y, width, height);
      return;
    }

    const frameW = this.image.naturalWidth / this.cols;
    const frameH = this.image.naturalHeight / this.rows;

    // Hitung Frame Animasi
    this.tickCount++;
    if (this.tickCount >= anim.speed) {
      this.tickCount = 0;
      if (this.frameIndex < anim.start || this.frameIndex > anim.end) {
        this.frameIndex = anim.start;
      } else {
        if (this.frameIndex < anim.end) {
          this.frameIndex++;
        } else if (anim.loop !== false) {
          this.frameIndex = anim.start;
        }
      }
    }

    let col = this.frameIndex % this.cols;
    let row = anim.row;

    let sx = col * frameW;
    let sy = row * frameH;

    ctx.save();
    if (facingLeft) {
      ctx.translate(x + width, y);
      ctx.scale(-1, 1);
      ctx.drawImage(this.image, sx, sy, frameW, frameH, 0, 0, width, height);
    } else {
      ctx.drawImage(this.image, sx, sy, frameW, frameH, x, y, width, height);
    }
    ctx.restore();
  }
}

// Inisialisasi Animator
const heroAnimator = new SpriteAnimator(assets.heroine, 10, 5);
const monsterAnimator = new SpriteAnimator(assets.monsters, 10, 6);

// Definisi Animasi Heroine
const HERO_ANIMS = {
  idle:   { row: 0, start: 0, end: 3, speed: 10 },
  run:    { row: 0, start: 4, end: 9, speed: 5 },
  dash:   { row: 1, start: 0, end: 3, speed: 3 },
  attack: { row: 1, start: 4, end: 8, speed: 3 },
  jump:   { row: 3, start: 0, end: 3, speed: 6 },
  death:  { row: 4, start: 0, end: 6, speed: 10, loop: false }
};

// Definisi Animasi Musuh (Reaper Skeleton)
const REAPER_ANIMS = {
  walk:   { row: 0, start: 0, end: 7, speed: 8 },
  attack: { row: 1, start: 0, end: 7, speed: 6 },
  death:  { row: 2, start: 3, end: 6, speed: 10, loop: false }
};

// Objek Karakter Utama
const player = {
  x: 100,
  y: 300,
  width: 60,
  height: 75,
  vx: 0,
  vy: 0,
  speed: 4.5,
  jumpPower: -12,
  gravity: 0.55,
  isGrounded: false,
  facing: 'right',
  hp: 100,
  maxHp: 100,
  state: 'idle', // idle, run, jump, dash, attack, dead
  attackTimer: 0,
  dashTimer: 0,
  invulnerable: false
};

// Pijakan/Platforms
const platforms = [
  { x: 0, y: 460, width: 960, height: 80 }, // Tanah Utama
  { x: 180, y: 340, width: 180, height: 30 },
  { x: 440, y: 250, width: 200, height: 30 },
  { x: 720, y: 350, width: 180, height: 30 }
];

// Menara Tower
const towers = [
  { x: 760, y: 230, width: 80, height: 120, hp: 120, maxHp: 120, shootTimer: 0 }
];

// Musuh Monster Tengkorak
const enemies = [
  { x: 220, y: 280, width: 55, height: 60, hp: 60, maxHp: 60, vx: 1.2, minX: 180, maxX: 320, dir: 1, state: 'walk', isDead: false },
  { x: 500, y: 400, width: 55, height: 60, hp: 80, maxHp: 80, vx: 1.5, minX: 450, maxX: 680, dir: 1, state: 'walk', isDead: false }
];

// Proyektil Tembakan Tower
const projectiles = [];

// Input Keyboard
window.addEventListener('keydown', (e) => {
  if (player.state === 'dead') return;
  if (e.key === 'ArrowLeft' || e.key === 'a') keys.left = true;
  if (e.key === 'ArrowRight' || e.key === 'd') keys.right = true;
  if ((e.key === 'ArrowUp' || e.key === 'w' || e.key === ' ') && player.isGrounded) player.vy = player.jumpPower;
  if (e.key === 'j' || e.key === 'z') triggerAttack();
  if (e.key === 'k' || e.key === 'x') triggerDash();
});

window.addEventListener('keyup', (e) => {
  if (e.key === 'ArrowLeft' || e.key === 'a') keys.left = false;
  if (e.key === 'ArrowRight' || e.key === 'd') keys.right = false;
});

// Event Layar Sentuh HP
function bindTouchButton(id, onStart, onEnd) {
  const btn = document.getElementById(id);
  
  const handleStart = (e) => {
    if (e.cancelable) e.preventDefault();
    btn.classList.add('pressed');
    onStart();
  };
  
  const handleEnd = (e) => {
    if (e.cancelable) e.preventDefault();
    btn.classList.remove('pressed');
    if (onEnd) onEnd();
  };

  btn.addEventListener('touchstart', handleStart, { passive: false });
  btn.addEventListener('touchend', handleEnd, { passive: false });
  btn.addEventListener('touchcancel', handleEnd, { passive: false });
  btn.addEventListener('mousedown', handleStart);
  btn.addEventListener('mouseup', handleEnd);
  btn.addEventListener('mouseleave', handleEnd);
}

bindTouchButton('btn-left', () => keys.left = true, () => keys.left = false);
bindTouchButton('btn-right', () => keys.right = true, () => keys.right = false);
bindTouchButton('btn-jump', () => { if (player.isGrounded && player.state !== 'dead') player.vy = player.jumpPower; });
bindTouchButton('btn-attack', () => triggerAttack());
bindTouchButton('btn-dash', () => triggerDash());

// Aksi Serangan (Attack)
function triggerAttack() {
  if (player.state === 'dead' || player.state === 'attack') return;
  player.state = 'attack';
  player.attackTimer = 18;
  
  const hitWidth = 70;
  const hitX = player.facing === 'right' ? player.x + player.width : player.x - hitWidth;
  const attackBox = { x: hitX, y: player.y - 10, width: hitWidth, height: player.height + 20 };

  // Serang Musuh
  enemies.forEach(enemy => {
    if (!enemy.isDead && checkCollision(attackBox, enemy)) {
      enemy.hp -= 30;
      if (enemy.hp <= 0) {
        enemy.hp = 0;
        enemy.isDead = true;
        enemy.state = 'death';
      }
    }
  });

  // Serang Tower
  towers.forEach(tower => {
    if (tower.hp > 0 && checkCollision(attackBox, tower)) {
      tower.hp -= 25;
      if (tower.hp < 0) tower.hp = 0;
    }
  });
}

// Aksi Dash
function triggerDash() {
  if (player.state === 'dead' || player.state === 'dash') return;
  player.state = 'dash';
  player.dashTimer = 14;
  player.invulnerable = true;
}

// Cek Tabrakan Bounding Box
function checkCollision(r1, r2) {
  return !(r1.x > r2.x + r2.width ||
           r1.x + r1.width < r2.x ||
           r1.y > r2.y + r2.height ||
           r1.y + r1.height < r2.y);
}

// Gambar Badge Darah (Health Badge Overhead)
function drawHealthBadge(ctx, x, y, width, height, currentHp, maxHp) {
  if (currentHp < 0) currentHp = 0;
  const pct = currentHp / maxHp;

  ctx.save();
  ctx.fillStyle = 'rgba(10, 0, 20, 0.85)';
  ctx.fillRect(x - 2, y - 2, width + 4, height + 4);
  ctx.strokeStyle = '#b5179e';
  ctx.lineWidth = 1.5;
  ctx.strokeRect(x - 2, y - 2, width + 4, height + 4);

  let barColor = pct > 0.5 ? '#7209b7' : (pct > 0.25 ? '#b5179e' : '#f72585');
  ctx.fillStyle = barColor;
  ctx.fillRect(x, y, width * pct, height);

  ctx.fillStyle = '#4cc9f0';
  ctx.fillRect(x, y, width * pct, 2);
  ctx.restore();
}

// Update Fisika & Logika Game
function update() {
  if (player.state === 'dead') return;

  // Status Dash
  let currentSpeed = player.speed;
  if (player.state === 'dash') {
    currentSpeed = 11;
    player.dashTimer--;
    if (player.dashTimer <= 0) {
      player.state = 'idle';
      player.invulnerable = false;
    }
  }

  // Status Serangan
  if (player.state === 'attack') {
    player.attackTimer--;
    if (player.attackTimer <= 0) {
      player.state = 'idle';
    }
  }

  // Pergerakan Karakter
  if (keys.left) {
    player.vx = -currentSpeed;
    player.facing = 'left';
    if (player.state !== 'dash' && player.state !== 'attack' && player.isGrounded) player.state = 'run';
  } else if (keys.right) {
    player.vx = currentSpeed;
    player.facing = 'right';
    if (player.state !== 'dash' && player.state !== 'attack' && player.isGrounded) player.state = 'run';
  } else {
    player.vx = 0;
    if (player.state !== 'dash' && player.state !== 'attack' && player.isGrounded) player.state = 'idle';
  }

  player.vy += player.gravity;
  player.x += player.vx;
  player.y += player.vy;

  // Cek Pijakan Tanah
  player.isGrounded = false;
  platforms.forEach(plat => {
    if (player.x < plat.x + plat.width &&
        player.x + player.width > plat.x &&
        player.y + player.height >= plat.y &&
        player.y + player.height <= plat.y + plat.height &&
        player.vy >= 0) {
      player.isGrounded = true;
      player.vy = 0;
      player.y = plat.y - player.height;
    }
  });

  if (!player.isGrounded && player.state !== 'dash' && player.state !== 'attack') {
    player.state = 'jump';
  }

  // Batas Layar
  if (player.x < 0) player.x = 0;
  if (player.x + player.width > canvas.width) player.x = canvas.width - player.width;

  // Logika Musuh
  enemies.forEach(enemy => {
    if (enemy.isDead) return;
    enemy.x += enemy.vx * enemy.dir;
    if (enemy.x <= enemy.minX || enemy.x + enemy.width >= enemy.maxX) {
      enemy.dir *= -1;
    }

    // Tabrakan Musuh dengan Karakter
    if (checkCollision(player, enemy) && !player.invulnerable) {
      player.hp -= 0.6;
    }
  });

  // Logika Tembakan Menara
  towers.forEach(tower => {
    if (tower.hp <= 0) return;
    tower.shootTimer++;
    if (tower.shootTimer > 110) {
      projectiles.push({
        x: tower.x,
        y: tower.y + 35,
        vx: -4.5,
        vy: 0,
        radius: 9
      });
      tower.shootTimer = 0;
    }
  });

  // Update Proyektil Tembakan
  for (let i = projectiles.length - 1; i >= 0; i--) {
    let p = projectiles[i];
    p.x += p.vx;

    const pBox = { x: p.x - p.radius, y: p.y - p.radius, width: p.radius*2, height: p.radius*2 };
    if (checkCollision(player, pBox) && !player.invulnerable) {
      player.hp -= 18;
      projectiles.splice(i, 1);
      continue;
    }

    if (p.x < 0) projectiles.splice(i, 1);
  }

  // Cek Kematian Karakter
  if (player.hp <= 0) {
    player.hp = 0;
    player.state = 'dead';
  }
}

// Render Tampilan Game
function draw() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  // Latar Belakang
  ctx.fillStyle = '#0f051a';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  // Render Pijakan
  platforms.forEach(plat => {
    if (assets.platforms.complete && assets.platforms.naturalWidth !== 0) {
      ctx.drawImage(assets.platforms, 0, 0, 300, 150, plat.x, plat.y, plat.width, plat.height);
    } else {
      ctx.fillStyle = '#3a0ca3';
      ctx.fillRect(plat.x, plat.y, plat.width, plat.height);
    }
  });

  // Render Menara Tower
  towers.forEach(tower => {
    if (tower.hp > 0) {
      if (assets.towers.complete && assets.towers.naturalWidth !== 0) {
        ctx.drawImage(assets.towers, 0, 0, 300, 300, tower.x, tower.y, tower.width, tower.height);
      } else {
        ctx.fillStyle = '#480ca8';
        ctx.fillRect(tower.x, tower.y, tower.width, tower.height);
      }
      drawHealthBadge(ctx, tower.x, tower.y - 14, tower.width, 7, tower.hp, tower.maxHp);
    }
  });

  // Render Musuh (Reaper)
  enemies.forEach(enemy => {
    if (!enemy.isDead) {
      monsterAnimator.drawFrame(
        ctx,
        REAPER_ANIMS[enemy.state] || REAPER_ANIMS.walk,
        enemy.x,
        enemy.y,
        enemy.width,
        enemy.height,
        enemy.dir === -1
      );
      drawHealthBadge(ctx, enemy.x, enemy.y - 12, enemy.width, 6, enemy.hp, enemy.maxHp);
    }
  });

  // Render Proyektil Tembakan
  projectiles.forEach(p => {
    ctx.fillStyle = '#f72585';
    ctx.beginPath();
    ctx.arc(p.x, p.y, p.radius, 0, Math.PI * 2);
    ctx.fill();
    ctx.shadowBlur = 12;
    ctx.shadowColor = '#f72585';
  });
  ctx.shadowBlur = 0;

  // Render Karakter Utama
  const currentAnim = HERO_ANIMS[player.state] || HERO_ANIMS.idle;
  heroAnimator.drawFrame(
    ctx,
    currentAnim,
    player.x,
    player.y,
    player.width,
    player.height,
    player.facing === 'left'
  );

  // Badge Darah Di Atas Kepala Karakter Utama
  if (player.state !== 'dead') {
    drawHealthBadge(ctx, player.x - 5, player.y - 16, player.width + 10, 8, player.hp, player.maxHp);
  } else {
    ctx.fillStyle = '#f72585';
    ctx.font = '900 40px sans-serif';
    ctx.textAlign = 'center';
    ctx.fillText('GAME OVER', canvas.width / 2, canvas.height / 2 - 10);
    ctx.font = '16px sans-serif';
    ctx.fillStyle = '#fff';
    ctx.fillText('Muat ulang / Refresh halaman untuk restart', canvas.width / 2, canvas.height / 2 + 30);
  }
}

// Loop Utama Game
function gameLoop() {
  update();
  draw();
  requestAnimationFrame(gameLoop);
}

gameLoop();
</script>
</body>
</html>
