<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Meerkat Miner</title>
    <script src="https://telegram.org/js/telegram-web-app.js"></script>
    <style>
        /* 1. RESET everything to fit single screen */
        * { box-sizing: border-box; }
        
        body {
            background-color: #000;
            color: white;
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
            margin: 0;
            padding: 0;
            height: 100vh; /* Full height */
            height: 100dvh; /* Dynamic full height for mobile browsers */
            overflow: hidden; /* No scrolling allowed */
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: space-between; /* Space out Top, Middle, Bottom */
            touch-action: manipulation;
        }

        /* 2. TOP SECTION (Score) */
        .header {
            padding-top: 20px;
            text-align: center;
            flex: 0 0 auto; /* Don't grow */
        }
        .balance-title { color: #888; font-size: 14px; text-transform: uppercase; letter-spacing: 1px; }
        #balance { font-size: 12vw; /* Text scales with screen width */ font-weight: 800; margin: 5px 0; }
        .currency { color: #eab308; font-weight: bold; }

        /* 3. MIDDLE SECTION (The Meerkat Button) */
        .game-area {
            flex: 1; /* Take up all available space */
            display: flex;
            justify-content: center;
            align-items: center;
            width: 100%;
        }

        #clickBtn {
            /* RESPONSIVE SIZE: Always 60% of the screen width */
            width: 60vw; 
            height: 60vw;
            
            /* Max limits so it doesn't get huge on desktop */
            max-width: 300px; 
            max-height: 300px;

            background: radial-gradient(circle at 30% 30%, #fde047, #ca8a04);
            border-radius: 50%;
            border: 8px solid #713f12; /* Dark Brown Border */
            box-shadow: 0 10px 30px rgba(234, 179, 8, 0.3), inset 0 -10px 10px rgba(0,0,0,0.2);
            
            display: flex;
            justify-content: center;
            align-items: center;
            cursor: pointer;
            position: relative;
            
            /* Disable blue highlight on tap */
            -webkit-tap-highlight-color: transparent; 
            transition: transform 0.05s ease;
        }

        /* The Meerkat Face (Emoji size scales too) */
        #clickBtn span {
            font-size: 30vw; /* Icon is half the button size */
            filter: drop-shadow(0 5px 5px rgba(0,0,0,0.3));
            pointer-events: none; /* Let clicks pass to button */
        }

        #clickBtn:active { transform: scale(0.92); }

        /* 4. BOTTOM SECTION (Energy) */
        .footer {
            width: 85%;
            padding-bottom: 30px; /* Safe space from bottom */
            flex: 0 0 auto;
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
            font-size: 24px;
            font-weight: bold;
            pointer-events: none;
            animation: floatUp 0.8s ease-out forwards;
            text-shadow: 0 2px 4px rgba(0,0,0,0.5);
            z-index: 100;
        }
        @keyframes floatUp {
            0% { transform: translateY(0) scale(1); opacity: 1; }
            100% { transform: translateY(-80px) scale(1.5); opacity: 0; }
        }
    </style>
</head>
<body>

    <div class="header">
        <div class="balance-title">Mining Balance</div>
        <div id="balance">0</div>
        <div class="currency">$MEERKAT</div>
    </div>

    <div class="game-area">
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
    // 1. Initialize Telegram
    const tg = window.Telegram.WebApp;
    tg.ready();
    tg.expand();
    
    // Fix: Force body height to handle mobile browser bars
    document.body.style.height = window.innerHeight + "px";

    // 2. Game Data
    let coins = localStorage.getItem('meerkat_coins') ? parseInt(localStorage.getItem('meerkat_coins')) : 0;
    let energy = 500;
    const maxEnergy = 500;

    // Update UI immediately
    document.getElementById('balance').innerText = coins.toLocaleString();

    function tap(e) {
        if (energy > 0) {
            // Logic
            coins++;
            energy--;
            
            // Save
            localStorage.setItem('meerkat_coins', coins);

            // Update UI
            updateUI();
            
            // Haptics (Vibration)
            if(tg.HapticFeedback) tg.HapticFeedback.impactOccurred('medium');

            // Floating Text Effect
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

    // Energy Regen Loop
    setInterval(() => {
        if(energy < maxEnergy) {
            energy++;
            updateUI();
        }
    }, 1000);

</script>
</body>
</html>
