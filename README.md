<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Meerkat Moon Jump</title>
    <script src="https://telegram.org/js/telegram-web-app.js"></script>
    <style>
        body {
            margin: 0;
            padding: 0;
            background-color: #0f172a;
            overflow: hidden;
            touch-action: none;
            font-family: 'Courier New', Courier, monospace;
            height: 100dvh;
            width: 100vw;
            display: flex;
            justify-content: center;
            align-items: center;
        }

        #gameCanvas {
            display: block;
            width: 100%;
            height: 100%;
        }

        #ui-layer {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            pointer-events: none;
            display: flex;
            flex-direction: column;
            justify-content: space-between;
        }

        #scoreBoard {
            text-align: center;
            margin-top: 40px;
            color: #fff;
            font-size: 20px;
            font-weight: bold;
            text-shadow: 0 2px 4px rgba(0,0,0,0.8);
            background: rgba(15, 23, 42, 0.6);
            padding: 10px;
            border-radius: 20px;
            align-self: center;
            backdrop-filter: blur(4px);
            border: 1px solid #22c55e;
            box-shadow: 0 0 15px rgba(34, 197, 94, 0.2);
        }

        .screen-modal {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: rgba(15, 23, 42, 0.95);
            padding: 30px;
            border-radius: 16px;
            border: 2px solid #22c55e;
            text-align: center;
            pointer-events: auto;
            box-shadow: 0 0 50px rgba(34, 197, 94, 0.3);
            width: 80%;
            max-width: 350px;
        }

        h1 { margin: 0 0 10px 0; color: #fff; font-size: 24px; }
        p { color: #cbd5e1; font-size: 14px; margin-bottom: 20px; }
        
        button {
            background: linear-gradient(135deg, #22c55e, #16a34a);
            color: white;
            border: none;
            padding: 15px 30px;
            font-size: 18px;
            font-weight: bold;
            border-radius: 12px;
            cursor: pointer;
            width: 100%;
            box-shadow: 0 4px 0 #14532d;
            transition: transform 0.1s;
        }

        button:active {
            transform: translateY(4px);
            box-shadow: none;
        }

        .hidden { display: none !important; }
        
        .rekt { animation: shake 0.5s; }
        @keyframes shake {
            0% { transform: translate(-50%, -50%) translate(1px, 1px) rotate(0deg); }
            10% { transform: translate(-50%, -50%) translate(-1px, -2px) rotate(-1deg); }
            30% { transform: translate(-50%, -50%) translate(3px, 2px) rotate(0deg); }
            50% { transform: translate(-50%, -50%) translate(-1px, 2px) rotate(-1deg); }
            70% { transform: translate(-50%, -50%) translate(3px, 1px) rotate(-1deg); }
            90% { transform: translate(-50%, -50%) translate(1px, 2px) rotate(0deg); }
            100% { transform: translate(-50%, -50%) translate(1px, -2px) rotate(-1deg); }
        }
    </style>
</head>
<body>

    <canvas id="gameCanvas"></canvas>

    <div id="ui-layer">
        <div id="scoreBoard">Market Cap: $0</div>
        
        <div id="startScreen" class="screen-modal">
            <h1>Meerkat Moon 🚀</h1>
            <p>Tap Left or Right to steer.</p>
            <p>Bounce on Green Candles.</p>
            <button onclick="startGame()">LAUNCH</button>
        </div>

        <div id="gameOverScreen" class="screen-modal hidden">
            <h1 style="color: #ef4444;">REKT! 📉</h1>
            <p>You missed the liquidity.</p>
            <p>Final Cap: <span id="finalScore" style="color:#22c55e; font-weight:bold;">$0</span></p>
            <button onclick="startGame()">BUY THE DIP</button>
        </div>
    </div>

<script>
    const tg = window.Telegram.WebApp;
    tg.expand();
    tg.ready();

    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    const scoreEl = document.getElementById('scoreBoard');
    const startScrn = document.getElementById('startScreen');
    const endScrn = document.getElementById('gameOverScreen');
    const finalScoreEl = document.getElementById('finalScore');

    let width, height;
    function resize() {
        width = window.innerWidth;
        height = window.innerHeight;
        canvas.width = width;
        canvas.height = height;
    }
    window.addEventListener('resize', resize);
    resize();

    // --- GAME VARIABLES ---
    let gameRunning = false;
    let score = 0;
    let platforms = [];
    let keys = { left: false, right: false };

    // --- REBALANCED PHYSICS (Slower & Easier) ---
    const CONFIG = {
        gravity: 0.15,      // Reduced from 0.25 (Floatier feel)
        speed: 3.5,         // Reduced from 6.0 (More precise steering)
        jumpForce: -7.5,    // Reduced from -9 (Balanced with new gravity)
        platformWidth: 70,
        platformHeight: 20,
        platformGap: 100    // Reduced gap slightly for easier climbing
    };

    let meerkat = {
        x: 0,
        y: 0,
        w: 40,
        h: 50,
        vx: 0,
        vy: 0,
        faceDir: 1 
    };

    // --- INPUTS ---
    canvas.addEventListener('touchstart', (e) => {
        e.preventDefault();
        handleInput(e.touches[0].clientX);
    }, {passive: false});

    canvas.addEventListener('touchmove', (e) => {
        e.preventDefault();
        handleInput(e.touches[0].clientX);
    }, {passive: false});

    canvas.addEventListener('touchend', () => {
        keys.left = false;
        keys.right = false;
    });

    function handleInput(x) {
        if (x < width / 2) {
            keys.left = true;
            keys.right = false;
        } else {
            keys.right = true;
            keys.left = false;
        }
    }

    // Keyboard for testing
    window.addEventListener('keydown', (e) => {
        if (e.code === 'ArrowLeft') keys.left = true;
        if (e.code === 'ArrowRight') keys.right = true;
    });
    window.addEventListener('keyup', (e) => {
        if (e.code === 'ArrowLeft') keys.left = false;
        if (e.code === 'ArrowRight') keys.right = false;
    });

    // --- GENERATION LOGIC ---
    function initPlatforms() {
        platforms = [];
        let count = Math.ceil(height / CONFIG.platformGap) + 1;
        
        // 1. SAFETY PLATFORM: Always spawn one directly under the start position
        platforms.push({
            x: width / 2 - 35, // Centered
            y: height - 100,
            w: 70,
            h: 20,
            isMoving: false
        });

        // 2. Generate the rest
        for (let i = 1; i < count; i++) {
            platforms.push({
                x: Math.random() * (width - CONFIG.platformWidth),
                y: height - (i * CONFIG.platformGap) - 100,
                w: CONFIG.platformWidth,
                h: CONFIG.platformHeight,
                isMoving: false // Disable moving platforms initially
            });
        }
    }

    // --- DRAWING ---
    function drawMeerkat(x, y, w, h, dir) {
        ctx.save();
        ctx.translate(x + w/2, y + h/2);
        if(dir === -1) ctx.scale(-1, 1);

        // Body
        ctx.fillStyle = '#D97706'; 
        ctx.beginPath();
        ctx.ellipse(0, 10, w/2.2, h/2.5, 0, 0, Math.PI * 2);
        ctx.fill();

        // Belly
        ctx.fillStyle = '#FCD34D'; 
        ctx.beginPath();
        ctx.ellipse(0, 12, w/4, h/3, 0, 0, Math.PI * 2);
        ctx.fill();

        // Head
        ctx.fillStyle = '#D97706';
        ctx.beginPath();
        ctx.arc(0, -15, w/2, 0, Math.PI*2);
        ctx.fill();

        // Ears
        ctx.fillStyle = '#78350F'; 
        ctx.beginPath();
        ctx.arc(-12, -22, 5, 0, Math.PI*2); 
        ctx.arc(12, -22, 5, 0, Math.PI*2);  
        ctx.fill();

        // Eyes
        ctx.fillStyle = '#78350F';
        ctx.beginPath();
        ctx.ellipse(-6, -15, 6, 4, 0, 0, Math.PI*2);
        ctx.ellipse(6, -15, 6, 4, 0, 0, Math.PI*2);
        ctx.fill();

        ctx.fillStyle = '#000';
        ctx.beginPath();
        ctx.arc(-6, -15, 2, 0, Math.PI*2);
        ctx.arc(6, -15, 2, 0, Math.PI*2);
        ctx.fill();
        
        // Snout
        ctx.fillStyle = '#000';
        ctx.beginPath();
        ctx.moveTo(-2, -8);
        ctx.lineTo(2, -8);
        ctx.lineTo(0, -5);
        ctx.fill();

        ctx.restore();
    }

    function drawCandle(p) {
        ctx.fillStyle = '#14532d';
        ctx.fillRect(p.x + p.w/2 - 1, p.y - 6, 2, 6); // Wick
        ctx.fillStyle = '#22c55e';
        ctx.fillRect(p.x, p.y, p.w, p.h); // Body
        ctx.fillStyle = '#86efac';
        ctx.fillRect(p.x + 2, p.y + 2, 4, p.h - 4); // Highlight
        ctx.fillStyle = '#15803d';
        ctx.fillRect(p.x, p.y + p.h, p.w, 4); // Shadow
    }

    // --- UPDATE LOOP ---
    function update() {
        if (!gameRunning) return;

        // MOVEMENT
        if (keys.left) {
            meerkat.x -= CONFIG.speed;
            meerkat.faceDir = -1;
        }
        if (keys.right) {
            meerkat.x += CONFIG.speed;
            meerkat.faceDir = 1;
        }

        // Screen Wrap
        if (meerkat.x + meerkat.w < 0) meerkat.x = width;
        if (meerkat.x > width) meerkat.x = -meerkat.w;

        // PHYSICS
        meerkat.vy += CONFIG.gravity;
        meerkat.y += meerkat.vy;

        // COLLISIONS (Forgiving Hitbox)
        if (meerkat.vy > 0) {
            platforms.forEach(p => {
                // Hitbox padding: Player is slightly wider than visual to help land jumps
                if (
                    meerkat.x + meerkat.w * 0.5 > p.x && 
                    meerkat.x + meerkat.w * 0.5 < p.x + p.w &&
                    meerkat.y + meerkat.h > p.y &&
                    meerkat.y + meerkat.h < p.y + p.h + 15 // Increased detection range
                ) {
                    meerkat.vy = CONFIG.jumpForce;
                    if(tg.HapticFeedback) tg.HapticFeedback.impactOccurred('light');
                }
            });
        }

        // CAMERA SCROLL
        if (meerkat.y < height / 2.5) {
            let shift = (height / 2.5) - meerkat.y;
            meerkat.y = height / 2.5;
            score += Math.floor(shift);
            scoreEl.innerText = "Market Cap: $" + score.toLocaleString();

            platforms.forEach(p => {
                p.y += shift;
                if(p.isMoving) p.x += Math.sin(Date.now() / 800) * 1.5; // Slower moving platforms

                if (p.y > height) {
                    p.y = -20;
                    p.x = Math.random() * (width - CONFIG.platformWidth);
                    // Only introduce moving platforms after score > 3000
                    p.isMoving = score > 3000 && Math.random() > 0.8; 
                }
            });
        }

        // GAME OVER
        if (meerkat.y > height) {
            gameOver();
        }

        draw();
        requestAnimationFrame(update);
    }

    function draw() {
        let grad = ctx.createLinearGradient(0, 0, 0, height);
        grad.addColorStop(0, '#020617');
        grad.addColorStop(1, '#1e293b');
        ctx.fillStyle = grad;
        ctx.fillRect(0, 0, width, height);

        platforms.forEach(p => drawCandle(p));
        drawMeerkat(meerkat.x, meerkat.y, meerkat.w, meerkat.h, meerkat.faceDir);
    }

    function startGame() {
        startScrn.classList.add('hidden');
        endScrn.classList.add('hidden');
        endScrn.classList.remove('rekt');
        
        score = 0;
        scoreEl.innerText = "Market Cap: $0";
        
        resize(); // Ensure size is correct on start
        initPlatforms();
        
        // Spawn player above the safe platform
        meerkat.x = width / 2 - 20;
        meerkat.y = height - 250;
        meerkat.vy = -5;
        
        gameRunning = true;
        update();
    }

    function gameOver() {
        gameRunning = false;
        if(tg.HapticFeedback) tg.HapticFeedback.notificationOccurred('error');
        finalScoreEl.innerText = "$" + score.toLocaleString();
        endScrn.classList.remove('hidden');
        endScrn.classList.add('rekt');
    }
</script>
</body>
</html>
