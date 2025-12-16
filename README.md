<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Meerkat Miner</title>
    <script src="https://telegram.org/js/telegram-web-app.js"></script>
    <style>
        /* RESET & NO SCROLLING */
        * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
        
        body {
            background-color: #000;
            color: white;
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
            margin: 0;
            padding: 0;
            overflow: hidden; /* Stop scrolling completely */
            position: fixed; /* Lock body in place */
            width: 100%;
            height: 100%;
            top: 0;
            left: 0;
        }

        /* 1. TOP HEADER (Fixed to top) */
        .header {
            position: absolute;
            top: 20px; /* Safe distance from top */
            left: 0;
            width: 100%;
            text-align: center;
            z-index: 10;
        }
        .balance-title { color: #888; font-size: 12px; text-transform: uppercase; letter-spacing: 1px; }
        #balance { font-size: 48px; font-weight: 800; margin: 0; line-height: 1.2; }
        .currency { color: #eab308; font-weight: bold; font-size: 14px; }

        /* 2. CENTER GAME AREA (The Meerkat) */
        .game-container {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -60%); /* Shift up slightly so footer doesn't cover it */
            width: auto;
            height: auto;
            z-index: 5;
            display: flex;
            justify-content: center;
            align-items: center;
        }

        #clickBtn {
            /* Responsive Circle: 70% of screen width, but max 300px */
            width: 70vw;
            height: 70vw;
            max-width: 280px;
            max-height: 280px;

            background: radial-gradient(circle at 30% 30%, #fde047, #ca8a04);
            border-radius: 50%;
            border: 8px solid #713f12;
            box-shadow: 0 10px 40px rgba(234, 179, 8, 0.4);
            
            display: flex;
            justify-content: center;
            align-items: center;
            cursor: pointer;
            transition: transform 0.05s ease;
        }

        #clickBtn:active { transform: scale(0.95); }

        /* The Emoji inside */
        #clickBtn span {
            font-size: 80px; 
            filter: drop-shadow(0 4px 4px rgba(0,0,0,0.3));
            pointer-events: none;
        }

        /* 3. BOTTOM FOOTER (Fixed to bottom) */
        .footer {
            position: absolute;
            bottom: 30px; /* Force 30px up from bottom edge */
            left: 50%;
            transform: translateX(-50%);
            width: 85%;
            z-index: 10;
        }
        
        .energy-text {
            display: flex;
            justify-content: space-between;
            margin-bottom: 8px;
            font-size: 14px;
            color: #ccc;
            font-weight: bold;
        }
        
        .progress-bg {
            background: #333;
            height: 12px;
            border-radius: 6px;
            width: 100%;
            overflow: hidden;
            border: 1px solid #444;
        }
        
        #energyBar {
            background: linear-gradient(90deg, #eab308, #22c55e);
            height: 100%;
            width: 100%;
            transition: width 0.1s linear;
        }

        /* Floating +1 Animation */
        .pop {
            position: absolute;
            color: white;
            font-size: 28px;
            font-weight: 900;
            pointer-events: none;
            animation: floatUp 0.8s ease-out forwards;
            z-index: 100;
            text-shadow: 0 2px 5px rgba(0,0,0,0.5);
        }
        @keyframes floatUp {
            0% { transform: translateY(0) scale(1); opacity: 1; }
            100% { transform: translateY(-100px) scale(1.5); opacity: 0; }
        }
    </style>
</head>
<body>

    <div class="header">
        <div class="balance-title">Mining Balance</div>
        <div id="balance">0</div>
        <div class="currency">$MEERKAT</div>
    </div>

    <div class="game-container">
        <div id="clickBtn" onclick="tap(event)">
            <span>🐹</span>
        </div>
    </div>

    <div class="footer">
        <div class="energy-text">
            <span>⚡ Energy</span>
            <span id="energyVal">500 / 500</span>
        </div>
        <div class="progress-bg">
            <div id="energyBar"></div>
        </div>
    </div>

<script>
    // 1. TELEGRAM SETUP
    const tg = window.Telegram.WebApp;
    tg.ready();
    tg.expand(); // Force full height

    // 2. FORCE HEIGHT FIX (Crucial for mobile)
    // This constantly updates the body height to match the visible window
    function setHeight() {
        document.body.style.height = window.innerHeight + "px";
    }
    window.addEventListener('resize', setHeight);
    setHeight(); // Call immediately

    // 3. GAME LOGIC
    let coins = localStorage.getItem('meerkat_coins') ? parseInt(localStorage.getItem('meerkat_coins')) : 0;
    let energy = 500;
    const maxEnergy = 500;

    // Update UI immediately
    document.getElementById('balance').innerText = coins.toLocaleString();

    function tap(e) {
        if (energy > 0) {
            coins++;
            energy--;
            
            // Save & Update
            localStorage.setItem('meerkat_coins', coins);
            updateUI();
            
            // Haptics
            if(tg.HapticFeedback) tg.HapticFeedback.impactOccurred('medium');

            // Floating Text
            showFloat(e.clientX, e.clientY);
        } else {
            if(tg.HapticFeedback) tg.HapticFeedback.notificationOccurred('error');
        }
    }

    function updateUI() {
        document.getElementById('balance').innerText = coins.toLocaleString();
        document.getElementById('energyVal').innerText = energy + " / " + maxEnergy;
        document.getElementById('energyBar').style.width = (energy / maxEnergy * 100) + "%";
    }

    function showFloat(x, y) {
        const float = document.createElement('div');
        float.innerText = "+1";
        float.className = 'pop';
        float.style.left = x + "px";
        float.style.top = y + "px";
        document.body.appendChild(float);
        setTimeout(() => float.remove(), 800);
    }

    // Energy Regen
    setInterval(() => {
        if(energy < maxEnergy) {
            energy++;
            updateUI();
        }
    }, 1000);

</script>
</body>
</html>
