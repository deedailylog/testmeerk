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
            overflow: hidden;
            background-color: #0f172a; /* Dark Crypto Theme */
            font-family: 'Courier New', Courier, monospace;
            touch-action: none; /* Disables scrolling/zooming */
        }
        canvas {
            display: block;
        }
        #ui {
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
        #score {
            text-align: center;
            color: #22c55e;
            font-size: 24px;
            font-weight: bold;
            margin-top: 20px;
            text-shadow: 2px 2px 0px #000;
        }
        #startScreen, #gameOverScreen {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: rgba(0, 0, 0, 0.9);
            padding: 25px;
            border: 2px solid #22c55e;
            border-radius: 15px;
            text-align: center;
            color: white;
            pointer-events: auto; /* Enable clicking buttons */
            box-shadow: 0 0 20px rgba(34, 197, 94, 0.3);
            min-width: 250px;
        }
        button {
            background: #22c55e;
            color: black;
            border: none;
            padding: 12px 25px;
            font-size: 18px;
            font-weight: bold;
            cursor: pointer;
            border-radius: 8px;
            margin-top: 15px;
            font-family: inherit;
        }
        button:active { transform: scale(0.95); }
        .hidden { display: none !important; }
    </style>
</head>
<body>

    <div id="ui">
        <div id="score">CAP: $0</div>
    </div>

    <div id="startScreen">
        <h1 style="margin:0 0 10px 0;">Moon Jump 🚀</h1>
        <p style="color:#aaa; font-size:14px;">Tap Left/Right to Move</p>
        <button onclick="startGame()">START</button>
    </div>

    <div id="gameOverScreen" class="hidden">
        <h1 style="color: #ef4444; margin:0;">RUG PULL!</h1>
        <p>You fell.</p>
        <p>Final Cap: <span id="finalScore">$0</span></p>
        <button onclick="resetGame()">AGAIN 🔄</button>
    </div>

    <canvas id="gameCanvas"></canvas>

<script>
    // --- TELEGRAM SETUP ---
    const tg = window.Telegram.WebApp;
    tg.ready();
    tg.expand(); // Make full screen

    // --- GAME SETUP ---
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    
    // Resize canvas to fit ANY screen size
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;

    let gameRunning = false;
    let score = 0;
    
    // Meerkat Variables
    let meerkat = {
        x: canvas.width / 2,
        y: canvas.height - 150,
        w: 30,  // Slimmer body
        h: 45,  // Taller body
        vx: 0,
        vy: 0,
        speed: 6, // Faster movement
        jump: -10
    };

    let gravity = 0.25;
    let platforms = [];

    // Controls
    let keys = {};
    window.addEventListener('keydown', (e) => keys[e.code] = true);
    window.addEventListener('keyup', (e) => keys[e.code] = false);

    // Touch Controls
    let touchX = null;
    window.addEventListener('touchstart', (e) => {
        touchX = e.touches[0].clientX;
    });
    window.addEventListener('touchmove', (e) => {
        e.preventDefault(); // Stop scrolling
        let currentX = e.touches[0].clientX;
        if (currentX < window.innerWidth / 2) { keys['ArrowLeft'] = true; keys['ArrowRight'] = false; }
        else { keys['ArrowRight'] = true; keys['ArrowLeft'] = false; }
    });
    window.addEventListener('touchend', () => {
        keys['ArrowLeft'] = false;
        keys['ArrowRight'] = false;
    });

    // --- DRAWING FUNCTIONS ---

    function drawMeerkat(x, y, w, h) {
        // 1. Body (Tan Color)
        ctx.fillStyle = '#eab308'; 
        ctx.beginPath();
        ctx.roundRect(x, y, w, h, 10);
        ctx.fill();

        // 2. Belly (Lighter)
        ctx.fillStyle = '#fde047';
        ctx.beginPath();
        ctx.ellipse(x + w/2, y + h/2 + 5, w/3, h/4, 0, 0, Math.PI * 2);
        ctx.fill();

        // 3. Eyes (Black Patches - Classic Meerkat)
        ctx.fillStyle = 'black';
        ctx.beginPath();
        ctx.ellipse(x + 8, y + 12, 4, 3, 0, 0, Math.PI * 2); // Left
        ctx.ellipse(x + w - 8, y + 12, 4, 3, 0, 0, Math.PI * 2); // Right
        ctx.fill();

        // 4. Eye Sparkle (White dots)
        ctx.fillStyle = 'white';
        ctx.fillRect(x + 7, y + 11, 2, 2);
        ctx.fillRect(x + w - 9, y + 11, 2, 2);

        // 5. Nose & Ears
        ctx.fillStyle = '#713f12'; // Dark brown
        ctx.beginPath();
        ctx.arc(x + w/2, y + 18, 2, 0, Math.PI * 2); // Nose
        ctx.fill();
        
        // Ears
        ctx.beginPath();
        ctx.arc(x, y + 8, 3, 0, Math.PI * 2); // Left Ear
        ctx.arc(x + w, y + 8, 3, 0, Math.PI * 2); // Right Ear
        ctx.fill();
    }

    function drawPlatform(p) {
        // Green Candle Body
        ctx.fillStyle = '#22c55e';
        ctx.fillRect(p.x, p.y, p.w, p.h);
        
        // Wick (Visual only)
        ctx.fillStyle = '#15803d'; 
        ctx.fillRect(p.x + p.w/2 - 1, p.y + p.h, 2, 10); // Wick goes down
    }

    // --- GAME LOOP ---

    function initPlatforms() {
        platforms = [];
        let count = Math.floor(canvas.height / 90); // Density based on screen height
        for (let i = 0; i < count; i++) {
            platforms.push({
                x: Math.random() * (canvas.width - 60),
                y: canvas.height - (i * 90) - 50,
                w: 60,
                h: 15
            });
        }
    }

    function update() {
        if (!gameRunning) return;

        // Move Player
        if (keys['ArrowLeft']) meerkat.x -= meerkat.speed;
        if (keys['ArrowRight']) meerkat.x += meerkat.speed;

        // Screen Wrap
        if (meerkat.x + meerkat.w < 0) meerkat.x = canvas.width;
        if (meerkat.x > canvas.width) meerkat.x = -meerkat.w;

        // Physics
        meerkat.vy += gravity;
        meerkat.y += meerkat.vy;

        // Collision: Jump on Platforms (Green Candles)
        if (meerkat.vy > 0) { 
            platforms.forEach(p => {
                if (
                    meerkat.x + meerkat.w > p.x &&
                    meerkat.x < p.x + p.w &&
                    meerkat.y + meerkat.h > p.y &&
                    meerkat.y + meerkat.h < p.y + p.h + 10 // Tolerance
                ) {
                    meerkat.vy = meerkat.jump;
                    // Optional: Add simple sound effect here
                }
            });
        }

        // Camera Scroll (Infinite World)
        if (meerkat.y < canvas.height / 2) {
