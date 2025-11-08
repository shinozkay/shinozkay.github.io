<!doctype html>
<html lang="ja">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Tetris ComicSans — Fixed</title>
  <style>
    /* Page styling */
    body { margin:0; background:#fff; font-family:'Comic Sans MS', 'Apple Color Emoji', 'Segoe UI Emoji', cursive, sans-serif; }
    h1 { text-align:center; font-size:48px; margin:12px 0; color:#333; font-family:'Comic Sans MS', cursive, sans-serif; }
    .wrap{display:flex;justify-content:center;gap:20px;align-items:flex-start;padding:12px}

    /* Canvas and background grid (fixed shorthand issue) */
    #play { border:1px solid #ccc; background: #fff; 
             background-image: linear-gradient(0deg, transparent 23px, #ccc 24px),
                               linear-gradient(90deg, transparent 23px, #ccc 24px);
             background-size: 24px 24px; display:block; }

    .side{font-family:'Comic Sans MS', cursive, sans-serif;}

    /* Game over overlay */
    #gameOver { position:absolute; left:50%; top:50%; transform:translate(-50%,-50%);
               font-family:'Comic Sans MS', cursive, sans-serif; font-size:48px; color:#333; display:none; pointer-events:none; }

    /* Buttons */
    button{font-family:'Comic Sans MS',cursive, sans-serif; padding:8px 12px; margin:6px 0;}

    /* Responsive */
    @media (max-width:800px){ .wrap{flex-direction:column;align-items:center} }
  </style>
</head>
<body>
  <h1>Tetris</h1>
  <div class="wrap">
    <div style="position:relative">
      <canvas id="play" width="240" height="480"></canvas>
      <div id="gameOver">GAME OVER</div>
    </div>
    <div class="side">
      <div>Score: <span id="score">0</span></div>
      <div>Level: <span id="level">1</span></div>
      <button id="startBtn">Start / Pause</button>
      <button id="resetBtn">Reset</button>
      <button id="dropBtn">Hard Drop</button>
    </div>
  </div>

  <script>
    /* --- Tetris with: auto-drop, hard-drop, rotation, star effect, Comic Sans titles. --- */
    const COLS = 10, ROWS = 20, BLOCK = 24;
    const cvs = document.getElementById('play');
    const ctx = cvs.getContext('2d');
    ctx.imageSmoothingEnabled = false;

    const scoreEl = document.getElementById('score');
    const levelEl = document.getElementById('level');
    const startBtn = document.getElementById('startBtn');
    const resetBtn = document.getElementById('resetBtn');
    const dropBtn = document.getElementById('dropBtn');
    const gameOverDiv = document.getElementById('gameOver');

    // Pastel colors: index 1..7
    const COLORS = ['#000000','#FFF59D','#81D4FA','#90CAF9','#AED581','#A5D6A7','#CE93D8','#F48FB1'];

    const PIECES = {
      I: [[0,0,0,0],[1,1,1,1],[0,0,0,0],[0,0,0,0]],
      J: [[2,0,0],[2,2,2],[0,0,0]],
      L: [[0,0,3],[3,3,3],[0,0,0]],
      O: [[4,4],[4,4]],
      S: [[0,5,5],[5,5,0],[0,0,0]],
      T: [[0,6,0],[6,6,6],[0,0,0]],
      Z: [[7,7,0],[0,7,7],[0,0,0]]
    };
    const PIECE_KEYS = Object.keys(PIECES);

    // Game state
    let grid = null;
    let cur = null;
    let bag = [];
    let score = 0, level = 1, lines = 0;
    let dropInterval = 800; // ms
    let dropCounter = 0, lastTime = 0;
    let playing = false;
    let stars = [];

    /* DPR fix so canvas looks crisp on high-DPR screens */
    function fixDPR(){
      const dpr = window.devicePixelRatio || 1;
      const cssW = COLS*BLOCK, cssH = ROWS*BLOCK;
      cvs.style.width = cssW + 'px';
      cvs.style.height = cssH + 'px';
      cvs.width = cssW * dpr; cvs.height = cssH * dpr; ctx.setTransform(dpr,0,0,dpr,0,0);
    }
    fixDPR();
    window.addEventListener('resize', fixDPR);

    function makeGrid(){
      const g = [];
      for(let r=0;r<ROWS;r++) g[r] = Array(COLS).fill(0);
      return g;
    }

    // Draw a single cell (filled) - empty cells handled by CSS background grid
    function drawCell(x,y,val){
      if(!val) return;
      ctx.fillStyle = COLORS[val];
      ctx.fillRect(x*BLOCK+1, y*BLOCK+1, BLOCK-2, BLOCK-2);
      // subtle inner stroke for pastel tiles
      ctx.strokeStyle = 'rgba(0,0,0,0.06)';
      ctx.strokeRect(x*BLOCK+1, y*BLOCK+1, BLOCK-2, BLOCK-2);
    }

    function draw(){
      ctx.clearRect(0,0,cvs.width,cvs.height);
      // draw grid tiles
      for(let r=0;r<ROWS;r++){
        for(let c=0;c<COLS;c++){
          drawCell(c,r, grid[r][c]);
        }
      }
      // draw current piece
      if(cur){
        for(let r=0;r<cur.shape.length;r++){
          for(let c=0;c<cur.shape[r].length;c++){
            if(cur.shape[r][c]) drawCell(cur.x + c, cur.y + r, cur.shape[r][c]);
          }
        }
      }
      drawStars();
    }

    // Rotation (clockwise) with simple wall kick attempts
    function rotateShape(shape){
      const N = shape.length;
      const out = Array.from({length:N}, ()=> Array(N).fill(0));
      for(let r=0;r<N;r++) for(let c=0;c<N;c++) out[c][N-1-r] = shape[r][c];
      // trim empty rows/cols (keeps compact shapes)
      while(out.length > 0 && out[0].every(v=>v===0)) out.shift();
      while(out.length > 0 && out[out.length-1].every(v=>v===0)) out.pop();
      return out;
    }

    function collide(shape, x, y){
      for(let r=0;r<shape.length;r++){
        for(let c=0;c<shape[r].length;c++){
          if(shape[r][c]){
            const nx = x + c, ny = y + r;
            if(nx < 0 || nx >= COLS || ny >= ROWS) return true;
            if(ny >= 0 && grid[ny][nx]) return true;
          }
        }
      }
      return false;
    }

    function place(){
      for(let r=0;r<cur.shape.length;r++){
        for(let c=0;c<cur.shape[r].length;c++){
          if(cur.shape[r][c]){
            const nx = cur.x + c, ny = cur.y + r;
            if(ny < 0){
              // piece placed above visible area -> game over
              return gameOver();
            }
            grid[ny][nx] = cur.shape[r][c];
          }
        }
      }
      clearLines();
      spawnPiece();
    }

    function clearLines(){
      let cleared = 0;
      for(let r = ROWS - 1; r >= 0; r--){
        if(grid[r].every(v => v !== 0)){
          spawnStars(r);
          grid.splice(r, 1);
          grid.unshift(Array(COLS).fill(0));
          cleared++; r++; // recheck same index after shifting
        }
      }
      if(cleared){
        lines += cleared;
        score += [0,40,100,300,1200][cleared] * level;
        level = Math.floor(lines / 10) + 1;
        dropInterval = Math.max(80, 800 - (level - 1) * 50);
        scoreEl.textContent = score;
        levelEl.textContent = level;
      }
    }

    // 7-bag implementation
    function refillBag(){
      let bagKeys = PIECE_KEYS.slice();
      for(let i = bagKeys.length - 1; i > 0; i--){
        const j = Math.floor(Math.random() * (i + 1));
        [bagKeys[i], bagKeys[j]] = [bagKeys[j], bagKeys[i]];
      }
      for(const k of bagKeys){
        const shape = JSON.parse(JSON.stringify(PIECES[k]));
        const id = PIECE_KEYS.indexOf(k) + 1;
        for(let r=0;r<shape.length;r++) for(let c=0;c<shape[r].length;c++) if(shape[r][c]) shape[r][c] = id;
        bag.push({shape, id});
      }
    }

    function spawnPiece(){
      if(bag.length === 0) refillBag();
      cur = bag.pop();
      cur.x = Math.floor((COLS - cur.shape[0].length) / 2);
      cur.y = -getTopOffset(cur.shape);
      // immediate collision -> game over
      if(collide(cur.shape, cur.x, cur.y)) gameOver();
    }

    function getTopOffset(shape){
      for(let r=0;r<shape.length;r++) if(shape[r].some(v=>v)) return r; return 0;
    }

    function hardDrop(){
      if(!cur) return;
      while(!collide(cur.shape, cur.x, cur.y + 1)) cur.y++;
      place();
      draw();
    }

    function rotateCurrent(){
      if(!cur) return;
      const r = rotateShape(cur.shape);
      // simple kicks
      const kicks = [0, -1, 1, -2, 2];
      for(const k of kicks){
        if(!collide(r, cur.x + k, cur.y)){
          cur.shape = r; cur.x += k; return;
        }
      }
    }

    /* --- Star particle effect for cleared lines --- */
    function spawnStars(row){
      // spawn a bunch of small star particles from random x positions across the cleared row
      for(let i=0;i<18;i++){
        const cx = (Math.random() * COLS) * BLOCK;
        const cy = row * BLOCK + (Math.random() * BLOCK);
        stars.push({ x: cx, y: cy, dx: (Math.random()-0.5) * 2.5, dy: -Math.random()*3 - 1, life: 60, size: 2 + Math.random()*2 });
      }
    }
    function updateStars(){
      for(const s of stars){ s.x += s.dx; s.y += s.dy; s.dy += 0.08; s.life--; }
      stars = stars.filter(s => s.life > 0);
    }
    function drawStars(){
      for(const s of stars){
        ctx.beginPath(); ctx.fillStyle = 'rgba(255,215,0,' + Math.max(0.08, s.life/60) + ')';
        ctx.arc(s.x, s.y, s.size, 0, Math.PI*2); ctx.fill();
      }
    }

    /* --- Game loop (requestAnimationFrame) --- */
    function update(time = 0){
      if(!playing) return;
      if(!lastTime) lastTime = time;
      const delta = time - lastTime;
      lastTime = time;
      dropCounter += delta;
      if(dropCounter > dropInterval){
        if(cur && !collide(cur.shape, cur.x, cur.y + 1)){
          cur.y += 1;
        } else if(cur){
          place();
        }
        dropCounter = 0;
      }
      updateStars();
      draw();
      if(playing) requestAnimationFrame(update);
    }

    function start(){
      if(playing){ playing = false; startBtn.textContent = 'Resume'; }
      else{ if(!grid) reset(); playing = true; lastTime = 0; dropCounter = 0; requestAnimationFrame(update); startBtn.textContent = 'Pause'; }
    }

    function reset(){
      playing = false; gameOverDiv.style.display = 'none';
      grid = makeGrid(); bag = []; score = 0; level = 1; lines = 0; dropInterval = 800; stars = [];
      scoreEl.textContent = score; levelEl.textContent = level;
      spawnPiece(); draw(); startBtn.textContent = 'Start / Pause';
    }

    function gameOver(){
      playing = false;
      gameOverDiv.style.display = 'block';
    }

    /* --- Input handlers --- */
    window.addEventListener('keydown', (e) => {
      if(!cur) return;
      if(e.key === 'ArrowLeft'){ if(!collide(cur.shape, cur.x - 1, cur.y)) cur.x--; }
      else if(e.key === 'ArrowRight'){ if(!collide(cur.shape, cur.x + 1, cur.y)) cur.x++; }
      else if(e.key === 'ArrowUp'){ rotateCurrent(); }
      else if(e.key === 'ArrowDown'){ hardDrop(); }
      else if(e.key === ' '){ e.preventDefault(); hardDrop(); }
      draw();
    });

    // Buttons
    startBtn.addEventListener('click', start);
    resetBtn.addEventListener('click', () => { if(confirm('Reset game?')) reset(); });
    dropBtn.addEventListener('click', () => { hardDrop(); });

    // Basic touch buttons (if you add on-screen buttons later, connect them here)

    // initialize
    reset();

  </script>
</body>
</html>
