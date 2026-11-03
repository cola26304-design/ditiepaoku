# ditiepaoku
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>跑酷｜本地最高分版</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; font-family: system-ui, sans-serif; }
    body { background: #0f141c; color: #eef; overflow: hidden; touch-action: none; }
    #gameWrap { width: 100vw; height: 100vh; display: block; }
    #gameCanvas { display: block; background: #111827; }
    .ui { position: absolute; left: 16px; top: 16px; background: rgba(0,0,0,0.55); padding: 12px 14px; border-radius: 14px; backdrop-filter: blur(8px); }
    #scoreText { font-size: 22px; font-weight: bold; color: #4cc9f0; }
    #timerText { font-size: 18px; color: #f72585; margin-top: 4px; }
    #bestText { font-size: 16px; color: #22c55e; margin-top: 4px; }
    #resetBestBtn { position: absolute; right: 16px; top: 16px; padding: 10px 14px; border: none; border-radius: 12px; background: #991b1b; color: #fff; font-weight: bold; cursor: pointer; }
    #restartBtn { position: absolute; right: 16px; top: 72px; padding: 10px 14px; border: none; border-radius: 12px; background: #1d4ed8; color: #fff; font-weight: bold; cursor: pointer; }
    .modal { position: absolute; inset: 0; background: rgba(0,0,0,0.78); display: flex; align-items: center; justify-content: center; z-index: 100; }
    .modal.hidden { display: none; }
    .box { background: #182030; padding: 26px; border-radius: 18px; width: min(92%, 460px); }
    .box h2 { margin-bottom: 18px; color: #4cc9f0; }
    .stat-row { display: flex; justify-content: space-between; padding: 10px 0; border-bottom: 1px solid #222c3f; }
    .stat-label { color: #aab; }
    .stat-value { font-weight: bold; color: #fff; }
    .row { display: flex; flex-direction: column; gap: 12px; margin-top: 18px; }
    .btn { padding: 12px 14px; border: none; border-radius: 10px; font-weight: bold; cursor: pointer; }
    .btn-primary { background: #2196f3; color: #fff; }
    .btn-green { background: #22c55e; color: #000; }
    .btn-gray { background: #333; color: #ddd; }
  </style>
</head>
<body>
  <div id="gameWrap">
    <canvas id="gameCanvas"></canvas>
    <div class="ui">
      <div id="scoreText">本局：0</div>
      <div id="timerText">倒计时：60</div>
      <div id="bestText">🏆 最高分：0</div>
    </div>
    <button id="resetBestBtn">重置最高分</button>
    <button id="restartBtn">重新开始</button>
  </div>

  <div id="gameOverModal" class="modal hidden">
    <div class="box">
      <h2>🏁 游戏结束</h2>
      <div class="stat-row">
        <span class="stat-label">本局得分</span>
        <span id="finalScoreVal" class="stat-value">0</span>
      </div>
      <div class="stat-row">
        <span class="stat-label">历史最高分</span>
        <span id="bestScoreVal" class="stat-value">0</span>
      </div>
      <div class="row">
        <button id="modalRestartBtn" class="btn btn-primary">再来一局</button>
        <button id="modalCloseBtn" class="btn btn-gray">关闭</button>
      </div>
    </div>
  </div>

  <script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    const scoreText = document.getElementById('scoreText');
    const timerText = document.getElementById('timerText');
    const bestText = document.getElementById('bestText');
    const resetBestBtn = document.getElementById('resetBestBtn');
    const restartBtn = document.getElementById('restartBtn');
    const gameOverModal = document.getElementById('gameOverModal');
    const finalScoreVal = document.getElementById('finalScoreVal');
    const bestScoreVal = document.getElementById('bestScoreVal');
    const modalRestartBtn = document.getElementById('modalRestartBtn');
    const modalCloseBtn = document.getElementById('modalCloseBtn');

    const STORAGE_KEY = 'parkour_best_score';

    function getBestScore() {
      const raw = localStorage.getItem(STORAGE_KEY);
      return raw ? parseInt(raw, 10) : 0;
    }
    function saveBestScore(v) {
      localStorage.setItem(STORAGE_KEY, String(v));
    }
    function clearBestScore() {
      localStorage.removeItem(STORAGE_KEY);
      renderBest();
    }
    function renderBest() {
      const b = getBestScore();
      bestText.innerText = `🏆 最高分：${b}`;
      bestScoreVal.innerText = b;
    }

    function resizeCanvas() {
      canvas.width = innerWidth;
      canvas.height = innerHeight;
    }
    resizeCanvas();
    window.addEventListener('resize', resizeCanvas);

    let score = 0;
    let timeLeft = 60;
    let lane = 1;
    let speed = 6;
    let coins = [];
    let obstacles = [];
    let lastSpawn = 0;
    let gameRunning = true;
    let touchStartX = 0;
    let animTime = 0;

    function spawnCoin() {
      const l = Math.floor(Math.random() * 3);
      coins.push({ lane: l, y: -40, rot: Math.random() * Math.PI * 2, rotSpeed: 0.08 + Math.random() * 0.12 });
    }
    function spawnObstacle() {
      const l = Math.floor(Math.random() * 3);
      obstacles.push({ lane: l, y: -70, rot: 0, rotSpeed: 0.04 });
    }

    function drawRoad() {
      const w = canvas.width;
      const h = canvas.height;
      const cx = w / 2;
      const nearW = w * 0.75;
      const farW = w * 0.25;
      ctx.fillStyle = '#161d2b';
      ctx.fillRect(0, 0, w, h);
      ctx.strokeStyle = '#2a374f';
      ctx.lineWidth = 2;
      for (let i = 0; i < 12; i++) {
        const t = i / 12;
        const ny = h * t;
        const nw = farW + (nearW - farW) * (1 - t);
        ctx.beginPath();
        ctx.moveTo(cx - nw, ny);
        ctx.lineTo(cx + nw, ny);
        ctx.stroke();
      }
      ctx.strokeStyle = '#3f5374';
      for (let p = 0; p < 4; p++) {
        const t = p / 3;
        const baseX = cx - farW + t * farW * 2;
        ctx.beginPath();
        ctx.moveTo(baseX, 0);
        ctx.lineTo(cx - nearW + t * nearW * 2, h);
        ctx.stroke();
      }
    }

    function laneX(laneIdx) {
      const w = canvas.width;
      const h = canvas.height;
      const nearW = w * 0.75;
      const farW = w * 0.25;
      const cx = w / 2;
      return cx - nearW / 2 + (laneIdx + 0.5) * nearW / 3;
    }

    function drawCoin(c, t) {
      const x = laneX(c.lane);
      const y = c.y;
      c.rot += c.rotSpeed;
      const flip = Math.cos(c.rot);
      const scaleX = Math.abs(flip);
      ctx.save();
      ctx.translate(x, y);
      ctx.scale(scaleX, 1);
      const grad = ctx.createRadialGradient(0, 0, 0, 0, 0, 18);
      grad.addColorStop(0, '#fff');
      grad.addColorStop(0.4, '#ffdd66');
      grad.addColorStop(1, '#b87333');
      ctx.fillStyle = grad;
      ctx.beginPath();
      ctx.arc(0, 0, 18, 0, Math.PI * 2);
      ctx.fill();
      ctx.fillStyle = 'rgba(255,255,255,0.35)';
      ctx.beginPath();
      ctx.arc(-5, -5, 6, 0, Math.PI * 2);
      ctx.fill();
      ctx.restore();
    }

    function drawObstacle(o, t) {
      const x = laneX(o.lane);
      const y = o.y;
      o.rot += o.rotSpeed;
      ctx.save();
      ctx.translate(x, y);
      ctx.rotate(o.rot);
      ctx.shadowColor = '#ef4444';
      ctx.shadowBlur = 12;
      ctx.fillStyle = '#dc2626';
      ctx.fillRect(-28, -36, 56, 72);
      ctx.fillStyle = '#fca5a5';
      ctx.fillRect(-18, -22, 36, 14);
      ctx.shadowBlur = 0;
      ctx.restore();
    }

    function drawPlayer() {
      const x = laneX(lane);
      const y = canvas.height - 90;
      ctx.save();
      ctx.shadowColor = '#22c55e';
      ctx.shadowBlur = 14;
      ctx.fillStyle = '#22c55e';
      ctx.beginPath();
      ctx.arc(x, y, 28, 0, Math.PI * 2);
      ctx.fill();
      ctx.fillStyle = '#fff';
      ctx.beginPath();
      ctx.arc(x - 8, y - 8, 7, 0, Math.PI * 2);
      ctx.fill();
      ctx.shadowBlur = 0;
      ctx.restore();
    }

    function checkCollision() {
      const px = laneX(lane);
      const py = canvas.height - 90;
      const hitRadius = 30;
      coins = coins.filter(c => {
        const cx = laneX(c.lane);
        const cy = c.y;
        const dist = Math.hypot(cx - px, cy - py);
        if (dist < hitRadius + 18) {
          score += 10;
          return false;
        }
        return true;
      });
      for (let o of obstacles) {
        const ox = laneX(o.lane);
        const oy = o.y;
        const dist = Math.hypot(ox - px, oy - py);
        if (dist < hitRadius + 24) {
          gameOver();
          return;
        }
      }
    }

    function gameOver() {
      gameRunning = false;
      const b = getBestScore();
      if (score > b) saveBestScore(score);
      finalScoreVal.innerText = score;
      renderBest();
      gameOverModal.classList.remove('hidden');
    }

    function resetGame() {
      score = 0;
      timeLeft = 60;
      lane = 1;
      speed = 6;
      coins = [];
      obstacles = [];
      lastSpawn = 0;
      gameRunning = true;
      gameOverModal.classList.add('hidden');
      renderBest();
    }

    function gameLoop(ts) {
      animTime = ts;
      if (!gameRunning) {
        drawRoad();
        requestAnimationFrame(gameLoop);
        return;
      }
      speed = 6 + (60 - timeLeft) * 0.24;
      const now = Date.now();
      if (now - lastSpawn > 700) {
        lastSpawn = now;
        if (Math.random() < 0.6) spawnCoin();
        if (Math.random() < 0.3) spawnObstacle();
      }
      coins.forEach(c => c.y += speed);
      obstacles.forEach(o => o.y += speed);
      coins = coins.filter(c => c.y < canvas.height + 80);
      obstacles = obstacles.filter(o => o.y < canvas.height + 100);
      checkCollision();
      drawRoad();
      coins.forEach(c => drawCoin(c, ts));
      obstacles.forEach(o => drawObstacle(o, ts));
      drawPlayer();
      scoreText.innerText = `本局：${score}`;
      timerText.innerText = `倒计时：${timeLeft}`;
      renderBest();
      requestAnimationFrame(gameLoop);
    }

    function startTimer() {
      const t = setInterval(() => {
        if (!gameRunning) return;
        timeLeft--;
        if (timeLeft <= 0) {
          clearInterval(t);
          gameOver();
        }
      }, 1000);
    }

    document.addEventListener('keydown', e => {
      if (e.code === 'ArrowLeft' && lane > 0) lane--;
      if (e.code === 'ArrowRight' && lane < 2) lane++;
    });
    canvas.addEventListener('touchstart', e => {
      touchStartX = e.touches[0].clientX;
    });
    canvas.addEventListener('touchend', e => {
      const endX = e.changedTouches[0].clientX;
      const dx = endX - touchStartX;
      if (Math.abs(dx) > 28) {
        if (dx > 0 && lane < 2) lane++;
        else if (dx < 0 && lane > 0) lane--;
      }
    });

    resetBestBtn.onclick = () => {
      if (confirm('确定要清空本地最高分吗？')) clearBestScore();
    };
    restartBtn.onclick = resetGame;
    modalRestartBtn.onclick = resetGame;
    modalCloseBtn.onclick = () => gameOverModal.classList.add('hidden');

    renderBest();
    startTimer();
    requestAnimationFrame(gameLoop);
  </script>
</body>
</html>