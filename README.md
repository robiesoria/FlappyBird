<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Flappy Bird</title>
  <style>
    :root{
      --bg:#70c5ce;
      --ground:#ded895;
      --accent:#ffd166;
      --text:#173f5f;
    }
    html,body{
      height:100%;
      margin:0;
      font-family: system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;
      background: linear-gradient(#70c5ce 0%, #9be7ff 50%);
      color:var(--text);
      -webkit-tap-highlight-color: transparent;
    }
    .wrap{
      min-height:100%;
      display:flex;
      align-items:center;
      justify-content:center;
      padding:28px;
      box-sizing:border-box;
    }
    .card{
      width:100%;
      max-width:480px;
      background: rgba(255,255,255,0.04);
      border-radius:12px;
      box-shadow: 0 10px 30px rgba(12,40,60,0.12);
      padding:16px;
      backdrop-filter: blur(6px);
    }
    header{
      display:flex;
      align-items:center;
      justify-content:space-between;
      gap:12px;
      margin-bottom:12px;
    }
    h1{font-size:18px;margin:0}
    .controls{
      display:flex;
      gap:8px;
      align-items:center;
    }
    button{
      background:var(--accent);
      border:0;
      padding:8px 12px;
      border-radius:8px;
      font-weight:600;
      cursor:pointer;
      box-shadow: 0 4px 8px rgba(0,0,0,0.12);
    }
    button.secondary{
      background:transparent;
      border:1px solid rgba(0,0,0,0.06);
      color:var(--text);
    }
    .canvas-wrap{
      background: linear-gradient(#9be7ff,#70c5ce);
      border-radius:8px;
      padding:8px;
    }
    canvas{
      display:block;
      width:100%;
      border-radius:6px;
      touch-action: manipulation;
      background: linear-gradient(#9be7ff,#70c5ce);
      box-shadow: inset 0 -6px 20px rgba(0,0,0,0.06);
    }
    .footer{
      display:flex;
      justify-content:space-between;
      margin-top:10px;
      font-size:13px;
      color:rgba(23,63,95,0.8);
    }
    .hint{opacity:0.9}
    @media (max-width:420px){
      h1{font-size:16px}
      button{padding:6px 10px}
    }
  </style>
</head>
<body>
  <div class="wrap">
    <div class="card" role="application" aria-label="Flappy Bird game">
      <header>
        <h1>Flappy Bird</h1>
        <div class="controls">
          <button id="startBtn">Start</button>
          <button id="muteBtn" class="secondary">Mute</button>
        </div>
      </header>

      <div class="canvas-wrap">
        <canvas id="gameCanvas" width="480" height="640" aria-hidden="false"></canvas>
      </div>

      <div class="footer">
        <div class="hint">Space / Click / Tap to flap</div>
        <div id="bestScore">Best: 0</div>
      </div>
    </div>
  </div>

  <script>
  /* Flappy Bird clone — single-file
     - Responsive canvas scaling for crisp rendering
     - Touch/mouse/keyboard controls
     - LocalStorage best score
     - Minimal WebAudio beeps (no external assets)
  */

  (function(){
    // --- DOM ---
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d', { alpha: false });
    const startBtn = document.getElementById('startBtn');
    const muteBtn = document.getElementById('muteBtn');
    const bestScoreEl = document.getElementById('bestScore');

    // --- HiDPI scaling ---
    function resizeCanvas() {
      const rect = canvas.getBoundingClientRect();
      const dpr = Math.min(window.devicePixelRatio || 1, 2);
      canvas.width = Math.round(rect.width * dpr);
      canvas.height = Math.round(rect.height * dpr);
      canvas.style.width = rect.width + 'px';
      canvas.style.height = rect.height + 'px';
      ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
      // store scaled logical size
      game.width = rect.width;
      game.height = rect.height;
    }

    window.addEventListener('resize', resizeCanvas);

    // --- Simple audio using Web Audio API ---
    const AudioEngine = (() => {
      const ctx = new (window.AudioContext || window.webkitAudioContext)();
      let _muted = false;
      function beep(freq=440, time=0.06, type='sine', gain=0.05){
        if(_muted) return;
        const o = ctx.createOscillator();
        const g = ctx.createGain();
        o.type = type;
        o.frequency.value = freq;
        g.gain.value = gain;
        o.connect(g); g.connect(ctx.destination);
        o.start();
        g.gain.exponentialRampToValueAtTime(0.0001, ctx.currentTime + time);
        o.stop(ctx.currentTime + time + 0.01);
      }
      return {
        flap(){beep(880,0.04,'triangle',0.04);},
        point(){beep(1200,0.08,'sine',0.06);},
        hit(){beep(120,0.25,'square',0.14);},
        toggleMute(){ _muted = !_muted; return _muted; },
        isMuted(){ return _muted; }
      };
    })();

    // --- Game state ---
    const game = {
      running: false,
      over: false,
      width: 480,
      height: 640,
      gravity: 1100, // pixels/s^2
      flapVelocity: -320, // pixels/s impulse on flap
      birdRadius: 14,
      pipeGap: 150,
      pipeWidth: 56,
      pipeSpeed: 160, // px/s
      spawnInterval: 1.5, // seconds
      pipes: [],
      bird: null,
      lastTime: null,
      spawnTimer: 0,
      score: 0,
      best: parseInt(localStorage.getItem('flappy_best') || '0',10)
    };

    bestScoreEl.textContent = 'Best: ' + game.best;

    // --- Bird object ---
    function resetBird() {
      return {
        x: Math.max(60, game.width * 0.24),
        y: game.height * 0.45,
        vy: 0,
        angle: 0
      };
    }

    // --- Pipes ---
    function spawnPipe() {
      const margin = 36;
      const minY = margin;
      const maxY = game.height - margin - game.pipeGap - 60; // ensure some ground space
      const topY = Math.random() * (maxY - minY) + minY;
      const pipe = {
        x: game.width + game.pipeWidth,
        top: topY,
        passed: false
      };
      game.pipes.push(pipe);
    }

    // --- Controls ---
    function doFlap(){
      if(!game.running){
        startGame();
      }
      if(game.over) {
        // restart quickly
        startGame();
        return;
      }
      game.bird.vy = game.flapVelocity;
      AudioEngine.flap();
    }

    // mouse / touch / keyboard
    canvas.addEventListener('pointerdown', (e) => {
      e.preventDefault();
      doFlap();
    }, {passive:false});
    document.addEventListener('keydown', (e) => {
      if(e.code === 'Space' || e.code === 'ArrowUp') {
        e.preventDefault();
        doFlap();
      }
    });

    // buttons
    startBtn.addEventListener('click', () => {
      if(game.running && !game.over) {
        // toggle pause
        game.running = false;
        startBtn.textContent = 'Start';
      } else {
        startGame();
      }
    });

    muteBtn.addEventListener('click', () => {
      const muted = AudioEngine.toggleMute();
      muteBtn.textContent = muted ? 'Unmute' : 'Mute';
    });

    // --- Physics & collision helpers ---
    function rectsIntersect(rx, ry, rw, rh, cx, cy, cr) {
      // AABB vs circle (approx)
      const nearestX = Math.max(rx, Math.min(cx, rx + rw));
      const nearestY = Math.max(ry, Math.min(cy, ry + rh));
      const dx = cx - nearestX;
      const dy = cy - nearestY;
      return (dx*dx + dy*dy) < (cr*cr);
    }

    // --- Game loop ---
    function startGame(){
      game.pipes = [];
      game.bird = resetBird();
      game.score = 0;
      game.spawnTimer = 0;
      game.over = false;
      game.running = true;
      game.lastTime = performance.now();
      startBtn.textContent = 'Pause';
      resizeCanvas();
      requestAnimationFrame(loop);
    }

    function endGame(){
      game.running = false;
      game.over = true;
      AudioEngine.hit();
      // save best
      if(game.score > game.best){
        game.best = game.score;
        localStorage.setItem('flappy_best', String(game.best));
        bestScoreEl.textContent = 'Best: ' + game.best;
      }
      startBtn.textContent = 'Restart';
    }

    function loop(now){
      if(!game.running){
        draw(); // draw static frame
        return;
      }
      const dt = Math.min((now - game.lastTime) / 1000, 0.033); // clamp delta (s)
      game.lastTime = now;

      // spawn pipes
      game.spawnTimer += dt;
      if(game.spawnTimer >= game.spawnInterval){
        game.spawnTimer -= game.spawnInterval;
        spawnPipe();
      }

      // update bird
      game.bird.vy += game.gravity * dt;
      game.bird.y += game.bird.vy * dt;
      // angle based on velocity
      game.bird.angle = Math.max(-0.6, Math.min(0.9, game.bird.vy / 700));

      // update pipes
      for(let i = game.pipes.length - 1; i >= 0; --i){
        let p = game.pipes[i];
        p.x -= game.pipeSpeed * dt;
        // scoring
        if(!p.passed && p.x + game.pipeWidth < game.bird.x - game.birdRadius){
          p.passed = true;
          game.score += 1;
          AudioEngine.point();
        }
        // remove off-screen
        if(p.x + game.pipeWidth < -10){
          game.pipes.splice(i,1);
        }
      }

      // collision with ground or ceiling
      const groundY = game.height - 24; // visual ground
      if(game.bird.y + game.birdRadius >= groundY){
        game.bird.y = groundY - game.birdRadius;
        endGame();
      }
      if(game.bird.y - game.birdRadius <= 0){
        game.bird.y = game.birdRadius;
        game.bird.vy = 0;
      }

      // collision with pipes
      for(let p of game.pipes){
        // top pipe rect: x..x+pipeWidth, y=0..p.top
        if(rectsIntersect(p.x, 0, game.pipeWidth, p.top, game.bird.x, game.bird.y, game.birdRadius)) {
          endGame();
          break;
        }
        const bottomY = p.top + game.pipeGap;
        // bottom pipe rect: x..x+pipeWidth, bottomY..height-ground
        if(rectsIntersect(p.x, bottomY, game.pipeWidth, game.height - bottomY - 24, game.bird.x, game.bird.y, game.birdRadius)) {
          endGame();
          break;
        }
      }

      // draw
      draw();

      if(game.running) requestAnimationFrame(loop);
    }

    // --- Drawing functions ---
    function draw(){
      const w = game.width;
      const h = game.height;
      // clear
      ctx.clearRect(0,0,w,h);

      // sky gradient already handled by CSS background; but draw far background shapes
      // draw sun/clouds simplified
      drawClouds();

      // draw pipes
      for(let p of game.pipes) drawPipe(p);

      // draw ground strip
      ctx.fillStyle = '#ded895';
      ctx.fillRect(0, h-24, w, 24);
      // ground pattern stripes
      ctx.fillStyle = 'rgba(0,0,0,0.03)';
      for(let gx=0; gx<w; gx+=18){
        ctx.fillRect(gx, h-24 + 12, 10, 4);
      }

      // draw bird
      drawBird(game.bird.x, game.bird.y, game.birdRadius, game.bird.angle);

      // HUD - score
      ctx.fillStyle = 'rgba(23,63,95,0.95)';
      ctx.font = '700 34px system-ui, -apple-system, "Segoe UI", Roboto, Arial';
      ctx.textAlign = 'center';
      ctx.fillText(game.score, w/2, 64);

      // overlay instructions or game over
      if(!game.running && !game.over) {
        ctx.fillStyle = 'rgba(23,63,95,0.9)';
        ctx.font = '600 16px system-ui, -apple-system, "Segoe UI", Roboto, Arial';
        ctx.textAlign = 'center';
        ctx.fillText('Click / Tap / Space to flap — press Start to play', w/2, h/2 - 10);
      }
      if(game.over) {
        ctx.fillStyle = 'rgba(0,0,0,0.5)';
        ctx.fillRect(w/2 - 150, h/2 - 80, 300, 140);
        ctx.fillStyle = 'white';
        ctx.textAlign = 'center';
        ctx.font = '700 20px system-ui, -apple-system, "Segoe UI", Roboto, Arial';
        ctx.fillText('Game Over', w/2, h/2 - 30);
        ctx.font = '600 16px system-ui, -apple-system, "Segoe UI", Roboto, Arial';
        ctx.fillText('Score: ' + game.score, w/2, h/2 - 2);
        ctx.fillText('Best: ' + game.best, w/2, h/2 + 20);
        ctx.font = '500 13px system-ui, -apple-system, "Segoe UI", Roboto, Arial';
        ctx.fillText('Click / Tap to restart', w/2, h/2 + 44);
      }
    }

    function drawClouds(){
      const w = game.width;
      const h = game.height;
      // simple moving clouds
      ctx.save();
      ctx.globalAlpha = 0.18;
      ctx.fillStyle = '#ffffff';
      const t = (performance.now() * 0.00005) % 1;
      const cw = 80;
      for(let i=0;i<5;i++){
        const x = ((i*230) + t*w*0.5) % (w + cw) - cw;
        const y = 40 + (i%2)*20;
        roundedCloud(x,y,cw,28);
      }
      ctx.restore();
    }

    function roundedCloud(x,y,wc,hc){
      ctx.beginPath();
      ctx.ellipse(x+wc*0.2,y,hc*0.6,hc*0.6,0,0,Math.PI*2);
      ctx.ellipse(x+wc*0.5,y-hc*0.2,hc*0.7,hc*0.7,0,0,Math.PI*2);
      ctx.ellipse(x+wc*0.8,y,hc*0.5,hc*0.5,0,0,Math.PI*2);
      ctx.fill();
    }

    function drawPipe(p){
      const w = p.x;
      const top = p.top;
      const pipeW = game.pipeWidth;
      const gap = game.pipeGap;
      // pipe color gradient
      const g = ctx.createLinearGradient(p.x, 0, p.x + pipeW, 0);
      g.addColorStop(0, '#4aa96c');
      g.addColorStop(1, '#3a8a56');
      // top
      ctx.fillStyle = g;
      roundRect(ctx, p.x, 0, pipeW, top, 6, true, false);
      // bottom
      roundRect(ctx, p.x, top + gap, pipeW, game.height - (top + gap) - 24, 6, true, false);

      // cutout highlight
      ctx.fillStyle = 'rgba(255,255,255,0.06)';
      ctx.fillRect(p.x + 4, 6, 8, Math.max(2, top - 10));
    }

    function drawBird(x,y,r,angle){
      ctx.save();
      ctx.translate(x,y);
      ctx.rotate(angle);
      // body
      ctx.beginPath();
      ctx.fillStyle = '#ffdd57';
      ctx.ellipse(0,0,r*1.05,r*0.85,0,0,Math.PI*2);
      ctx.fill();
      // wing
      ctx.fillStyle = '#ffd166';
      ctx.beginPath();
      ctx.ellipse(-4,0,r*0.6,r*0.3,Math.PI/6,0,Math.PI*2);
      ctx.fill();
      // eye
      ctx.fillStyle = '#000';
      ctx.beginPath();
      ctx.arc(6,-4,3,0,Math.PI*2);
      ctx.fill();
      // beak
      ctx.fillStyle = '#ff7b00';
      ctx.beginPath();
      ctx.moveTo(r+2, 0);
      ctx.lineTo(r+12, -4);
      ctx.lineTo(r+12, 6);
      ctx.closePath();
      ctx.fill();
      ctx.restore();
    }

    // rounded rect helper
    function roundRect(ctx, x, y, w, h, r, fill, stroke){
      if(typeof r === 'undefined') r = 5;
      ctx.beginPath();
      ctx.moveTo(x + r, y);
      ctx.arcTo(x + w, y, x + w, y + h, r);
      ctx.arcTo(x + w, y + h, x, y + h, r);
      ctx.arcTo(x, y + h, x, y, r);
      ctx.arcTo(x, y, x + w, y, r);
      ctx.closePath();
      if(fill) ctx.fill();
      if(stroke) ctx.stroke();
    }

    // --- Initialize ---
    // set initial size and draw placeholder
    resizeCanvas();
    draw();

    // expose for debugging (optional)
    window._flappy = { game, startGame };
  })();
  </script>
</body>
</html>
