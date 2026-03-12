# <!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
    <title>Flappy Probability: Ultra 60FPS</title>
    <style>
        :root {
            --primary: #FFD700;
            --secondary: #FF8C00;
            --sky-top: #4facfe;
            --sky-bottom: #00f2fe;
            --grass: #2ecc71;
        }

        body {
            margin: 0; background-color: #0f172a;
            display: flex; justify-content: center; align-items: center;
            min-height: 100vh; font-family: 'Kanit', sans-serif;
            overflow: hidden; touch-action: none; -webkit-user-select: none;
        }

        #device-frame {
            width: 360px; height: 640px;
            background: #1e293b; border: 10px solid #334155;
            border-radius: 45px; position: relative;
            box-shadow: 0 50px 100px rgba(0,0,0,0.8);
            overflow: hidden;
        }

        canvas { display: block; background: var(--sky-top); }

        /* UI Overlays */
        .overlay {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            display: flex; flex-direction: column; justify-content: center;
            align-items: center; z-index: 100; text-align: center;
            transition: opacity 0.4s ease;
        }

        .glass-panel {
            background: rgba(255, 255, 255, 0.95);
            backdrop-filter: blur(10px); width: 85%; padding: 30px 20px;
            border-radius: 35px; border: 4px solid var(--primary);
            box-shadow: 0 20px 40px rgba(0,0,0,0.4);
        }

        #hud {
            position: absolute; top: 35px; width: 100%;
            display: flex; justify-content: space-around;
            pointer-events: none; z-index: 10;
        }

        .badge {
            background: rgba(15, 23, 42, 0.6); color: white;
            padding: 8px 18px; border-radius: 50px;
            font-size: 20px; font-weight: bold; border: 2px solid var(--primary);
        }

        button {
            background: linear-gradient(135deg, var(--secondary), #f39c12);
            color: white; border: none; padding: 16px 45px;
            font-size: 20px; border-radius: 50px; font-weight: 800;
            cursor: pointer; box-shadow: 0 6px 0 #d35400;
            transition: 0.1s; margin-top: 20px;
        }
        button:active { transform: translateY(4px); box-shadow: 0 2px 0 #d35400; }

        input {
            width: 90%; padding: 15px; margin: 20px 0;
            border: 3px solid #e2e8f0; border-radius: 15px;
            font-size: 24px; text-align: center; outline: none;
        }

        .hidden { opacity: 0; pointer-events: none; }
        #tap-info { position: absolute; bottom: 100px; color: white; font-size: 18px; animation: pulse 1s infinite; }
        @keyframes pulse { 0% { opacity: 0.5; } 50% { opacity: 1; } 100% { opacity: 0.5; } }
    </style>
</head>
<body>

<div id="device-frame">
    <div id="hud" class="hidden">
        <div class="badge">Score: <span id="score">0</span></div>
        <div class="badge">❤️ <span id="hearts">0</span></div>
    </div>

    <canvas id="gameCanvas"></canvas>

    <div id="start-screen" class="overlay">
        <h1 style="color: white; font-size: 38px; margin-bottom: 0; text-shadow: 0 4px 15px rgba(0,0,0,0.5);">PROBABILITY</h1>
        <h2 style="color: var(--primary); font-size: 45px; margin-top: -10px; text-shadow: 3px 3px 0 #000;">BIRD</h2>
        <div style="font-size: 90px; margin: 20px 0;">🐥</div>
        <button id="btn-start">LET'S FLY</button>
    </div>

    <div id="quiz-ui" class="overlay hidden">
        <div class="glass-panel">
            <h2 style="margin: 0; color: var(--secondary); font-size: 28px;">CHALLENGE!</h2>
            <p id="q-text" style="font-size: 18px; margin: 15px 0; color: #334155; font-weight: 600;"></p>
            <input type="text" id="q-input" placeholder="?" autocomplete="off">
            <button id="btn-submit">SUBMIT</button>
            <p style="margin-top: 15px; color: #64748b;">Chance Left: <span id="q-lives">3</span></p>
        </div>
    </div>

    <div id="tap-info" class="hidden">TAP TO FLAP</div>
</div>

<script>
    // --- Audio System (Optimized for Mobile) ---
    const actx = new (window.AudioContext || window.webkitAudioContext)();
    function sfx(f, t, d) {
        if (actx.state === 'suspended') actx.resume();
        const o = actx.createOscillator(), g = actx.createGain();
        o.type = t; o.frequency.setValueAtTime(f, actx.currentTime);
        g.gain.setValueAtTime(0.1, actx.currentTime);
        g.gain.exponentialRampToValueAtTime(0.01, actx.currentTime + d);
        o.connect(g); g.connect(actx.destination);
        o.start(); o.stop(actx.currentTime + d);
    }

    // --- Game Logic ---
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    canvas.width = 360; canvas.height = 640;

    let state = "START", score = 0, hearts = 0, frame = 0, pipes = [], qTries = 3;
    let bgClouds = [{x: 100, y: 100, s: 0.3}, {x: 400, y: 200, s: 0.2}];

    const bird = {
        x: 80, y: 320, w: 40, h: 30, v: 0, g: 0.25, jump: 5.5,
        draw() {
            ctx.save(); ctx.translate(this.x + this.w/2, this.y + this.h/2);
            ctx.rotate(Math.min(Math.PI/4, Math.max(-Math.PI/4, this.v * 0.07)));
            // Body
            ctx.fillStyle = "#FFD700"; ctx.beginPath(); ctx.roundRect(-this.w/2, -this.h/2, this.w, this.h, 10); ctx.fill();
            // Wing
            ctx.fillStyle = "#E67E22"; ctx.fillRect(-18, Math.sin(frame*0.2)*5, 18, 10);
            // Eye
            ctx.fillStyle = "white"; ctx.fillRect(10, -10, 12, 12);
            ctx.fillStyle = "black"; ctx.fillRect(16, -8, 5, 5);
            // Beak
            ctx.fillStyle = "#FF4500"; ctx.fillRect(this.w/2 - 2, 0, 14, 10);
            ctx.restore();
        },
        update() {
            if (state !== "PLAYING") return;
            this.v += this.g; this.y += this.v;
            if (this.y + this.h > canvas.height - 80) triggerGameOver();
        }
    };

    class Pipe {
        constructor() {
            this.x = canvas.width; this.w = 65; this.gap = 175;
            this.top = Math.random() * (canvas.height - 350) + 90;
            this.passed = false;
        }
        draw() {
            const grd = ctx.createLinearGradient(this.x, 0, this.x + this.w, 0);
            grd.addColorStop(0, '#27ae60'); grd.addColorStop(1, '#2ecc71');
            ctx.fillStyle = grd;
            ctx.fillRect(this.x, 0, this.w, this.top);
            ctx.fillRect(this.x, this.top + this.gap, this.w, canvas.height);
            // Caps
            ctx.fillStyle = "#1b5e20";
            ctx.fillRect(this.x - 5, this.top - 25, this.w + 10, 25);
            ctx.fillRect(this.x - 5, this.top + this.gap, this.w + 10, 25);
        }
        update() {
            if (state !== "PLAYING") return;
            this.x -= 2.8;
            // Collision
            if (bird.x + bird.w - 6 > this.x && bird.x + 6 < this.x + this.w && (bird.y + 6 < this.top || bird.y + bird.h - 6 > this.top + this.gap)) triggerGameOver();
            // Score
            if (!this.passed && bird.x > this.x + this.w) {
                score++; this.passed = true; sfx(600, 'sine', 0.1);
                document.getElementById('score').innerText = score;
                if (score % 5 === 0) startQuiz();
            }
        }
    }

    const quizzes = [
        { q: "ความน่าจะเป็นที่เหรียญจะออกก้อย? (เศษส่วน)", a: "1/2" },
        { q: "ทอดลูกเต๋า 1 ลูก โอกาสได้เลข 6? (เศษส่วน)", a: "1/6" },
        { q: "ถุงมีลูกบอลสีขาว 3 สีดำ 7 สุ่มหยิบได้สีขาวคือเท่าใด?", a: "3/10" },
        { q: "ความน่าจะเป็นที่เหตุการณ์จะเกิดขึ้น 100% คือเลขอะไร?", a: "1" },
        { q: "โยนเหรียญ 2 อันพร้อมกัน โอกาสออกหัวทั้งคู่? (เศษส่วน)", a: "1/4" }
    ];

    function handleInput(e) {
        if (e.cancelable) e.preventDefault();
        if (state === "WAITING" || state === "PLAYING") {
            state = "PLAYING"; bird.v = -bird.jump; sfx(450, 'sine', 0.1);
            document.getElementById('tap-info').classList.add('hidden');
        }
    }

    window.addEventListener('touchstart', handleInput, {passive: false});
    window.addEventListener('mousedown', (e) => { if(e.target.tagName !== 'BUTTON' && e.target.tagName !== 'INPUT') handleInput(e); });

    document.getElementById('btn-start').onclick = () => {
        state = "WAITING"; score = 0; hearts = 0; pipes = []; bird.y = 320; bird.v = 0;
        document.getElementById('start-screen').classList.add('hidden');
        document.getElementById('hud').classList.remove('hidden');
        document.getElementById('tap-info').classList.remove('hidden');
        document.getElementById('score').innerText = "0";
        document.getElementById('hearts').innerText = "0";
    };

    function startQuiz() {
        state = "QUIZ"; qTries = 3;
        const q = quizzes[Math.floor(Math.random() * quizzes.length)];
        document.getElementById('q-text').innerText = q.q;
        document.getElementById('q-lives').innerText = 3;
        document.getElementById('quiz-ui').classList.remove('hidden');
        document.getElementById('q-input').value = ""; document.getElementById('q-input').focus();

        document.getElementById('btn-submit').onclick = () => {
            if (document.getElementById('q-input').value.trim() === q.a) {
                sfx(800, 'sine', 0.2); hearts++; document.getElementById('hearts').innerText = hearts;
                state = "PLAYING"; document.getElementById('quiz-ui').classList.add('hidden');
            } else {
                sfx(150, 'sawtooth', 0.3); qTries--; document.getElementById('q-lives').innerText = qTries;
                if (qTries <= 0) {
                    alert("เสียดายจัง! คำตอบคือ: " + q.a + "\nสู้ๆ นะ บินต่อไป!");
                    state = "PLAYING"; document.getElementById('quiz-ui').classList.add('hidden');
                }
            }
        };
    }

    function triggerGameOver() {
        sfx(100, 'square', 0.3);
        if (hearts > 0) {
            hearts--; document.getElementById('hearts').innerText = hearts;
            pipes = []; bird.y = 200; bird.v = -2; return;
        }
        alert("GAME OVER! สถิติของคุณ: " + score);
        state = "START"; document.getElementById('start-screen').classList.remove('hidden');
        document.getElementById('hud').classList.add('hidden');
    }

    function engine() {
        // Drawing background
        const g = ctx.createLinearGradient(0, 0, 0, canvas.height);
        g.addColorStop(0, '#4facfe'); g.addColorStop(1, '#00f2fe');
        ctx.fillStyle = g; ctx.fillRect(0, 0, canvas.width, canvas.height);

        // Clouds
        ctx.fillStyle = "rgba(255,255,255,0.7)";
        bgClouds.forEach(c => {
            ctx.beginPath(); ctx.arc(c.x, c.y, 30, 0, 7); ctx.arc(c.x+25, c.y-15, 30, 0, 7); ctx.arc(c.x+50, c.y, 30, 0, 7); ctx.fill();
            c.x -= c.s; if (c.x < -100) c.x = canvas.width + 100;
        });

        pipes.forEach((p, i) => { p.update(); p.draw(); if (p.x < -100) pipes.splice(i, 1); });

        // Ground
        ctx.fillStyle = "#8d6e63"; ctx.fillRect(0, canvas.height - 80, canvas.width, 80);
        ctx.fillStyle = "#4caf50"; ctx.fillRect(0, canvas.height - 80, canvas.width, 15);

        bird.update(); bird.draw();
        if (state === "PLAYING") {
            frame++;
            if (frame % 110 === 0) pipes.push(new Pipe());
        }
        requestAnimationFrame(engine);
    }

    engine();
</script>
</body>
</html>
