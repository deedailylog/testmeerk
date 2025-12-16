<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Meerkat Jump</title>
    <script src="https://telegram.org/js/telegram-web-app.js"></script>
    <style>
        body {
            margin: 0;
            padding: 0;
            background-color: #0f172a; /* Dark Blue Background */
            overflow: hidden; /* NO SCROLLBARS */
            touch-action: none; /* Stop zooming */
            font-family: 'Courier New', Courier, monospace;
        }

        /* UI Overlay (Title & Score) */
        #ui-layer {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            padding: 10px 0;
            text-align: center;
            pointer-events: none; /* Let clicks pass through */
            z-index: 10;
        }

        h1 {
            color: #22c55e; /* Green */
            margin: 0;
            font-size: 20px;
            text-shadow: 1px 1px 0 #000;
        }

        #score {
            color: white;
            font-size: 24px;
            font-weight: bold;
            margin-top: 5px;
        }

        /* Canvas fills the screen */
        canvas {
            display: block;
            width: 100%;
            height: 100%;
        }

        /* Start / Game Over Screens */
        #menu {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: rgba(0, 0, 0, 0.85);
            padding: 30px;
            border: 2px solid #22c55e;
            border-radius: 15px;
            text-align: center;
            color: white;
            z-index: 20;
            min-width: 200px;
        }

        button {
            background: #22c55e;
            color: black;
            border: none;
            padding: 15px 30px;
            font-size: 20px;
            font-weight: bold;
            cursor: pointer;
            border-radius: 8px;
            margin-top: 15px;
        }
        
        button:active { transform: scale(0.95); }
        .hidden { display: none !important; }
    </style>
</head>
<body>

    <div id="ui-layer">
        <h1>Meerkat Moon Jump</h1>
        <div id="score">CAP: $0</div>
    </div>

    <canvas id="gameCanvas"></canvas>

    <div id="menu">
        <h2 id="menuTitle" style="margin-top:0;">🚀 READY?</h2>
        <p id="menuText">Tap Left / Right to Move</p>
        <button onclick="startGame()">JUMP!</button>
    </div>

<script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    const menu = document.getElementById('menu');
    const scoreEl = document.getElementById('score');

    // 1. Setup Full Screen (Responsive)
    function resize() {
        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;
    }
    window.addEventListener('resize', resize);
    resize();

    // 2. Game Variables
    let gameRunning = false;
    let score = 0;
    
    // Physics Tuning (Make it Easier)
    const GRAVITY = 0.25;
    const JUMP_FORCE = -9.5; // Stronger jump
    const MOVEMENT_SPEED = 5;
    const PLATFORM_GAP = 85; // Closer platforms (was 100+)

    let meerkat = { x: 0, y: 0, w: 30, h: 40, vx: 0, vy: 0 };
    let platforms = [];
    let keys = {};

    // 3. Inputs (Touch + Keyboard)
    window.addEventListener('keydown', e => keys[e.code] = true);
    window.addEventListener('keyup', e => keys[e.code] = false);
    
    // Simple Touch Logic (Left/Right side of screen)
    window.addEventListener('touchstart', e => {
        e.preventDefault();
        const touchX = e.touches[0].clientX;
        if (touchX < window.innerWidth / 2) keys['ArrowLeft'] = true;
        else keys['ArrowRight'] = true;
    }, {passive: false});

    window.addEventListener('touchend', () => {
        keys['ArrowLeft'] = false;
        keys['ArrowRight'] = false;
    });

    // 4. Game Logic
    function initGame() {
        score = 0;
        meerkat.x = canvas.width / 2 - 15;
        meerkat.y = canvas.height - 150;
        meerkat.vy = 0;
        
        platforms = [];
        // Generate platforms starting from bottom
        for (let y = canvas.height; y > -100; y -= PLATFORM_GAP) {
            platforms.push({
                x: Math.random() * (canvas.width - 70),
                y: y,
                w: 70, // Wider platforms (Easier)
                h: 15
            });
        }
        
        gameRunning = true;
        menu.classList.add('hidden');
        loop();
    }

    function loop() {
        if (!gameRunning) return;

        // Move Left/Right
        if (keys['ArrowLeft']) meerkat.x -= MOVEMENT_SPEED;
        if (keys['ArrowRight']) meerkat.x += MOVEMENT_SPEED;

        // Screen Wrap (Teleport from left to right)
        if (meerkat.x < -meerkat.w) meerkat.x = canvas.width;
        if (meerkat.x > canvas.width) meerkat.x = -meerkat.w;

        // Apply Gravity
        meerkat.vy += GRAVITY;
        meerkat.y += meerkat.vy;

        // Collision Check (Jump on platforms)
        if (meerkat.vy > 0) { // Only check when falling down
            platforms.forEach(p => {
                if (meerkat.x + meerkat.w > p.x &&
                    meerkat.x < p.x + p.w &&
                    meerkat.y + meerkat.h > p.y &&
                    meerkat.y + meerkat.h < p.y + p.h + 10) {
                    
                    meerkat.vy = JUMP_FORCE; // BOING!
                }
            });
        }

        // Camera Scroll (Infinite World)
        if (meerkat.y < canvas.height / 2) {
            let diff = (canvas.height / 2) - meerkat.y;
            meerkat.y = canvas.height / 2;
            score += Math.floor(diff);
            
            platforms.forEach(p => {
                p.y += diff;
                if (p.y > canvas.height) {
                    // Recycle platform to top
                    p.y = -20;
                    p.x = Math.random() * (canvas.width - 70);
                }
            });
        }

        // Game Over (Fall off screen)
        if (meerkat.y > canvas.height) {
            gameOver();
            return;
        }

        draw();
        requestAnimationFrame(loop);
    }

    function draw() {
        ctx.clearRect(0, 0, canvas.width, canvas.height);
        
        // Update Score Text
        scoreEl.innerText = "CAP: $" + score.toLocaleString();

        // Draw Platforms (Green Candles)
        ctx.fillStyle = '#22c55e';
        platforms.forEach(p => {
            ctx.fillRect(p.x, p.y, p.w, p.h);
            // Wick
            ctx.fillStyle = '#14532d';
            ctx.fillRect(p.x + p.w/2 - 2, p.y + p.h, 4, 20); 
            ctx.fillStyle = '#22c55e';
        });

        // Draw Meerkat (Gold Box with Eyes)
        ctx.fillStyle = '#eab308';
        ctx.fillRect(meerkat.x, meerkat.y, meerkat.w, meerkat.h);
        
        // Eyes
        ctx.fillStyle = 'black';
        ctx.fillRect(meerkat.x + 5, meerkat.y + 10, 5, 5);
        ctx.fillRect(meerkat.x + 20, meerkat.y + 10, 5, 5);
    }

    function gameOver() {
        gameRunning = false;
        document.getElementById('menuTitle').innerText = "💀 RUG PULL!";
        document.getElementById('menuTitle').style.color = "#ef4444";
        document.getElementById('menuText').innerText = "Final Cap: $" + score;
        menu.classList.remove('hidden');
    }

    window.startGame = initGame;
</script>
</body>
</html>
