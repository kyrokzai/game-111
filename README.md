<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
<title>Батыр и Баурсаки</title>
<style>
  html, body {
    margin: 0;
    padding: 0;
    width: 100%;
    height: 100%;
    overflow: hidden;
    background: #111827;
    font-family: Arial, sans-serif;
    touch-action: none;
  }

  canvas {
    display: block;
    margin: 0 auto;
    width: 100vw;
    height: 100vh;
    max-width: 520px;
    background: #87ceeb;
  }
</style>
</head>
<body>
<canvas id="game"></canvas>

<script>
(() => {
  const canvas = document.getElementById("game");
  const ctx = canvas.getContext("2d");

  const W = 420;
  const H = 740;
  canvas.width = W;
  canvas.height = H;

  let state = "menu";
  let lastTime = 0;
  let score = 0;
  let best = Number(localStorage.getItem("batyr_best") || 0);
  let muted = false;
  let shake = 0;
  let flash = 0;

  let audioCtx = null;

  const groundY = 610;

  const player = {
    x: 92,
    y: groundY,
    w: 64,
    h: 86,
    vy: 0,
    onGround: true,
    shield: 0,
    magnet: 0,
    blink: 0
  };

  let obstacles = [];
  let coins = [];
  let bonuses = [];
  let particles = [];
  let clouds = [];

  let obstacleTimer = 0;
  let coinTimer = 0;
  let bonusTimer = 7;
  let distance = 0;
  let speed = 250;
  let bgOffset = 0;

  for (let i = 0; i < 7; i++) {
    clouds.push({
      x: Math.random() * W,
      y: 70 + Math.random() * 170,
      s: 0.6 + Math.random() * 0.9,
      v: 10 + Math.random() * 15
    });
  }

  function resetGame() {
    score = 0;
    distance = 0;
    speed = 250;
    shake = 0;
    flash = 0;
    player.y = groundY;
    player.vy = 0;
    player.onGround = true;
    player.shield = 0;
    player.magnet = 0;
    player.blink = 0;
    obstacles = [];
    coins = [];
    bonuses = [];
    particles = [];
    obstacleTimer = 0.6;
    coinTimer = 0.4;
    bonusTimer = 5 + Math.random() * 4;
  }

  function initAudio() {
    if (!audioCtx) {
      audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    }
    if (audioCtx.state === "suspended") audioCtx.resume();
  }

  function sound(type) {
    if (muted || !audioCtx) return;

    const osc = audioCtx.createOscillator();
    const gain = audioCtx.createGain();
    osc.connect(gain);
    gain.connect(audioCtx.destination);

    const t = audioCtx.currentTime;

    if (type === "jump") {
      osc.type = "triangle";
      osc.frequency.setValueAtTime(280, t);
      osc.frequency.exponentialRampToValueAtTime(520, t + 0.12);
      gain.gain.setValueAtTime(0.12, t);
      gain.gain.exponentialRampToValueAtTime(0.001, t + 0.16);
      osc.start(t);
      osc.stop(t + 0.17);
    }

    if (type === "coin") {
      osc.type = "sine";
      osc.frequency.setValueAtTime(650, t);
      osc.frequency.exponentialRampToValueAtTime(1100, t + 0.08);
      gain.gain.setValueAtTime(0.09, t);
      gain.gain.exponentialRampToValueAtTime(0.001, t + 0.13);
      osc.start(t);
      osc.stop(t + 0.14);
    }

    if (type === "bonus") {
      osc.type = "square";
      osc.frequency.setValueAtTime(420, t);
      osc.frequency.setValueAtTime(640, t + 0.08);
      osc.frequency.setValueAtTime(880, t + 0.16);
      gain.gain.setValueAtTime(0.08, t);
      gain.gain.exponentialRampToValueAtTime(0.001, t + 0.35);
      osc.start(t);
      osc.stop(t + 0.36);
    }

    if (type === "hit") {
      osc.type = "sawtooth";
      osc.frequency.setValueAtTime(140, t);
      osc.frequency.exponentialRampToValueAtTime(45, t + 0.2);
      gain.gain.setValueAtTime(0.16, t);
      gain.gain.exponentialRampToValueAtTime(0.001, t + 0.25);
      osc.start(t);
      osc.stop(t + 0.26);
    }
  }

  function jump() {
    if (state !== "play") return;
    if (player.onGround) {
      player.vy = -720;
      player.onGround = false;
      createDust(player.x + 15, groundY - 6, 12);
      sound("jump");
    }
  }

  function startGame() {
    initAudio();
    resetGame();
    state = "play";
  }

  function gameOver() {
    state = "over";
    shake = 14;
    flash = 0.5;
    sound("hit");
    if (score > best) {
      best = score;
      localStorage.setItem("batyr_best", String(best));
    }
  }

  function createDust(x, y, count) {
    for (let i = 0; i < count; i++) {
      particles.push({
        x,
        y,
        vx: -60 - Math.random() * 150,
        vy: -40 - Math.random() * 90,
        r: 2 + Math.random() * 4,
        life: 0.35 + Math.random() * 0.25,
        color: "rgba(220,180,100,"
      });
    }
  }

  function createSpark(x, y, count, color = "rgba(255,220,60,") {
    for (let i = 0; i < count; i++) {
      const a = Math.random() * Math.PI * 2;
      const sp = 80 + Math.random() * 220;
      particles.push({
        x,
        y,
        vx: Math.cos(a) * sp,
        vy: Math.sin(a) * sp,
        r: 2 + Math.random() * 4,
        life: 0.45 + Math.random() * 0.3,
        color
      });
    }
  }

  function rectsOverlap(a, b) {
    return a.x < b.x + b.w &&
           a.x + a.w > b.x &&
           a.y < b.y + b.h &&
           a.y + a.h > b.y;
  }

  function circleRect(cx, cy, cr, r) {
    const nx = Math.max(r.x, Math.min(cx, r.x + r.w));
    const ny = Math.max(r.y, Math.min(cy, r.y + r.h));
    const dx = cx - nx;
    const dy = cy - ny;
    return dx * dx + dy * dy < cr * cr;
  }

  function playerRect() {
    return {
      x: player.x + 8,
      y: player.y - player.h + 12,
      w: player.w - 14,
      h: player.h - 18
    };
  }

  function spawnObstacle() {
    const type = Math.random() < 0.75 ? "wolf" : "rock";
    obstacles.push({
      type,
      x: W + 40,
      y: groundY,
      w: type === "wolf" ? 68 : 48,
      h: type === "wolf" ? 44 : 50,
      phase: Math.random() * Math.PI * 2
    });
  }

  function spawnCoinLine() {
    const count = 3 + Math.floor(Math.random() * 4);
    const baseY = 455 + Math.random() * 80;
    for (let i = 0; i < count; i++) {
      coins.push({
        x: W + 40 + i * 44,
        y: baseY - Math.sin(i * 0.9) * 25,
        r: 14,
        rot: Math.random() * 6
      });
    }
  }

  function spawnBonus() {
    bonuses.push({
      x: W + 50,
      y: 390 + Math.random() * 100,
      r: 18,
      rot: 0
    });
  }

  function update(dt) {
    if (state !== "play") return;

    dt = Math.min(dt, 0.033);

    distance += speed * dt;
    score = Math.floor(distance / 9);
    speed = 250 + Math.min(220, distance / 18);

    bgOffset += speed * dt;

    obstacleTimer -= dt;
    coinTimer -= dt;
    bonusTimer -= dt;

    if (obstacleTimer <= 0) {
      spawnObstacle();
      obstacleTimer = Math.max(0.65, 1.35 - speed / 650 + Math.random() * 0.55);
    }

    if (coinTimer <= 0) {
      spawnCoinLine();
      coinTimer = 1.0 + Math.random() * 1.4;
    }

    if (bonusTimer <= 0) {
      spawnBonus();
      bonusTimer = 8 + Math.random() * 7;
    }

    player.vy += 1800 * dt;
    player.y += player.vy * dt;

    if (player.y >= groundY) {
      if (!player.onGround && player.vy > 300) {
        createDust(player.x + 20, groundY - 5, 10);
      }
      player.y = groundY;
      player.vy = 0;
      player.onGround = true;
    }

    if (player.shield > 0) player.shield -= dt;
    if (player.magnet > 0) player.magnet -= dt;
    if (player.blink > 0) player.blink -= dt;
    if (shake > 0) shake -= dt * 25;
    if (flash > 0) flash -= dt;

    for (const o of obstacles) {
      o.x -= speed * dt;
      o.phase += dt * 8;
    }

    for (const c of coins) {
      c.x -= speed * dt;
      c.rot += dt * 7;

      if (player.magnet > 0) {
        const pr = playerRect();
        const tx = pr.x + pr.w / 2;
        const ty = pr.y + pr.h / 2;
        const dx = tx - c.x;
        const dy = ty - c.y;
        const d = Math.sqrt(dx * dx + dy * dy);
        if (d < 170) {
          c.x += dx * dt * 5;
          c.y += dy * dt * 5;
        }
      }
    }

    for (const b of bonuses) {
      b.x -= speed * dt;
      b.rot += dt * 5;
    }

    for (const p of particles) {
      p.life -= dt;
      p.x += p.vx * dt;
      p.y += p.vy * dt;
      p.vy += 280 * dt;
    }

    obstacles = obstacles.filter(o => o.x + o.w > -60);
    coins = coins.filter(c => c.x > -50 && !c.dead);
    bonuses = bonuses.filter(b => b.x > -50 && !b.dead);
    particles = particles.filter(p => p.life > 0);

    for (const cl of clouds) {
      cl.x -= cl.v * dt;
      if (cl.x < -120) {
        cl.x = W + 120;
        cl.y = 60 + Math.random() * 170;
        cl.s = 0.6 + Math.random() * 0.9;
      }
    }

    const pr = playerRect();

    for (const c of coins) {
      if (!c.dead && circleRect(c.x, c.y, c.r, pr)) {
        c.dead = true;
        score += 10;
        distance += 90;
        createSpark(c.x, c.y, 9, "rgba(255,210,55,");
        sound("coin");
      }
    }

    for (const b of bonuses) {
      if (!b.dead && circleRect(b.x, b.y, b.r, pr)) {
        b.dead = true;
        player.shield = 6;
        player.magnet = 6;
        score += 40;
        distance += 360;
        shake = 5;
        createSpark(b.x, b.y, 24, "rgba(80,240,255,");
        sound("bonus");
      }
    }

    for (const o of obstacles) {
      const or = {
        x: o.x + 8,
        y: o.y - o.h + 8,
        w: o.w - 14,
        h: o.h - 10
      };

      if (rectsOverlap(pr, or)) {
        if (player.shield > 0 && player.blink <= 0) {
          player.shield = Math.max(0, player.shield - 2);
          player.blink = 0.8;
          o.x -= 120;
          shake = 9;
          createSpark(pr.x + pr.w / 2, pr.y + pr.h / 2, 22, "rgba(80,240,255,");
          sound("bonus");
        } else if (player.blink <= 0) {
          gameOver();
        }
      }
    }
  }

  function roundedRect(x, y, w, h, r) {
    ctx.beginPath();
    ctx.moveTo(x + r, y);
    ctx.lineTo(x + w - r, y);
    ctx.quadraticCurveTo(x + w, y, x + w, y + r);
    ctx.lineTo(x + w, y + h - r);
    ctx.quadraticCurveTo(x + w, y + h, x + w - r, y + h);
    ctx.lineTo(x + r, y + h);
    ctx.quadraticCurveTo(x, y + h, x, y + h - r);
    ctx.lineTo(x, y + r);
    ctx.quadraticCurveTo(x, y, x + r, y);
    ctx.closePath();
  }

  function drawBackground() {
    const sky = ctx.createLinearGradient(0, 0, 0, H);
    sky.addColorStop(0, "#60c9ff");
    sky.addColorStop(0.55, "#b8f0ff");
    sky.addColorStop(1, "#ffe9a3");
    ctx.fillStyle = sky;
    ctx.fillRect(0, 0, W, H);

    ctx.fillStyle = "#ffd14f";
    ctx.beginPath();
    ctx.arc(335, 105, 42, 0, Math.PI * 2);
    ctx.fill();

    ctx.fillStyle = "rgba(255,255,255,0.82)";
    for (const cl of clouds) {
      ctx.save();
      ctx.translate(cl.x, cl.y);
      ctx.scale(cl.s, cl.s);
      ctx.beginPath();
      ctx.arc(0, 15, 22, 0, Math.PI * 2);
      ctx.arc(25, 5, 28, 0, Math.PI * 2);
      ctx.arc(55, 15, 21, 0, Math.PI * 2);
      ctx.arc(25, 22, 30, 0, Math.PI * 2);
      ctx.fill();
      ctx.restore();
    }

    ctx.fillStyle = "#78b7d6";
    ctx.beginPath();
    ctx.moveTo(0, 365);
    ctx.lineTo(80, 245);
    ctx.lineTo(155, 365);
    ctx.lineTo(230, 260);
    ctx.lineTo(320, 365);
    ctx.lineTo(420, 235);
    ctx.lineTo(420, 420);
    ctx.lineTo(0, 420);
    ctx.closePath();
    ctx.fill();

    ctx.fillStyle = "#6aa4c5";
    ctx.beginPath();
    ctx.moveTo(0, 405);
    ctx.lineTo(70, 310);
    ctx.lineTo(155, 405);
    ctx.lineTo(250, 300);
    ctx.lineTo(350, 405);
    ctx.lineTo(420, 320);
    ctx.lineTo(420, 460);
    ctx.lineTo(0, 460);
    ctx.closePath();
    ctx.fill();

    ctx.fillStyle = "#e9b85d";
    ctx.fillRect(0, 430, W, H - 430);

    ctx.fillStyle = "#5eb84f";
    ctx.fillRect(0, groundY, W, H - groundY);

    ctx.fillStyle = "#48a840";
    for (let x = -40 - (bgOffset * 0.45) % 80; x < W + 80; x += 80) {
      ctx.beginPath();
      ctx.ellipse(x, groundY + 35, 60, 18, 0, 0, Math.PI * 2);
      ctx.fill();
    }

    ctx.strokeStyle = "rgba(130,90,30,0.35)";
    ctx.lineWidth = 3;
    for (let x = -60 - (bgOffset * 0.8) % 90; x < W + 90; x += 90) {
      ctx.beginPath();
      ctx.moveTo(x, groundY + 12);
      ctx.quadraticCurveTo(x + 25, groundY + 30, x + 55, groundY + 15);
      ctx.stroke();
    }

    ctx.fillStyle = "#ff6fb1";
    for (let x = -20 - (bgOffset * 1.2) % 70; x < W + 70; x += 70) {
      ctx.beginPath();
      ctx.arc(x, groundY + 55, 4, 0, Math.PI * 2);
      ctx.fill();
    }
  }

  function drawPlayer() {
    const x = player.x;
    const y = player.y;
    const bob = player.onGround ? Math.sin(performance.now() / 80) * 2 : 0;

    if (player.shield > 0) {
      ctx.save();
      ctx.globalAlpha = 0.35 + Math.sin(performance.now() / 90) * 0.12;
      ctx.strokeStyle = "#35e8ff";
      ctx.lineWidth = 5;
      ctx.beginPath();
      ctx.arc(x + 34, y - 45, 58, 0, Math.PI * 2);
      ctx.stroke();
      ctx.restore();
    }

    ctx.save();
    ctx.translate(x, y + bob);

    ctx.fillStyle = "#7a3f20";
    ctx.beginPath();
    ctx.ellipse(33, -35, 38, 24, 0, 0, Math.PI * 2);
    ctx.fill();

    ctx.fillStyle = "#8d4b27";
    ctx.beginPath();
    ctx.ellipse(66, -54, 19, 14, -0.3, 0, Math.PI * 2);
    ctx.fill();

    ctx.fillStyle = "#3b1d12";
    ctx.beginPath();
    ctx.moveTo(52, -65);
    ctx.lineTo(62, -85);
    ctx.lineTo(67, -62);
    ctx.fill();

    ctx.fillStyle = "#111";
    ctx.beginPath();
    ctx.arc(73, -58, 2.5, 0, Math.PI * 2);
    ctx.fill();

    ctx.strokeStyle = "#4a2614";
    ctx.lineWidth = 6;
    const leg = Math.sin(performance.now() / 70) * 9;
    ctx.beginPath();
    ctx.moveTo(10, -18);
    ctx.lineTo(5 + leg, 0);
    ctx.moveTo(30, -17);
    ctx.lineTo(27 - leg, 0);
    ctx.moveTo(48, -18);
    ctx.lineTo(48 + leg, 0);
    ctx.moveTo(62, -20);
    ctx.lineTo(65 - leg, 0);
    ctx.stroke();

    ctx.strokeStyle = "#2b150d";
    ctx.lineWidth = 5;
    ctx.beginPath();
    ctx.moveTo(-2, -43);
    ctx.quadraticCurveTo(-25, -52, -18, -72);
    ctx.stroke();

    ctx.fillStyle = "#2563eb";
    ctx.fillRect(24, -70, 20, 31);

    ctx.fillStyle = "#f3b35b";
    ctx.beginPath();
    ctx.arc(34, -82, 12, 0, Math.PI * 2);
    ctx.fill();

    ctx.fillStyle = "#d71920";
    ctx.beginPath();
    ctx.moveTo(18, -91);
    ctx.lineTo(50, -91);
    ctx.lineTo(42, -105);
    ctx.lineTo(25, -105);
    ctx.closePath();
    ctx.fill();

    ctx.strokeStyle = "#f7d46b";
    ctx.lineWidth = 4;
    ctx.beginPath();
    ctx.moveTo(43, -64);
    ctx.lineTo(65, -42);
    ctx.stroke();

    ctx.restore();
  }

  function drawWolf(o) {
    const x = o.x;
    const y = o.y;
    const run = Math.sin(o.phase) * 4;

    ctx.save();
    ctx.translate(x, y);

    ctx.fillStyle = "#515866";
    ctx.beginPath();
    ctx.ellipse(30, -28, 32, 17, 0, 0, Math.PI * 2);
    ctx.fill();

    ctx.fillStyle = "#626b7a";
    ctx.beginPath();
    ctx.ellipse(60, -38, 17, 13, -0.2, 0, Math.PI * 2);
    ctx.fill();

    ctx.fillStyle = "#515866";
    ctx.beginPath();
    ctx.moveTo(53, -50);
    ctx.lineTo(57, -67);
    ctx.lineTo(65, -50);
    ctx.closePath();
    ctx.fill();

    ctx.beginPath();
    ctx.moveTo(66, -49);
    ctx.lineTo(77, -62);
    ctx.lineTo(74, -43);
    ctx.closePath();
    ctx.fill();

    ctx.fillStyle = "#111827";
    ctx.beginPath();
    ctx.arc(67, -40, 2.5, 0, Math.PI * 2);
    ctx.fill();

    ctx.strokeStyle = "#3a404a";
    ctx.lineWidth = 5;
    ctx.beginPath();
    ctx.moveTo(12, -16);
    ctx.lineTo(8 + run, 0);
    ctx.moveTo(32, -15);
    ctx.lineTo(30 - run, 0);
    ctx.moveTo(48, -16);
    ctx.lineTo(50 + run, 0);
    ctx.stroke();

    ctx.strokeStyle = "#515866";
    ctx.lineWidth = 5;
    ctx.beginPath();
    ctx.moveTo(2, -31);
    ctx.quadraticCurveTo(-18, -45, -10, -55);
    ctx.stroke();

    ctx.restore();
  }

  function drawRock(o) {
    ctx.save();
    ctx.translate(o.x, o.y);
    ctx.fillStyle = "#7b6d62";
    ctx.beginPath();
    ctx.moveTo(5, 0);
    ctx.lineTo(10, -35);
    ctx.lineTo(28, -52);
    ctx.lineTo(50, -35);
    ctx.lineTo(45, 0);
    ctx.closePath();
    ctx.fill();

    ctx.fillStyle = "rgba(255,255,255,0.18)";
    ctx.beginPath();
    ctx.moveTo(17, -33);
    ctx.lineTo(29, -45);
    ctx.lineTo(40, -31);
    ctx.lineTo(29, -36);
    ctx.closePath();
    ctx.fill();
    ctx.restore();
  }

  function drawCoin(c) {
    ctx.save();
    ctx.translate(c.x, c.y);
    ctx.rotate(c.rot);
    ctx.fillStyle = "#ffcc33";
    ctx.beginPath();
    ctx.ellipse(0, 0, 14, 11, 0, 0, Math.PI * 2);
    ctx.fill();

    ctx.strokeStyle = "#fff1a8";
    ctx.lineWidth = 3;
    ctx.beginPath();
    ctx.ellipse(0, 0, 8, 6, 0, 0, Math.PI * 2);
    ctx.stroke();

    ctx.fillStyle = "#9a5b00";
    ctx.font = "bold 12px Arial";
    ctx.textAlign = "center";
    ctx.textBaseline = "middle";
    ctx.fillText("Б", 0, 1);
    ctx.restore();
  }

  function drawBonus(b) {
    ctx.save();
    ctx.translate(b.x, b.y);
    ctx.rotate(b.rot);

    ctx.fillStyle = "#2dd4ff";
    ctx.beginPath();
    for (let i = 0; i < 10; i++) {
      const r = i % 2 === 0 ? 22 : 10;
      const a = i * Math.PI / 5;
      ctx.lineTo(Math.cos(a) * r, Math.sin(a) * r);
    }
    ctx.closePath();
    ctx.fill();

    ctx.fillStyle = "#ffffff";
    ctx.beginPath();
    ctx.arc(0, 0, 8, 0, Math.PI * 2);
    ctx.fill();

    ctx.restore();
  }

  function drawParticles() {
    for (const p of particles) {
      const a = Math.max(0, p.life * 2);
      ctx.fillStyle = p.color + a + ")";
      ctx.beginPath();
      ctx.arc(p.x, p.y, p.r, 0, Math.PI * 2);
      ctx.fill();
    }
  }

  function drawUI() {
    ctx.fillStyle = "rgba(0,0,0,0.25)";
    roundedRect(14, 14, 150, 42, 14);
    ctx.fill();

    ctx.fillStyle = "#fff";
    ctx.font = "bold 24px Arial";
    ctx.textAlign = "left";
    ctx.textBaseline = "middle";
    ctx.fillText("Счёт: " + score, 25, 36);

    if (player.shield > 0) {
      ctx.fillStyle = "rgba(0,0,0,0.25)";
      roundedRect(14, 62, 155, 32, 12);
      ctx.fill();

      ctx.fillStyle = "#6ff5ff";
      ctx.font = "bold 16px Arial";
      ctx.fillText("Супер: " + Math.ceil(player.shield) + "с", 26, 78);
    }

    ctx.fillStyle = "rgba(0,0,0,0.28)";
    roundedRect(W - 60, 14, 46, 42, 14);
    ctx.fill();

    ctx.fillStyle = "#fff";
    ctx.font = "bold 22px Arial";
    ctx.textAlign = "center";
    ctx.fillText(muted ? "🔇" : "🔊", W - 37, 36);
  }

  function drawMenu() {
    drawBackground();

    ctx.fillStyle = "rgba(0,0,0,0.35)";
    ctx.fillRect(0, 0, W, H);

    ctx.textAlign = "center";
    ctx.textBaseline = "middle";

    ctx.fillStyle = "#fff7c2";
    ctx.font = "bold 42px Arial";
    ctx.fillText("Батыр", W / 2, 190);
    ctx.fillText("и Баурсаки", W / 2, 240);

    ctx.fillStyle = "#ffffff";
    ctx.font = "18px Arial";
    ctx.fillText("Прыгай через волков и собирай баурсаки", W / 2, 300);

    ctx.fillStyle = "#16a34a";
    roundedRect(90, 365, 240, 72, 22);
    ctx.fill();

    ctx.fillStyle = "#ffffff";
    ctx.font = "bold 30px Arial";
    ctx.fillText("Играть", W / 2, 402);

    ctx.fillStyle = "rgba(255,255,255,0.88)";
    ctx.font = "16px Arial";
    ctx.fillText("Рекорд: " + best, W / 2, 470);

    drawSoundButtonOnly();
  }

  function drawGameOver() {
    drawBackground();

    for (const c of coins) drawCoin(c);
    for (const b of bonuses) drawBonus(b);
    for (const o of obstacles) o.type === "wolf" ? drawWolf(o) : drawRock(o);
    drawPlayer();
    drawParticles();

    ctx.fillStyle = "rgba(0,0,0,0.48)";
    ctx.fillRect(0, 0, W, H);

    ctx.textAlign = "center";
    ctx.textBaseline = "middle";

    ctx.fillStyle = "#fff";
    ctx.font = "bold 40px Arial";
    ctx.fillText("Игра окончена", W / 2, 235);

    ctx.font = "bold 26px Arial";
    ctx.fillText("Счёт: " + score, W / 2, 300);

    ctx.fillStyle = "#fff7c2";
    ctx.font = "22px Arial";
    ctx.fillText("Рекорд: " + best, W / 2, 338);

    ctx.fillStyle = "#2563eb";
    roundedRect(90, 395, 240, 72, 22);
    ctx.fill();

    ctx.fillStyle = "#ffffff";
    ctx.font = "bold 30px Arial";
    ctx.fillText("Ещё раз", W / 2, 432);

    drawSoundButtonOnly();
  }

  function drawSoundButtonOnly() {
    ctx.fillStyle = "rgba(0,0,0,0.28)";
    roundedRect(W - 60, 14, 46, 42, 14);
    ctx.fill();

    ctx.fillStyle = "#fff";
    ctx.font = "bold 22px Arial";
    ctx.textAlign = "center";
    ctx.textBaseline = "middle";
    ctx.fillText(muted ? "🔇" : "🔊", W - 37, 36);
  }

  function render() {
    ctx.save();

    if (shake > 0) {
      ctx.translate((Math.random() - 0.5) * shake, (Math.random() - 0.5) * shake);
    }

    if (state === "menu") {
      drawMenu();
      ctx.restore();
      return;
    }

    if (state === "over") {
      drawGameOver();
      ctx.restore();
      return;
    }

    drawBackground();

    for (const c of coins) drawCoin(c);
    for (const b of bonuses) drawBonus(b);
    for (const o of obstacles) o.type === "wolf" ? drawWolf(o) : drawRock(o);

    drawPlayer();
    drawParticles();
    drawUI();

    if (flash > 0) {
      ctx.fillStyle = "rgba(255,255,255," + flash + ")";
      ctx.fillRect(0, 0, W, H);
    }

    ctx.restore();
  }

  function loop(t) {
    const dt = (t - lastTime) / 1000 || 0;
    lastTime = t;
    update(dt);
    render();
    requestAnimationFrame(loop);
  }

  function getPos(e) {
    const rect = canvas.getBoundingClientRect();
    let clientX, clientY;

    if (e.touches && e.touches.length) {
      clientX = e.touches[0].clientX;
      clientY = e.touches[0].clientY;
    } else if (e.changedTouches && e.changedTouches.length) {
      clientX = e.changedTouches[0].clientX;
      clientY = e.changedTouches[0].clientY;
    } else {
      clientX = e.clientX;
      clientY = e.clientY;
    }

    return {
      x: (clientX - rect.left) * (W / rect.width),
      y: (clientY - rect.top) * (H / rect.height)
    };
  }

  function isSoundButton(p) {
    return p.x >= W - 65 && p.x <= W - 8 && p.y >= 8 && p.y <= 65;
  }

  function isMainButton(p) {
    if (state === "menu") return p.x >= 90 && p.x <= 330 && p.y >= 365 && p.y <= 437;
    if (state === "over") return p.x >= 90 && p.x <= 330 && p.y >= 395 && p.y <= 467;
    return false;
  }

  let touchStart = null;

  canvas.addEventListener("pointerdown", e => {
    e.preventDefault();
    const p = getPos(e);
    touchStart = p;

    if (isSoundButton(p)) {
      initAudio();
      muted = !muted;
      return;
    }

    if (isMainButton(p)) {
      startGame();
      return;
    }

    if (state === "play") {
      initAudio();
      jump();
    }
  });

  canvas.addEventListener("pointerup", e => {
    e.preventDefault();
    const p = getPos(e);
    if (!touchStart) return;

    const dy = p.y - touchStart.y;
    if (state === "play" && dy < -25) {
      jump();
    }

    touchStart = null;
  });

  window.addEventListener("keydown", e => {
    initAudio();

    if (e.code === "Space" || e.code === "ArrowUp") {
      if (state === "menu" || state === "over") startGame();
      else jump();
    }

    if (e.code === "KeyM") {
      muted = !muted;
    }
  });

  requestAnimationFrame(loop);
})();
</script>
</body>
</html>
