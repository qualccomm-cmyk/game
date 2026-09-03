<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
  <title>2D Shadow Purple Platformer</title>
  <style>
    * {
      box-sizing: border-box;
      margin: 0;
      padding: 0;
      user-select: none;
      -webkit-user-select: none;
    }
    body {
      background-color: #0b0512;
      color: #fff;
      font-family: Arial, sans-serif;
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
      max-width: 900px;
      height: 506px; /* Aspect Ratio 16:9 */
      background: #180a29;
      border: 3px solid #7b2cbf;
      box-shadow: 0 0 20px rgba(123, 44, 191, 0.6);
      overflow: hidden;
    }
    canvas {
      display: block;
      width: 100%;
      height: 100%;
    }
    /* Mobile Controls Overlay */
    #touch-controls {
      position: absolute;
      bottom: 10px;
      left: 0;
      width: 100%;
      display: flex;
      justify-content: space-between;
      padding: 0 15px;
      pointer-events: none;
    }
    .btn-group {
      display: flex;
      gap: 10px;
      pointer-events: auto;
    }
    .btn {
      width: 55px;
      height: 55px;
      background: rgba(123, 44, 191, 0.4);
      border: 2px solid #b5179e;
      border-radius: 50%;
      color: white;
      font-weight: bold;
      font-size: 16px;
      display: flex;
      align-items: center;
      justify-content: center;
      touch-action: manipulation;
    }
    .btn:active {
      background: rgba(181, 23, 158, 0.8);
      transform: scale(0.95);
    }
    .btn-action {
      background: rgba(247, 37, 133, 0.4);
      border-color: #f72585;
    }
  </style>
</head>
<body>

<div id="game-container">
  <canvas id="gameCanvas" width="900" height="506"></canvas>
  
  <div id="touch-controls">
    <div class="btn-group">
      <div class="btn" id="btn-left">◄</div>
      <div class="btn" id="btn-right">►</div>
    </div>
    <div class="btn-group">
      <div class="btn btn-action" id="btn-dash">DASH</div>
      <div class="btn btn-action" id="btn-attack">ATK</div>
      <div class="btn btn-action" id="btn-jump">JUMP</div>
    </div>
  </div>
</div>

<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

// Load Assets
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

// Inputs Status
const keys = { left: false, right: false, jump: false, attack: false, dash: false };

// Player Configuration
const player = {
  x: 100,
  y: 300,
  width: 50,
  height: 65,
  vx: 0,
  vy: 0,
  speed: 4,
  jumpPower: -11,
  gravity: 0.5,
  isGrounded: false,
  facing: 'right',
  hp: 100,
  maxHp: 100,
  isAttacking: false,
  attackTimer: 0,
  isDashing: false,
  dashTimer: 0,
  isDead: false,
  deathTimer: 0,
  animFrame: 0,
  animTimer: 0
};

// Platforms Data
const platforms = [
  { x: 0, y: 440, width: 900, height: 66 }, // Tanah Utama
  { x: 180, y: 320, width: 160, height: 25 },
  { x: 420, y: 240, width: 180, height: 25 },
  { x: 680, y: 340, width: 160, height: 25 }
];

// Towers Data
const towers = [
  { x: 720, y: 220, width: 70, height: 120, hp: 120, maxHp: 120, shootTimer: 0 }
];

// Enemies Data
const enemies = [
  { x: 230, y: 270, width: 45, height: 50, hp: 50, maxHp: 50, vx: 1, minX: 180, maxX: 300, dir: 1, isDead: false },
  { x: 500, y: 390, width: 45, height: 50, hp: 60, maxHp: 60, vx: 1.5, minX: 450, maxX: 650, dir: 1, isDead: false }
];

// Projectiles Data
const projectiles = [];

// Event Listeners (Keyboard)
window.addEventListener('keydown', (e) => {
  if (player.isDead) return;
  if (e.key === 'ArrowLeft' || e.key === 'a') keys.left = true;
  if (e.key === 'ArrowRight' || e.key === 'd') keys.right = true;
  if ((e.key === 'ArrowUp' || e.key === 'w' || e.key === ' ') && player.isGrounded) keys.jump = true;
  if (e.key === 'j' || e.key === 'z') triggerAttack();
  if (e.key === 'k' || e.key === 'x') triggerDash();
});

window.addEventListener('keyup', (e) => {
  if (e.key === 'ArrowLeft' || e.key === 'a') keys.left = false;
  if (e.key === 'ArrowRight' || e.key === 'd') keys.right = false;
});

// Mobile Controls Event Bindings
function bindTouch(id, actionStart, actionEnd) {
  const btn = document.getElementById(id);
  btn.addEventListener('touchstart', (e) => { e.preventDefault(); actionStart(); });
  btn.addEventListener('touchend', (e) => { e.preventDefault(); if (actionEnd) actionEnd(); });
  btn.addEventListener('mousedown', (e) => { actionStart(); });
  btn.addEventListener('mouseup', (e) => { if (actionEnd) actionEnd(); });
}

bindTouch('btn-left', () => keys.left = true, () => keys.left = false);
bindTouch('btn-right', () => keys.right = true, () => keys.right = false);
bindTouch('btn-jump', () => { if (player.isGrounded && !player.isDead) player.vy = player.jumpPower; });
bindTouch('btn-attack', () => triggerAttack());
bindTouch('btn-dash', () => triggerDash());

function triggerAttack() {
  if (!player.isAttacking && !player.isDead) {
    player.isAttacking = true;
    player.attackTimer = 15;
    
    // Hit Detection
    const atkBox = {
      x: player.facing === 'right' ? player.x + player.width : player.x - 60,
      y: player.y,
      width: 60,
      height: player.height
    };

    // Damage Enemies
    enemies.forEach(enemy => {
      if (!enemy.isDead && checkCollision(atkBox, enemy)) {
        enemy.hp -= 25;
        if (enemy.hp <= 0) enemy.isDead = true;
      }
    });

    // Damage Towers
    towers.forEach(tower => {
      if (tower.hp > 0 && checkCollision(atkBox, tower)) {
        tower.hp -= 20;
      }
    });
  }
}

function triggerDash() {
  if (!player.isDashing && !player.isDead) {
    player.isDashing = true;
    player.dashTimer = 12;
  }
}

function checkCollision(r1, r2) {
  return !(r1.x > r2.x + r2.width ||
           r1.x + r1.width < r2.x ||
           r1.y > r2.y + r2.height ||
           r1.y + r1.height < r2.y);
}

// Game Update Loop
function update() {
  if (player.isDead) {
    player.deathTimer++;
    return;
  }

  // Dash Mechanics
  let currentSpeed = player.speed;
  if (player.isDashing) {
    currentSpeed = 12;
    player.dashTimer--;
    if (player.dashTimer <= 0) player.isDashing = false;
  }

  // Movement Physics
  if (keys.left) {
    player.vx = -currentSpeed;
    player.facing = 'left';
  } else if (keys.right) {
    player.vx = currentSpeed;
    player.facing = 'right';
  } else {
    player.vx = 0;
  }

  if (keys.jump && player.isGrounded) {
    player.vy = player.jumpPower;
    player.isGrounded = false;
    keys.jump = false;
  }

  player.vy += player.gravity;
  player.x += player.vx;
  player.y += player.vy;

  // Platform Collision
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

  // Keep inside Canvas Bounds
  if (player.x < 0) player.x = 0;
  if (player.x + player.width > canvas.width) player.x = canvas.width - player.width;

  // Attack Timer
  if (player.isAttacking) {
    player.attackTimer--;
    if (player.attackTimer <= 0) player.isAttacking = false;
  }

  // Enemy Mechanics
  enemies.forEach(enemy => {
    if (enemy.isDead) return;
    enemy.x += enemy.vx * enemy.dir;
    if (enemy.x <= enemy.minX || enemy.x + enemy.width >= enemy.maxX) {
      enemy.dir *= -1;
    }

    // Touch Enemy Damage
    if (checkCollision(player, enemy) && !player.isDashing) {
      player.hp -= 0.5;
    }
  });

  // Tower Shooting Mechanics
  towers.forEach(tower => {
    if (tower.hp <= 0) return;
    tower.shootTimer++;
    if (tower.shootTimer > 120) { // Shot every 2 seconds
      projectiles.push({
        x: tower.x,
        y: tower.y + 40,
        vx: -4,
        vy: 0,
        radius: 8
      });
      tower.shootTimer = 0;
    }
  });

  // Projectiles Update
  for (let i = projectiles.length - 1; i >= 0; i--) {
    let p = projectiles[i];
    p.x += p.vx;

    // Hit Player
    const pBox = { x: p.x - p.radius, y: p.y - p.radius, width: p.radius*2, height: p.radius*2 };
    if (checkCollision(player, pBox)) {
      player.hp -= 15;
      projectiles.splice(i, 1);
      continue;
    }

    // Remove Out of Bounds
    if (p.x < 0) projectiles.splice(i, 1);
  }

  // Check Death
  if (player.hp <= 0) {
    player.hp = 0;
    player.isDead = true;
  }
}

// Render Health Bar Helper
function drawHealthBar(x, y, width, height, currentHp, maxHp) {
  if (currentHp < 0) currentHp = 0;
  ctx.fillStyle = 'rgba(0, 0, 0, 0.6)';
  ctx.fillRect(x, y, width, height);
  ctx.fillStyle = currentHp > (maxHp * 0.3) ? '#7209b7' : '#f72585';
  ctx.fillRect(x, y, (currentHp / maxHp) * width, height);
  ctx.strokeStyle = '#fff';
  ctx.lineWidth = 1;
  ctx.strokeRect(x, y, width, height);
}

// Draw Function
function draw() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  // Background Grid/Vibe
  ctx.fillStyle = '#120720';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  // Platforms Rendering
  platforms.forEach(plat => {
    if (assets.platforms.complete && assets.platforms.naturalWidth !== 0) {
      ctx.drawImage(assets.platforms, 10, 10, 300, 150, plat.x, plat.y, plat.width, plat.height);
    } else {
      ctx.fillStyle = '#3a0ca3';
      ctx.fillRect(plat.x, plat.y, plat.width, plat.height);
    }
  });

  // Towers Rendering
  towers.forEach(tower => {
    if (tower.hp > 0) {
      if (assets.towers.complete && assets.towers.naturalWidth !== 0) {
        ctx.drawImage(assets.towers, 0, 0, 250, 300, tower.x, tower.y, tower.width, tower.height);
      } else {
        ctx.fillStyle = '#480ca8';
        ctx.fillRect(tower.x, tower.y, tower.width, tower.height);
      }
      drawHealthBar(tower.x, tower.y - 12, tower.width, 6, tower.hp, tower.maxHp);
    }
  });

  // Enemies Rendering
  enemies.forEach(enemy => {
    if (!enemy.isDead) {
      if (assets.monsters.complete && assets.monsters.naturalWidth !== 0) {
        ctx.drawImage(assets.monsters, 0, 0, 120, 120, enemy.x, enemy.y, enemy.width, enemy.height);
      } else {
        ctx.fillStyle = '#b5179e';
        ctx.fillRect(enemy.x, enemy.y, enemy.width, enemy.height);
      }
      drawHealthBar(enemy.x, enemy.y - 10, enemy.width, 5, enemy.hp, enemy.maxHp);
    }
  });

  // Projectiles Rendering
  projectiles.forEach(p => {
    ctx.fillStyle = '#f72585';
    ctx.beginPath();
    ctx.arc(p.x, p.y, p.radius, 0, Math.PI * 2);
    ctx.fill();
    ctx.shadowBlur = 10;
    ctx.shadowColor = '#f72585';
  });
  ctx.shadowBlur = 0;

  // Player Rendering
  if (!player.isDead) {
    if (assets.heroine.complete && assets.heroine.naturalWidth !== 0) {
      // Dynamic Sprite Crop Logic
      let sx = 0, sy = 0;
      if (player.isAttacking) { sx = 200; sy = 200; }
      else if (player.isDashing) { sx = 400; sy = 100; }
      
      ctx.save();
      if (player.facing === 'left') {
        ctx.translate(player.x + player.width, player.y);
        ctx.scale(-1, 1);
        ctx.drawImage(assets.heroine, sx, sy, 120, 120, 0, 0, player.width, player.height);
      } else {
        ctx.drawImage(assets.heroine, sx, sy, 120, 120, player.x, player.y, player.width, player.height);
      }
      ctx.restore();
    } else {
      ctx.fillStyle = player.isDashing ? '#4cc9f0' : '#4895ef';
      ctx.fillRect(player.x, player.y, player.width, player.height);
    }

    // Visual Effect for Attack
    if (player.isAttacking) {
      ctx.fillStyle = 'rgba(247, 37, 133, 0.5)';
      let atkX = player.facing === 'right' ? player.x + player.width : player.x - 40;
      ctx.fillRect(atkX, player.y, 40, player.height);
    }

    // Health Bar di Atas Kepala Karakter
    drawHealthBar(player.x - 5, player.y - 15, player.width + 10, 7, player.hp, player.maxHp);
  } else {
    // Death Animation & Text
    ctx.fillStyle = '#f72585';
    ctx.font = 'bold 36px Arial';
    ctx.textAlign = 'center';
    ctx.fillText('GAME OVER', canvas.width / 2, canvas.height / 2);
    ctx.font = '16px Arial';
    ctx.fillText('Refresh/Muat Ulang Halaman untuk Main Lagi', canvas.width / 2, canvas.height / 2 + 40);
  }
}

// Loop Utama Game
function gameLoop() {
  update();
  draw();
  requestAnimationFrame(gameLoop);
}

// Jalankan Game saat Siap
gameLoop();
</script>
</body>
</html>
