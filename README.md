<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no"/>
<title>Батыр с твоим лицом</title>

<style>
  :root {
    --vh: 1vh;
  }

  * {
    box-sizing: border-box;
    -webkit-tap-highlight-color: transparent;
  }

  html, body {
    margin: 0;
    padding: 0;
    width: 100%;
    height: calc(var(--vh) * 100);
    overflow: hidden;
    background: #111827;
    font-family: Arial, sans-serif;
    overscroll-behavior: none;
  }

  body {
    display: flex;
    align-items: center;
    justify-content: center;
  }

  #wrap {
    position: relative;
    width: min(100vw, calc(var(--vh) * 100 * 420 / 740), 520px);
    aspect-ratio: 420 / 740;
    max-height: calc(var(--vh) * 100);
    overflow: hidden;
    background: #7dd3fc;
  }

  canvas {
    display: block;
    width: 100%;
    height: 100%;
    touch-action: none;
  }

  #faceInput {
    position: absolute;
    left: -9999px;
    width: 1px;
    height: 1px;
    opacity: 0;
  }

  .uiButton {
    position: absolute;
    left: 19.5%;
    width: 61%;
    height: 8.6%;
    border: none;
    border-radius: 22px;
    color: white;
    font-size: clamp(18px, 5vw, 26px);
    font-weight: 800;
    font-family: Arial, sans-serif;
    text-align: center;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    user-select: none;
    z-index: 20;
    box-shadow: 0 8px 0 rgba(0,0,0,0.22);
    touch-action: manipulation;
  }

  #playBtn {
    top: 45.7%;
    background: #16a34a;
  }

  #uploadLabel {
    top: 56.8%;
    background: #f59e0b;
  }

  #againBtn {
    top: 56.5%;
    background: #2563eb;
  }

  #menuBtn {
    top: 66.5%;
    background: #f59e0b;
  }

  .smallBtn {
    position: absolute;
    top: 1.7%;
    width: 10.5%;
    height: 5.7%;
    border: none;
    border-radius: 14px;
    background: rgba(0,0,0,0.32);
    color: white;
    font-size: clamp(16px, 5vw, 24px);
    font-weight: bold;
    z-index: 25;
    cursor: pointer;
    touch-action: manipulation;
  }

  #soundBtn {
    right: 3%;
  }

  #pauseBtn {
    right: 15%;
  }

  .hidden {
    display: none !important;
  }
</style>
</head>

<body>
<div id="wrap">
  <canvas id="game"></canvas>

  <button id="playBtn" class="uiButton">Играть</button>

  <label id="uploadLabel" class="uiButton" for="faceInput">
    Загрузить лицо
  </label>

  <button id="againBtn" class="uiButton hidden">Ещё раз</button>
  <button id="menuBtn" class="uiButton hidden">В меню</button>

  <button id="pauseBtn" class="smallBtn">Ⅱ</button>
  <button id="soundBtn" class="smallBtn">♪</button>

  <input id="faceInput" type="file" accept="image/*">
</div>

<script>
(() => {
  function setRealVh() {
    document.documentElement.style.setProperty("--vh", window.innerHeight * 0.01 + "px");
  }

  setRealVh();
  window.addEventListener("resize", setRealVh);
  window.addEventListener("orientationchange", setRealVh);

  const canvas = document.getElementById("game");
  const ctx = canvas.getContext("2d");

  const faceInput = document.getElementById("faceInput");
  const playBtn = document.getElementById("playBtn");
  const uploadLabel = document.getElementById("uploadLabel");
  const againBtn = document.getElementById("againBtn");
  const menuBtn = document.getElementById("menuBtn");
  const soundBtn = document.getElementById("soundBtn");
  const pauseBtn = document.getElementById("pauseBtn");

  const W = 420;
  const H = 740;
  const GROUND = 610;

  canvas.width = W;
  canvas.height = H;

  let state = "menu";
  let lastTime = 0;

  let score = 0;
  let best = Number(localStorage.getItem("batyr_face_runner_best") || 0);
  let speed = 250;
  let distance = 0;
  let bgMove = 0;
  let shake = 0;
  let flash = 0;
  let muted = false;
  let combo = 0;
  let maxCombo = 0;

  let audio = null;

  let faceImg = new Image();
  let hasFace = false;

  const savedFace = localStorage.getItem("batyr_face_runner_photo");
  if (savedFace) {
    faceImg.onload = () => {
      hasFace = true;
      uploadLabel.textContent = "Сменить лицо";
    };
    faceImg.src = savedFace;
  }

  const player = {
    x: 78,
    y: GROUND,
    vy: 0,
    jumps: 0,
    shield: 0,
    magnet: 0,
    inv: 0,
    fastFall: false
  };

  let obstacles = [];
  let coins = [];
  let bonuses = [];
  let particles = [];
  let clouds = [];
  let birds = [];

  let obstacleTimer = 0.8;
  let coinTimer = 0.5;
  let bonusTimer = 7;

  for (let i = 0; i < 7; i++) {
    clouds.push({
      x: Math.random() * W,
      y: 50 + Math.random() * 170,
      s: 0.55 + Math.random() * 0.95,
      v: 8 + Math.random() * 18
    });
  }

  for (let i = 0; i < 5; i++) {
    birds.push({
      x: Math.random() * W,
      y: 110 + Math.random() * 190,
      v: 20 + Math.random() * 25,
      p: Math.random() * 10
    });
  }

  function initAudio() {
    if (!audio) {
      audio = new (window.AudioContext || window.webkitAudioContext)();
    }

    if (audio.state === "suspended") {
      audio.resume();
    }
  }

  function beep(type) {
    if (muted || !audio) return;

    const t = audio.currentTime;
    const osc = audio.createOscillator();
    const gain = audio.createGain();

    osc.connect(gain);
    gain.connect(audio.destination);

    if (type === "jump") {
      osc.type = "triangle";
      osc.frequency.setValueAtTime(250, t);
      osc.frequency.exponentialRampToValueAtTime(600, t + 0.13);
      gain.gain.setValueAtTime(0.11, t);
      gain.gain.exponentialRampToValueAtTime(0.001, t + 0.17);
      osc.start(t);
      osc.stop(t + 0.18);
    }

    if (type === "coin") {
      osc.type = "sine";
      osc.frequency.setValueAtTime(720, t);
      osc.frequency.exponentialRampToValueAtTime(1250, t + 0.08);
      gain.gain.setValueAtTime(0.075, t);
      gain.gain.exponentialRampToValueAtTime(0.001, t + 0.13);
      osc.start(t);
      osc.stop(t + 0.14);
    }

    if (type === "bonus") {
      osc.type = "square";
      osc.frequency.setValueAtTime(420, t);
      osc.frequency.setValueAtTime(680, t + 0.08);
      osc.frequency.setValueAtTime(920, t + 0.16);
      gain.gain.setValueAtTime(0.085, t);
      gain.gain.exponentialRampToValueAtTime(0.001, t + 0.33);
      osc.start(t);
      osc.stop(t + 0.34);
    }

    if (type === "hit") {
      osc.type = "sawtooth";
      osc.frequency.setValueAtTime(160, t);
      osc.frequency.exponentialRampToValueAtTime(45, t + 0.22);
      gain.gain.setValueAtTime(0.15, t);
      gain.gain.exponentialRampToValueAtTime(0.001, t + 0.27);
      osc.start(t);
      osc.stop(t + 0.28);
    }
  }

  function vibrate(ms) {
    if (navigator.vibrate) navigator.vibrate(ms);
  }

  faceInput.addEventListener("change", e => {
    const file = e.target.files && e.target.files[0];
    if (!file) return;

    const reader = new FileReader();

    reader.onload = ev => {
      const img = new Image();

      img.onload = () => {
        const size = 320;
        const off = document.createElement("canvas");
        const octx = off.getContext("2d");

        off.width = size;
        off.height = size;

        const scale = Math.max(size / img.width, size / img.height);
        const sw = size / scale;
        const sh = size / scale;
        const sx = (img.width - sw) / 2;
        const sy = (img.height - sh) / 2;

        octx.drawImage(img, sx, sy, sw, sh, 0, 0, size, size);

        const data = off.toDataURL("image/jpeg", 0.86);

        faceImg = new Image();
        faceImg.onload = () => {
          hasFace = true;
          uploadLabel.textContent = "Сменить лицо";
        };
        faceImg.src = data;

        try {
          localStorage.setItem("batyr_face_runner_photo", data);
        } catch (err) {
          console.log("Фото работает, но не сохранилось: мало места в localStorage.");
        }
      };

      img.src = ev.target.result;
    };

    reader.readAsDataURL(file);
    faceInput.value = "";
  });

  function resetGame() {
    score = 0;
    speed = 250;
    distance = 0;
    bgMove = 0;
    shake = 0;
    flash = 0;
    combo = 0;
    maxCombo = 0;

    player.y = GROUND;
    player.vy = 0;
    player.jumps = 0;
    player.shield = 0;
    player.magnet = 0;
    player.inv = 0;
    player.fastFall = false;

    obstacles = [];
    coins = [];
    bonuses = [];
    particles = [];

    obstacleTimer = 0.9;
    coinTimer = 0.5;
    bonusTimer = 5 + Math.random() * 4;
  }

  function startGame() {
    initAudio();
    resetGame();
    state = "play";
    updateButtons();
  }

  function gameOver() {
    state = "over";
    shake = 18;
    flash = 0.45;
    vibrate(120);
    beep("hit");

    const finalScore = Math.floor(score);
    if (finalScore > best) {
      best = finalScore;
      localStorage.setItem("batyr_face_runner_best", String(best));
    }

    updateButtons();
  }

  function goMenu() {
    state = "menu";
    updateButtons();
  }

  function togglePause() {
    if (state === "play") {
      state = "pause";
    } else if (state === "pause") {
      state = "play";
    }

    updateButtons();
  }

  function updateButtons() {
    playBtn.classList.toggle("hidden", state !== "menu");
    uploadLabel.classList.toggle("hidden", state !== "menu");

    againBtn.classList.toggle("hidden", state !== "over");
    menuBtn.classList.toggle("hidden", state !== "over");

    pauseBtn.classList.toggle("hidden", state === "menu" || state === "over");

    pauseBtn.textContent = state === "pause" ? "▶" : "Ⅱ";
    soundBtn.textContent = muted ? "×" : "♪";
    uploadLabel.textContent = hasFace ? "Сменить лицо" : "Загрузить лицо";
  }

  playBtn.addEventListener("click", () => {
    startGame();
  });

  againBtn.addEventListener("click", () => {
    startGame();
  });

  menuBtn.addEventListener("click", () => {
    goMenu();
  });

  pauseBtn.addEventListener("click", () => {
    initAudio();
    togglePause();
  });

  soundBtn.addEventListener("click", () => {
    initAudio();
    muted = !muted;
    updateButtons();
  });

  uploadLabel.addEventListener("click", () => {
    initAudio();
  });

  function jump() {
    if (state !== "play") return;

    if (player.jumps < 2) {
      player.vy = player.jumps === 0 ? -730 : -630;
      player.jumps++;
      player.fastFall = false;
      dust(player.x + 35, player.y - 4, 12);
      beep("jump");
    }
  }

  function fastFall() {
    if (state !== "play") return;

    if (player.y < GROUND - 20) {
      player.vy = 980;
      player.fastFall = true;
      spark(player.x + 38, player.y - 55, 12, "rgba(255,255,255,");
    }
  }

  function playerRect() {
    return {
      x: player.x + 13,
      y: player.y - 102,
      w: 54,
      h: 94
    };
  }

  function overlap(a, b) {
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
    return dx * dx + dy * dy <= cr * cr;
  }

  function dust(x, y, n) {
    for (let i = 0; i < n; i++) {
      particles.push({
        x,
        y,
        vx: -60 - Math.random() * 170,
        vy: -25 - Math.random() * 90,
        r: 2 + Math.random() * 4,
        life: 0.28 + Math.random() * 0.28,
        color: "rgba(196,135,54,"
      });
    }
  }

  function spark(x, y, n, color) {
    for (let i = 0; i < n; i++) {
      const a = Math.random() * Math.PI * 2;
      const s = 70 + Math.random() * 240;

      particles.push({
        x,
        y,
        vx: Math.cos(a) * s,
        vy: Math.sin(a) * s,
        r: 2 + Math.random() * 4,
        life: 0.3 + Math.random() * 0.4,
        color
      });
    }
  }

  function spawnObstacle() {
    const r = Math.random();
    let type = "wolf";

    if (distance > 1800 && r < 0.25) {
      type = "eagle";
    } else if (r < 0.5) {
      type = "rock";
    }

    if (type === "wolf") {
      obstacles.push({
        type,
        x: W + 50,
        y: GROUND,
        w: 74,
        h: 50,
        p: Math.random() * 10
      });
    }

    if (type === "rock") {
      obstacles.push({
        type,
        x: W + 50,
        y: GROUND,
        w: 52,
        h: 56,
        p: Math.random() * 10
      });
    }

    if (type === "eagle") {
      obstacles.push({
        type,
        x: W + 50,
        y: 420 + Math.random() * 45,
        w: 76,
        h: 44,
        p: Math.random() * 10
      });
    }
  }

  function spawnCoins() {
    const count = 4 + Math.floor(Math.random() * 4);
    const baseX = W + 40;
    const baseY = 455 + Math.random() * 70;
    const pattern = Math.random();

    for (let i = 0; i < count; i++) {
      let y = baseY;

      if (pattern < 0.4) {
        y = baseY - Math.sin(i * 0.85) * 35;
      } else if (pattern < 0.75) {
        y = baseY - i * 12;
      } else {
        y = baseY + (i % 2) * 30;
      }

      coins.push({
        x: baseX + i * 43,
        y,
        r: 13,
        rot: Math.random() * 6,
        gold: Math.random() < 0.08
      });
    }
  }

  function spawnBonus() {
    bonuses.push({
      kind: Math.random() < 0.55 ? "shield" : "magnet",
      x: W + 55,
      y: 390 + Math.random() * 95,
      r: 18,
      rot: 0
    });
  }

  function update(dt) {
    if (state !== "play") return;

    dt = Math.min(dt, 0.033);

    distance += speed * dt;
    speed = 250 + Math.min(285, distance / 15);
    bgMove += speed * dt;

    score += dt * 6;

    obstacleTimer -= dt;
    coinTimer -= dt;
    bonusTimer -= dt;

    if (obstacleTimer <= 0) {
      spawnObstacle();
      obstacleTimer = Math.max(0.58, 1.35 - speed / 720 + Math.random() * 0.45);
    }

    if (coinTimer <= 0) {
      spawnCoins();
      coinTimer = 1.0 + Math.random() * 1.2;
    }

    if (bonusTimer <= 0) {
      spawnBonus();
      bonusTimer = 8 + Math.random() * 7;
    }

    player.vy += (player.fastFall ? 2550 : 1800) * dt;
    player.y += player.vy * dt;

    if (player.y >= GROUND) {
      if (player.jumps > 0 && player.vy > 500) {
        dust(player.x + 35, GROUND - 2, 14);

        if (player.fastFall) {
          shake = 5;
          vibrate(25);
        }
      }

      player.y = GROUND;
      player.vy = 0;
      player.jumps = 0;
      player.fastFall = false;
    }

    if (player.shield > 0) player.shield -= dt;
    if (player.magnet > 0) player.magnet -= dt;
    if (player.inv > 0) player.inv -= dt;
    if (shake > 0) shake -= dt * 28;
    if (flash > 0) flash -= dt;

    for (const o of obstacles) {
      o.x -= speed * dt;
      o.p += dt * 7;
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

        if (d < 180) {
          c.x += dx * dt * 5.8;
          c.y += dy * dt * 5.8;
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
      p.vy += 300 * dt;
    }

    for (const c of clouds) {
      c.x -= c.v * dt;
      if (c.x < -130) {
        c.x = W + 120;
        c.y = 50 + Math.random() * 170;
        c.s = 0.55 + Math.random() * 0.95;
      }
    }

    for (const b of birds) {
      b.x -= b.v * dt;
      b.p += dt * 8;

      if (b.x < -70) {
        b.x = W + 80;
        b.y = 100 + Math.random() * 200;
      }
    }

    obstacles = obstacles.filter(o => o.x + o.w > -80);
    coins = coins.filter(c => c.x > -60 && !c.dead);
    bonuses = bonuses.filter(b => b.x > -60 && !b.dead);
    particles = particles.filter(p => p.life > 0);

    const pr = playerRect();

    for (const c of coins) {
      if (!c.dead && circleRect(c.x, c.y, c.r, pr)) {
        c.dead = true;

        if (c.gold) {
          score += 60 + combo * 2;
          combo += 3;
          spark(c.x, c.y, 22, "rgba(255,215,0,");
          beep("bonus");
          shake = 3;
        } else {
          score += 12 + Math.min(30, combo);
          combo++;
          spark(c.x, c.y, 8, "rgba(255,210,55,");
          beep("coin");
        }

        if (combo > maxCombo) maxCombo = combo;
      }
    }

    for (const b of bonuses) {
      if (!b.dead && circleRect(b.x, b.y, b.r, pr)) {
        b.dead = true;

        if (b.kind === "shield") player.shield = 7;
        if (b.kind === "magnet") player.magnet = 8;

        score += 45;
        spark(
          b.x,
          b.y,
          25,
          b.kind === "shield" ? "rgba(80,240,255," : "rgba(190,130,255,"
        );

        beep("bonus");
        shake = 5;
        vibrate(35);
      }
    }

    for (const o of obstacles) {
      let r;

      if (o.type === "eagle") {
        r = {
          x: o.x + 8,
          y: o.y - o.h / 2,
          w: o.w - 14,
          h: o.h
        };
      } else {
        r = {
          x: o.x + 8,
          y: o.y - o.h + 7,
          w: o.w - 14,
          h: o.h - 9
        };
      }

      if (overlap(pr, r)) {
        if (player.shield > 0 && player.inv <= 0) {
          player.shield = Math.max(0, player.shield - 3);
          player.inv = 0.9;
          o.x -= 140;
          combo = 0;
          shake = 11;
          flash = 0.18;
          spark(pr.x + pr.w / 2, pr.y + pr.h / 2, 28, "rgba(80,240,255,");
          beep("bonus");
          vibrate(60);
        } else if (player.inv <= 0) {
          combo = 0;
          gameOver();
        }
      }
    }
  }

  function rr(x, y, w, h, r) {
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

  function drawImageCover(img, x, y, w, h) {
    const iw = img.width;
    const ih = img.height;

    if (!iw || !ih) return;

    const scale = Math.max(w / iw, h / ih);
    const sw = w / scale;
    const sh = h / scale;
    const sx = (iw - sw) / 2;
    const sy = (ih - sh) / 2;

    ctx.drawImage(img, sx, sy, sw, sh, x, y, w, h);
  }

  function drawBackground() {
    const phase = Math.floor(distance / 2200) % 4;

    let top = "#60c9ff";
    let mid = "#b8f0ff";
    let low = "#ffe9a3";

    if (phase === 1) {
      top = "#38bdf8";
      mid = "#fde68a";
      low = "#fb923c";
    }

    if (phase === 2) {
      top = "#312e81";
      mid = "#7c3aed";
      low = "#fb7185";
    }

    if (phase === 3) {
      top = "#0f172a";
      mid = "#1e3a8a";
      low = "#64748b";
    }

    const g = ctx.createLinearGradient(0, 0, 0, H);
    g.addColorStop(0, top);
    g.addColorStop(0.56, mid);
    g.addColorStop(1, low);

    ctx.fillStyle = g;
    ctx.fillRect(0, 0, W, H);

    if (phase === 3) {
      ctx.fillStyle = "#fef3c7";
      ctx.beginPath();
      ctx.arc(335, 100, 34, 0, Math.PI * 2);
      ctx.fill();

      ctx.fillStyle = "rgba(255,255,255,0.9)";
      for (let i = 0; i < 35; i++) {
        const x = (i * 73 + bgMove * 0.02) % W;
        const y = 30 + (i * 41) % 260;
        ctx.beginPath();
        ctx.arc(x, y, 1.4, 0, Math.PI * 2);
        ctx.fill();
      }
    } else {
      ctx.fillStyle = "#ffd34d";
      ctx.beginPath();
      ctx.arc(335, 105, 42, 0, Math.PI * 2);
      ctx.fill();
    }

    ctx.fillStyle = "rgba(255,255,255,0.82)";
    for (const c of clouds) {
      ctx.save();
      ctx.translate(c.x, c.y);
      ctx.scale(c.s, c.s);

      ctx.beginPath();
      ctx.arc(0, 15, 22, 0, Math.PI * 2);
      ctx.arc(25, 5, 28, 0, Math.PI * 2);
      ctx.arc(55, 15, 21, 0, Math.PI * 2);
      ctx.arc(25, 24, 30, 0, Math.PI * 2);
      ctx.fill();

      ctx.restore();
    }

    ctx.strokeStyle = "rgba(30,41,59,0.6)";
    ctx.lineWidth = 3;

    for (const b of birds) {
      const wing = Math.sin(b.p) * 7;

      ctx.beginPath();
      ctx.moveTo(b.x, b.y);
      ctx.quadraticCurveTo(b.x + 10, b.y - wing, b.x + 20, b.y);
      ctx.quadraticCurveTo(b.x + 30, b.y - wing, b.x + 40, b.y);
      ctx.stroke();
    }

    ctx.fillStyle = phase === 3 ? "#334155" : "#79b5d2";
    ctx.beginPath();
    ctx.moveTo(0, 365);
    ctx.lineTo(75, 250);
    ctx.lineTo(155, 365);
    ctx.lineTo(230, 255);
    ctx.lineTo(320, 365);
    ctx.lineTo(420, 240);
    ctx.lineTo(420, 440);
    ctx.lineTo(0, 440);
    ctx.closePath();
    ctx.fill();

    ctx.fillStyle = phase === 3 ? "#475569" : "#e8b45b";
    ctx.fillRect(0, 430, W, H - 430);

    ctx.fillStyle = phase === 3 ? "#166534" : "#5eb84f";
    ctx.fillRect(0, GROUND, W, H - GROUND);

    ctx.fillStyle = "rgba(30,100,35,0.3)";
    for (let x = -60 - (bgMove * 0.45) % 85; x < W + 85; x += 85) {
      ctx.beginPath();
      ctx.ellipse(x, GROUND + 32, 62, 17, 0, 0, Math.PI * 2);
      ctx.fill();
    }

    ctx.strokeStyle = "rgba(92,57,20,0.35)";
    ctx.lineWidth = 3;

    for (let x = -70 - (bgMove * 0.85) % 90; x < W + 90; x += 90) {
      ctx.beginPath();
      ctx.moveTo(x, GROUND + 14);
      ctx.quadraticCurveTo(x + 25, GROUND + 31, x + 55, GROUND + 14);
      ctx.stroke();
    }

    ctx.fillStyle = "#f472b6";
    for (let x = -20 - (bgMove * 1.2) % 70; x < W + 70; x += 70) {
      ctx.beginPath();
      ctx.arc(x, GROUND + 55, 4, 0, Math.PI * 2);
      ctx.fill();
    }
  }

  function drawFace(cx, cy, r) {
    ctx.save();

    ctx.fillStyle = "#f3b35b";
    ctx.beginPath();
    ctx.arc(cx, cy, r + 3, 0, Math.PI * 2);
    ctx.fill();

    ctx.beginPath();
    ctx.arc(cx, cy, r, 0, Math.PI * 2);
    ctx.clip();

    if (hasFace && faceImg.complete) {
      drawImageCover(faceImg, cx - r, cy - r, r * 2, r * 2);
    } else {
      ctx.fillStyle = "#f3b35b";
      ctx.fillRect(cx - r, cy - r, r * 2, r * 2);

      ctx.fillStyle = "#111";
      ctx.beginPath();
      ctx.arc(cx - 7, cy - 3, 2.4, 0, Math.PI * 2);
      ctx.arc(cx + 7, cy - 3, 2.4, 0, Math.PI * 2);
      ctx.fill();

      ctx.strokeStyle = "#7c2d12";
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.arc(cx, cy + 4, 7, 0.2, Math.PI - 0.2);
      ctx.stroke();
    }

    ctx.restore();

    ctx.strokeStyle = "#fff";
    ctx.lineWidth = 3;
    ctx.beginPath();
    ctx.arc(cx, cy, r + 1.5, 0, Math.PI * 2);
    ctx.stroke();

    ctx.fillStyle = "#d71920";
    ctx.beginPath();
    ctx.moveTo(cx - 19, cy - 17);
    ctx.lineTo(cx + 19, cy - 17);
    ctx.lineTo(cx + 10, cy - 35);
    ctx.lineTo(cx - 10, cy - 35);
    ctx.closePath();
    ctx.fill();

    ctx.strokeStyle = "#facc15";
    ctx.lineWidth = 3;
    ctx.beginPath();
    ctx.moveTo(cx - 12, cy - 23);
    ctx.lineTo(cx + 12, cy - 23);
    ctx.stroke();
  }

  function drawPlayer() {
    const x = player.x;
    const y = player.y;
    const t = performance.now();
    const bob = player.jumps === 0 ? Math.sin(t / 85) * 2 : 0;

    if (player.shield > 0) {
      ctx.save();
      ctx.globalAlpha = 0.32 + Math.sin(t / 90) * 0.12;
      ctx.strokeStyle = "#38e8ff";
      ctx.lineWidth = 5;
      ctx.beginPath();
      ctx.arc(x + 38, y - 54, 62, 0, Math.PI * 2);
      ctx.stroke();
      ctx.restore();
    }

    if (player.inv > 0 && Math.floor(t / 80) % 2 === 0) return;

    ctx.save();
    ctx.translate(x, y + bob);

    ctx.fillStyle = "#7a3f20";
    ctx.beginPath();
    ctx.ellipse(36, -35, 41, 24, 0, 0, Math.PI * 2);
    ctx.fill();

    ctx.fillStyle = "#8d4b27";
    ctx.beginPath();
    ctx.ellipse(72, -54, 19, 14, -0.3, 0, Math.PI * 2);
    ctx.fill();

    ctx.fillStyle = "#3b1d12";
    ctx.beginPath();
    ctx.moveTo(58, -65);
    ctx.lineTo(68, -85);
    ctx.lineTo(73, -62);
    ctx.fill();

    ctx.fillStyle = "#111827";
    ctx.beginPath();
    ctx.arc(79, -58, 2.5, 0, Math.PI * 2);
    ctx.fill();

    ctx.strokeStyle = "#4a2614";
    ctx.lineWidth = 6;

    const leg = Math.sin(t / 65) * 9;

    ctx.beginPath();
    ctx.moveTo(12, -18);
    ctx.lineTo(7 + leg, 0);
    ctx.moveTo(33, -17);
    ctx.lineTo(30 - leg, 0);
    ctx.moveTo(51, -18);
    ctx.lineTo(52 + leg, 0);
    ctx.moveTo(66, -20);
    ctx.lineTo(69 - leg, 0);
    ctx.stroke();

    ctx.strokeStyle = "#2b150d";
    ctx.lineWidth = 5;
    ctx.beginPath();
    ctx.moveTo(0, -43);
    ctx.quadraticCurveTo(-24, -52, -18, -72);
    ctx.stroke();

    ctx.fillStyle = "#2563eb";
    rr(23, -72, 26, 38, 6);
    ctx.fill();

    ctx.strokeStyle = "#f7d46b";
    ctx.lineWidth = 4;
    ctx.beginPath();
    ctx.moveTo(47, -65);
    ctx.lineTo(69, -42);
    ctx.stroke();

    drawFace(37, -89, 22);

    ctx.restore();
  }

  function drawWolf(o) {
    const run = Math.sin(o.p) * 4;

    ctx.save();
    ctx.translate(o.x, o.y);

    ctx.fillStyle = "#515866";
    ctx.beginPath();
    ctx.ellipse(30, -28, 32, 17, 0, 0, Math.PI * 2);
    ctx.fill();

    ctx.fillStyle = "#626b7a";
    ctx.beginPath();
    ctx.ellipse(60, -39, 18, 13, -0.2, 0, Math.PI * 2);
    ctx.fill();

    ctx.fillStyle = "#515866";
    ctx.beginPath();
    ctx.moveTo(53, -51);
    ctx.lineTo(57, -68);
    ctx.lineTo(65, -51);
    ctx.fill();

    ctx.beginPath();
    ctx.moveTo(66, -50);
    ctx.lineTo(78, -62);
    ctx.lineTo(74, -43);
    ctx.fill();

    ctx.fillStyle = "#111827";
    ctx.beginPath();
    ctx.arc(67, -41, 2.5, 0, Math.PI * 2);
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
    ctx.lineTo(28, -55);
    ctx.lineTo(52, -35);
    ctx.lineTo(46, 0);
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

  function drawEagle(o) {
    const flap = Math.sin(o.p) * 16;

    ctx.save();
    ctx.translate(o.x, o.y);

    ctx.fillStyle = "#78350f";

    ctx.beginPath();
    ctx.moveTo(36, 0);
    ctx.quadraticCurveTo(8, -flap, 0, 5);
    ctx.quadraticCurveTo(18, 10, 36, 4);
    ctx.fill();

    ctx.beginPath();
    ctx.moveTo(36, 0);
    ctx.quadraticCurveTo(66, -flap, 78, 5);
    ctx.quadraticCurveTo(55, 12, 36, 4);
    ctx.fill();

    ctx.fillStyle = "#92400e";
    ctx.beginPath();
    ctx.ellipse(38, 3, 20, 11, 0, 0, Math.PI * 2);
    ctx.fill();

    ctx.fillStyle = "#f59e0b";
    ctx.beginPath();
    ctx.moveTo(57, 0);
    ctx.lineTo(72, 5);
    ctx.lineTo(57, 10);
    ctx.closePath();
    ctx.fill();

    ctx.fillStyle = "#111";
    ctx.beginPath();
    ctx.arc(47, 0, 2.5, 0, Math.PI * 2);
    ctx.fill();

    ctx.restore();
  }

  function drawCoin(c) {
    ctx.save();
    ctx.translate(c.x, c.y);
    ctx.rotate(c.rot);

    ctx.fillStyle = c.gold ? "#facc15" : "#fbbf24";
    ctx.beginPath();
    ctx.ellipse(0, 0, c.gold ? 18 : 14, c.gold ? 14 : 11, 0, 0, Math.PI * 2);
    ctx.fill();

    ctx.strokeStyle = "#fff7ad";
    ctx.lineWidth = 3;
    ctx.beginPath();
    ctx.ellipse(0, 0, c.gold ? 10 : 8, c.gold ? 8 : 6, 0, 0, Math.PI * 2);
    ctx.stroke();

    ctx.fillStyle = "#7c3f00";
    ctx.font = "bold 12px Arial";
    ctx.textAlign = "center";
    ctx.textBaseline = "middle";
    ctx.fillText(c.gold ? "★" : "Б", 0, 1);

    ctx.restore();
  }

  function drawBonus(b) {
    ctx.save();
    ctx.translate(b.x, b.y);
    ctx.rotate(b.rot);

    ctx.fillStyle = b.kind === "shield" ? "#22d3ee" : "#a855f7";

    ctx.beginPath();
    for (let i = 0; i < 12; i++) {
      const r = i % 2 === 0 ? 22 : 10;
      const a = i * Math.PI / 6;
      ctx.lineTo(Math.cos(a) * r, Math.sin(a) * r);
    }
    ctx.closePath();
    ctx.fill();

    ctx.fillStyle = "#fff";
    ctx.font = "bold 18px Arial";
    ctx.textAlign = "center";
    ctx.textBaseline = "middle";
    ctx.fillText(b.kind === "shield" ? "Щ" : "М", 0, 1);

    ctx.restore();
  }

  function drawParticles() {
    for (const p of particles) {
      const a = Math.max(0, Math.min(1, p.life * 2.3));
      ctx.fillStyle = p.color + a + ")";
      ctx.beginPath();
      ctx.arc(p.x, p.y, p.r, 0, Math.PI * 2);
      ctx.fill();
    }
  }

  function drawTopUI() {
    ctx.fillStyle = "rgba(0,0,0,0.28)";
    rr(12, 12, 155, 44, 14);
    ctx.fill();

    ctx.fillStyle = "#fff";
    ctx.font = "bold 23px Arial";
    ctx.textAlign = "left";
    ctx.textBaseline = "middle";
    ctx.fillText("Счёт: " + Math.floor(score), 24, 34);

    if (combo > 1) {
      ctx.fillStyle = "rgba(255,193,7,0.95)";
      rr(12, 62, 122, 32, 12);
      ctx.fill();

      ctx.fillStyle = "#592d00";
      ctx.font = "bold 17px Arial";
      ctx.fillText("Комбо x" + combo, 24, 78);
    }

    if (player.shield > 0 || player.magnet > 0) {
      const y = combo > 1 ? 100 : 62;

      ctx.fillStyle = "rgba(0,0,0,0.25)";
      rr(12, y, 175, 34, 12);
      ctx.fill();

      ctx.fillStyle = "#cffafe";
      ctx.font = "bold 15px Arial";

      let text = "";
      if (player.shield > 0) text += "Щит " + Math.ceil(player.shield) + "с ";
      if (player.magnet > 0) text += "Магнит " + Math.ceil(player.magnet) + "с";

      ctx.fillText(text, 24, y + 17);
    }
  }

  function drawScene() {
    drawBackground();

    for (const c of coins) drawCoin(c);
    for (const b of bonuses) drawBonus(b);

    for (const o of obstacles) {
      if (o.type === "wolf") drawWolf(o);
      if (o.type === "rock") drawRock(o);
      if (o.type === "eagle") drawEagle(o);
    }

    drawPlayer();
    drawParticles();

    if (state === "play") {
      drawTopUI();
    }

    if (flash > 0) {
      ctx.fillStyle = "rgba(255,255,255," + flash + ")";
      ctx.fillRect(0, 0, W, H);
    }
  }

  function drawMenu() {
    drawBackground();

    ctx.fillStyle = "rgba(0,0,0,0.42)";
    ctx.fillRect(0, 0, W, H);

    ctx.textAlign = "center";
    ctx.textBaseline = "middle";

    ctx.fillStyle = "#fff7c2";
    ctx.font = "bold 38px Arial";
    ctx.fillText("Батыр", W / 2, 128);
    ctx.fillText("с твоим лицом", W / 2, 174);

    ctx.fillStyle = "#fff";
    ctx.font = "18px Arial";
    ctx.fillText("Тап — прыжок", W / 2, 232);
    ctx.fillText("Свайп вниз — быстро упасть", W / 2, 258);

    ctx.save();
    ctx.translate(115, 525);
    drawPlayer();
    ctx.restore();

    ctx.font = "17px Arial";
    ctx.fillStyle = "rgba(255,255,255,0.92)";
    ctx.fillText("Рекорд: " + best, W / 2, 632);
    ctx.fillText("Фото хранится только в этом браузере", W / 2, 662);
  }

  function drawPause() {
    drawScene();

    ctx.fillStyle = "rgba(0,0,0,0.5)";
    ctx.fillRect(0, 0, W, H);

    ctx.textAlign = "center";
    ctx.textBaseline = "middle";

    ctx.fillStyle = "#fff";
    ctx.font = "bold 44px Arial";
    ctx.fillText("Пауза", W / 2, 320);

    ctx.font = "19px Arial";
    ctx.fillText("Нажми ▶ сверху, чтобы продолжить", W / 2, 370);
  }

  function drawGameOver() {
    drawScene();

    ctx.fillStyle = "rgba(0,0,0,0.58)";
    ctx.fillRect(0, 0, W, H);

    ctx.textAlign = "center";
    ctx.textBaseline = "middle";

    ctx.fillStyle = "#fff";
    ctx.font = "bold 38px Arial";
    ctx.fillText("Игра окончена", W / 2, 215);

    ctx.font = "bold 27px Arial";
    ctx.fillText("Счёт: " + Math.floor(score), W / 2, 286);

    ctx.fillStyle = "#fff7c2";
    ctx.font = "21px Arial";
    ctx.fillText("Рекорд: " + best, W / 2, 326);
    ctx.fillText("Лучшее комбо: x" + maxCombo, W / 2, 362);
  }

  function render() {
    updateButtons();

    ctx.save();

    if (shake > 0) {
      ctx.translate((Math.random() - 0.5) * shake, (Math.random() - 0.5) * shake);
    }

    if (state === "menu") drawMenu();
    if (state === "play") drawScene();
    if (state === "pause") drawPause();
    if (state === "over") drawGameOver();

    ctx.restore();
  }

  function loop(t) {
    const dt = (t - lastTime) / 1000 || 0;
    lastTime = t;

    update(dt);
    render();

    requestAnimationFrame(loop);
  }

  let startPoint = null;

  function getCanvasPos(e) {
    const rect = canvas.getBoundingClientRect();

    return {
      x: (e.clientX - rect.left) * (W / rect.width),
      y: (e.clientY - rect.top) * (H / rect.height)
    };
  }

  canvas.addEventListener("pointerdown", e => {
    e.preventDefault();
    initAudio();

    startPoint = getCanvasPos(e);

    if (state === "play") {
      jump();
    }
  });

  canvas.addEventListener("pointerup", e => {
    e.preventDefault();

    if (!startPoint) return;

    const p = getCanvasPos(e);
    const dx = p.x - startPoint.x;
    const dy = p.y - startPoint.y;

    if (state === "play") {
      if (dy > 32 && Math.abs(dy) > Math.abs(dx)) {
        fastFall();
      }

      if (dy < -32 && Math.abs(dy) > Math.abs(dx)) {
        jump();
      }
    }

    startPoint = null;
  });

  window.addEventListener("keydown", e => {
    initAudio();

    if (e.code === "Space" || e.code === "ArrowUp") {
      if (state === "menu" || state === "over") {
        startGame();
      } else if (state === "pause") {
        togglePause();
      } else {
        jump();
      }
    }

    if (e.code === "ArrowDown") {
      fastFall();
    }

    if (e.code === "KeyP") {
      togglePause();
    }

    if (e.code === "KeyM") {
      muted = !muted;
      updateButtons();
    }
  });

  updateButtons();
  requestAnimationFrame(loop);
})();
</script>
</body>
</html>
