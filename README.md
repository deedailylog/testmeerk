<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Meerkat Moon Jump</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            background-color: #0f172a; /* Dark Crypto Theme */
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            font-family: 'Courier New', Courier, monospace;
            overflow: hidden;
            touch-action: none; /* Prevents zooming on mobile */
        }
        #gameCanvas {
            border: 2px solid #22c55e; /* Green Candle Border */
            background: linear-gradient(to bottom, #000000, #1e293b);
            box-shadow: 0 0 20px rgba(34, 197, 94, 0.2);
            max-width: 100%;
            max-height: 100%;
        }
        #scoreBoard {
            position: absolute;
            top: 20px;
            left: 50%;
            transform: translateX(-50%);
            color: #fff;
            font-size: 24px;
            font-weight: bold;
            text-shadow: 2px 2px #000;
            pointer-events: none;
        }
        #startScreen, #gameOverScreen {
            position: absolute;
            color: white;
            text-align: center;
            background: rgba(0, 0, 0, 0.85);
            padding: 20px;
            border-radius: 10px;
            border: 1px solid #22c55e;
            display: flex;
            flex-direction: column;
            gap: 10px;
        }
        button {
            background: #22c55e;
            color: black;
            border: none;
            padding: 10px 20px;
            font-size: 18px;
            font-weight: bold;
            cursor: pointer;
            border-radius: 5px;
        }
        button:hover {
            background: #16a34a;
        }
        .hidden {
            display: none !important;
        }
    </style>
</head>
<body>

    <div id="scoreBoard">Market Cap: $0</div>
    <canvas id="gameCanvas" width="375" height="667"></canvas>

    <div id="startScreen">
        <h1>Meerkat Moon Jump</h1>
        <p>Tap left/right to move</p>
        <p>Jump on Green Candles 🕯️</p>
        <p>Don't fall into the Bear Market!</p>
        <button onclick="startGame()">LAUNCH 🚀</button>
    </div>

    <div id="gameOverScreen" class="hidden">
        <h1 style="color: #ef4444;">RUG PULL!</h1>
        <p>You fell into the dip.</p>
        <p>Final Market Cap: <span id="finalScore">$0</span></p>
        <button onclick="resetGame()">TRY AGAIN 🔄</button>
    </div>

<script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    const scoreElement = document.getElementById('scoreBoard');
    const startScreen = document.getElementById('startScreen');
    const gameOverScreen = document.getElementById('gameOverScreen');
    const finalScoreSpan = document.getElementById('finalScore');

    // Game Variables
    let gameRunning = false;
    let score = 0;
    let width = canvas.width;
    let height = canvas.height;
    
    // Physics
    let gravity = 0.2;
    let platforms = [];
    
    // Meerkat Object
    let meerkat = {
        x: width / 2 - 20,
        y: height - 150,
        width: 40,
        height: 40,
        vx: 0,
        vy: 0,
        speed: 5,
        jumpStrength: -8
    };

    // Controls
    let keys = {};
    window.addEventListener('keydown', (e) => keys[e.code] = true);
    window.addEventListener('keyup', (e) => keys[e.code] = false);

    // Touch Controls for Mobile
    let touchX = null;
    canvas.addEventListener('touchstart', (e) => {
        touchX = e.touches[0].clientX;
        e.preventDefault();
    });
    canvas.addEventListener('touchmove', (e) => {
        let currentTouch = e.touches[0].clientX;
        if (currentTouch < touchX) keys['ArrowLeft'] = true; 
        else keys['ArrowRight'] = true;
        touchX = currentTouch;
    });
    canvas.addEventListener('touchend', () => {
        keys['ArrowLeft'] = false;
        keys['ArrowRight'] = false;
    });

    // Load Meerkat Image (Optional - currently draws a square)
    // To use your sticker: Replace the ctx.fillRect line in draw() with ctx.drawImage(img, ...)
    
    function initPlatforms() {
        platforms = [];
        let platformCount = 7;
        for (let i = 0; i < platformCount; i++) {
            platforms.push({
                x: Math.random() * (width - 60),
                y: height - (i * 100) - 50,
                width: 60,
                height: 15, // Thin like a candle wick? No, thick for gameplay.
                type: 'green' // Green candles are safe
            });
        }
    }

    function update() {
        if (!gameRunning) return;

        // 1. Move Meerkat Left/Right
        if (keys['ArrowLeft']) meerkat.x -= meerkat.speed;
        if (keys['ArrowRight']) meerkat.x += meerkat.speed;

        // Wrap around screen
        if (meerkat.x + meerkat.width < 0) meerkat.x = width;
        if (meerkat.x > width) meerkat.x = -meerkat.width;

        // 2. Gravity
        meerkat.vy += gravity;
        meerkat.y += meerkat.vy;

        // 3. Jump Logic (Collision with Green Candles)
        if (meerkat.vy > 0) { // Only jump if falling
            platforms.forEach(p => {
                if (
                    meerkat.x < p.x + p.width &&
                    meerkat.x + meerkat.width > p.x &&
                    meerkat.y + meerkat.height > p.y &&
                    meerkat.y + meerkat.height < p.y + p.height
                ) {
                    meerkat.vy = meerkat.jumpStrength; // BOUNCE!
                }
            });
        }

        // 4. Camera Scroll (Infinite World)
        if (meerkat.y < height / 2) {
            let shift = (height / 2) - meerkat.y;
            meerkat.y = height / 2;
            
            score += Math.floor(shift); // Score based on height
            scoreElement.innerText = "Market Cap: $" + score.toLocaleString();

            platforms.forEach(p => {
                p.y += shift;
                // Recycle platforms that go off bottom
                if (p.y > height) {
                    p.y = -20;
                    p.x = Math.random() * (width - 60);
                }
            });
        }

        // 5. Game Over (Fall into the bear market)
        if (meerkat.y > height) {
            endGame();
        }

        draw();
        requestAnimationFrame(update);
    }

    function draw() {
        // Clear screen
        ctx.clearRect(0, 0, width, height);

        // Draw Platforms (Green Candles)
        ctx.fillStyle = '#22c55e'; // Green
        platforms.forEach(p => {
            // Draw Candle Body
            ctx.fillRect(p.x, p.y, p.width, p.height);
            // Draw Wick
            ctx.fillStyle = '#166534';
            ctx.fillRect(p.x + p.width/2 - 2, p.y + 15, 4, 20); // Wick below? No, visually just blocks.
            ctx.fillStyle = '#22c55e'; // Reset for next
        });

        // Draw Meerkat
        ctx.fillStyle = '#eab308'; // Gold/Meerkat Color
        ctx.fillRect(meerkat.x, meerkat.y, meerkat.width, meerkat.height);
        
        // Simple Face
        ctx.fillStyle = 'black';
        ctx.fillRect(meerkat.x + 5, meerkat.y + 10, 5, 5); // Eye L
        ctx.fillRect(meerkat.x + 30, meerkat.y + 10, 5, 5); // Eye R
        ctx.fillRect(meerkat.x + 15, meerkat.y + 25, 10, 5); // Mouth
    }

    function startGame() {
        startScreen.classList.add('hidden');
        gameOverScreen.classList.add('hidden');
        score = 0;
        meerkat.y = height - 150;
        meerkat.vy = 0;
        initPlatforms();
        gameRunning = true;
        update();
    }

    function resetGame() {
        startGame();
    }

    function endGame() {
        gameRunning = false;
        finalScoreSpan.innerText = "$" + score.toLocaleString();
        gameOverScreen.classList.remove('hidden');
    }

    // Initial Draw
    ctx.fillStyle = 'black';
    ctx.fillRect(0, 0, width, height);
</script>
</body>
</html>
