<!DOCTYPE html>
<html lang="en">
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
            overflow: hidden; /* No scrolling allowed */
            touch-action: none; /* Disables browser gestures */
            font-family: 'Courier New', Courier, monospace;
            height: 100dvh; /* Dynamic viewport height for mobile browsers */
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

        /* UI Overlays */
        #ui-layer {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            pointer-events: none; /* Let clicks pass through to canvas */
            display: flex;
            flex-direction: column;
            justify-content: space-between;
        }

        #scoreBoard {
            text-align: center;
            margin-top: 40px; /* Space for notches */
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
            pointer-events: auto; /* Re-enable clicks */
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
        
        /* Flashing effect for Rekt */
        .rekt { animation: shake 0.5s; }
        @keyframes shake {
            0% { transform: translate(-50%, -50%) translate(1px, 1px) rotate(0deg); }
            10% { transform: translate(-50%, -50%) translate(-1px, -2px) rotate(-1deg); }
            20% { transform: translate(-50%, -50%) translate(-3px, 0px) rotate(1deg); }
            30% { transform: translate(-50%, -50%) translate(3px, 2px) rotate(0deg); }
            40% { transform: translate(-50%, -50%) translate(1px, -1px) rotate(1deg); }
            50% { transform: translate(-50%, -50%) translate(-1px, 2px) rotate(-1deg); }
            60% { transform: translate(-50%, -50%) translate(-3px, 1px) rotate(0deg); }
            70% { transform: translate(-50%, -50%) translate(3px, 1px) rotate(-1deg); }
            80% { transform: translate(-50%, -50%) translate(-1px, -1px) rotate(1deg); }
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
            <button onclick="startGame()">HODL & JUMP</button>
        </div>

        <div id="gameOverScreen" class="screen-modal hidden">
            <h1 style="color: #ef4444;">REKT! 📉</h1>
            <p>You fell into the bear trap.</p>
            <p>Final Cap: <span id="finalScore" style="color:#22c55e; font-weight:bold;">$0</span></p>
            <button onclick="startGame()">BUY THE DIP (RETRY)</button>
        </div>
    </div>

<script>
    // --- TELEGRAM SETUP ---
    const tg = window.Telegram.WebApp;
    tg.expand(); // Force full screen
    tg.ready();

    // --- ENGINE SETUP ---
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    const scoreEl = document.getElementById('scoreBoard');
    const startScrn = document.getElementById('startScreen');
    const endScrn = document.getElementById('gameOverScreen');
    const finalScoreEl = document.getElementById('finalScore');

    let width, height;
    
    // Resize handling for mobile rotation/different screens
    function resize() {
        width = window.innerWidth;
        height = window.innerHeight;
        canvas.width = width;
        canvas.height = height;
    }
    window.addEventListener('resize', resize);
    resize();

    // --- GAME STATE ---
    let gameRunning = false;
    let score = 0;
    let platforms = [];
    let keys = { left: false, right: false };

    // --- PHYSICS CONFIG ---
    const CONFIG = {
        gravity: 0.25,
        speed: 6,
        jumpForce: -9,
        platformWidth: 70,
        platformHeight: 20,
        platformGap: 110
    };

    // --- MEERKAT OBJECT ---
    let meerkat = {
        x: 0,
        y: 0,
        w: 40,
        h: 50,
        vx: 0,
        vy: 0,
        faceDir: 1 // 1 = right, -1 = left
    };

    // --- INPUT HANDLING (Touch & Keyboard) ---
    // Touch - split screen logic
    canvas.addEventListener('touchstart', (e) => {
        e.preventDefault();
        handleInput(e.touches[0].clientX);
    }, {passive: false});

    canvas.addEventListener('touchmove', (e) => {
        e.preventDefault();
        handleInput(e.touches[0].clientX);
    }, {passive: false});

    canvas.addEventListener('touchend', (e) => {
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

    // Keyboard (for desktop testing)
    window.addEventListener('keydown', (e) => {
        if (e.code === 'ArrowLeft') keys.left = true;
        if (e.code === 'ArrowRight') keys.right = true;
    });
    window.addEventListener('keyup', (e) => {
        if (e.code === 'ArrowLeft') keys.left = false;
        if (e.code === 'ArrowRight') keys.right = false;
    });

    // --- ASSET GENERATION ---
    function initPlatforms() {
        platforms = [];
        let count = Math.ceil(height / CONFIG.platformGap) + 1;
        for (let i = 0; i < count; i++) {
            platforms.push({
                x: Math.random() * (width - CONFIG.platformWidth),
                y: height - (i * CONFIG.platformGap) - 100,
                w: CONFIG.platformWidth,
                h: CONFIG.platformHeight,
                isMoving: score > 2000 && Math.random() > 0.7 // Difficulty scaling
            });
        }
    }

    // --- DRAWING FUNCTIONS ---

    // Custom Vector Meerkat
    function drawMeerkat(x, y, w, h, dir) {
        ctx.save();
        ctx.translate(x + w/2, y + h/2);
        
        // Flip if facing left
        if(dir === -1) ctx.scale(-1, 1);

        // Body
        ctx.fillStyle = '#D97706'; // Dark Gold
        ctx.beginPath();
        ctx.ellipse(0, 10, w/2.2, h/2.5, 0, 0, Math.PI * 2);
        ctx.fill();

        // Belly Patch
        ctx.fillStyle = '#FCD34D'; // Light Yellow
        ctx.beginPath();
        ctx.ellipse(0, 12, w/4, h/3, 0, 0, Math.PI * 2);
        ctx.fill();

        // Head
        ctx.fillStyle = '#D97706';
        ctx.beginPath();
        ctx.arc(0, -15, w/2, 0, Math.PI*2);
        ctx.fill();

        // Ears
        ctx.fillStyle = '#78350F'; // Dark Brown
        ctx.beginPath();
        ctx.arc(-12, -22, 5, 0, Math.PI*2); // L
        ctx.arc(12, -22, 5, 0, Math.PI*2);  // R
        ctx.fill();

        // Eyes (Mask)
        ctx.fillStyle = '#78350F';
        ctx.beginPath();
        ctx.ellipse(-6, -15, 6, 4, 0, 0, Math.PI*2);
        ctx.ellipse(6, -15, 6, 4, 0, 0, Math.PI*2);
        ctx.fill();

        // Eyes (Pupils)
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
        // Wick
        ctx.fillStyle = '#14532d';
        ctx.fillRect(p.x + p.w/2 - 1, p.y - 6, 2, 6);
        
        // Body (Green Candle)
        ctx.fillStyle = '#22c55e';
        ctx.fillRect(p.x, p.y, p.w, p.h);
        
        // Shine/Highlight
        ctx.fillStyle = '#86efac';
        ctx.fillRect(p.x + 2, p.y + 2, 4, p.h - 4);
        
        // Bottom Shadow
        ctx.fillStyle = '#15803d';
        ctx.fillRect(p.x, p.y + p.h, p.w, 4);
    }

    // --- GAME LOOP ---
    function update() {
        if (!gameRunning) return;

        // 1. Movement
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

        // 2. Physics
        meerkat.vy += CONFIG.gravity;
        meerkat.y += meerkat.vy;

        // 3. Collision
        if (meerkat.vy > 0) {
            platforms.forEach(p => {
                // Generous hitbox for fun gameplay
                if (
                    meerkat.x + meerkat.w * 0.7 > p.x &&
                    meerkat.x + meerkat.w * 0.3 < p.x + p.w &&
                    meerkat.y + meerkat.h > p.y &&
                    meerkat.y + meerkat.h < p.y + p.h + 10 // +10 tolerance
                ) {
                    meerkat.vy = CONFIG.jumpForce;
                    // Haptic Feedback (Telegram only)
                    if(tg.HapticFeedback) tg.HapticFeedback.impactOccurred('light');
                }
            });
        }

        // 4. Scrolling / Score
        if (meerkat.y < height / 2.5) {
            let shift = (height / 2.5) - meerkat.y;
            meerkat.y = height / 2.5;
            score += Math.floor(shift);
            scoreEl.innerText = "Market Cap: $" + score.toLocaleString();

            platforms.forEach(p => {
                p.y += shift;
                
                // Difficulty: Moving platforms
                if(p.isMoving) {
                    p.x += Math.sin(Date.now() / 500) * 2;
                }

                if (p.y > height) {
                    p.y = -20;
                    p.x = Math.random() * (width - CONFIG.platformWidth);
                    p.isMoving = score > 3000 && Math.random() > 0.6; // Incr difficulty
                }
            });
        }

        // 5. Game Over
        if (meerkat.y > height) {
            gameOver();
        }

        draw();
        requestAnimationFrame(update);
    }

    function draw() {
        // Gradient Background
        let grad = ctx.createLinearGradient(0, 0, 0, height);
        grad.addColorStop(0, '#020617');
        grad.addColorStop(1, '#1e293b');
        ctx.fillStyle = grad;
        ctx.fillRect(0, 0, width, height);

        // Draw Platforms
        platforms.forEach(p => drawCandle(p));

        // Draw Meerkat
        drawMeerkat(meerkat.x, meerkat.y, meerkat.w, meerkat.h, meerkat.faceDir);
    }

    function startGame() {
        startScrn.classList.add('hidden');
        endScrn.classList.add('hidden');
        endScrn.classList.remove('rekt');
        
        score = 0;
        scoreEl.innerText = "Market Cap: $0";
        
        initPlatforms();
        
        meerkat.x = width / 2 - 20;
        meerkat.y = height - 200;
        meerkat.vy = -5; // Initial pop
        
        gameRunning = true;
        update();
    }

    function gameOver() {
        gameRunning = false;
        
        // Trigger Haptic "Heavy"
        if(tg.HapticFeedback) tg.HapticFeedback.notificationOccurred('error');

        finalScoreEl.innerText = "$" + score.toLocaleString();
        endScrn.classList.remove('hidden');
        endScrn.classList.add('rekt');
    }

</script>
</body>
</html>
