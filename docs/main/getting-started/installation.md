<!DOCTYPE html>
<html lang="ar">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>لعبة الأسطورة - نسخة سرقة السيارات</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            background-color: #333;
            overflow: hidden;
            font-family: sans-serif;
            user-select: none;
            touch-action: none;
        }
        #gameContainer {
            position: relative;
            width: 100vw;
            height: 100vh;
            background-color: #3a3a3a;
        }
        canvas {
            display: block;
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background-color: #3a3a3a;
        }
        #hud {
            position: absolute;
            top: 15px;
            left: 15px;
            color: #fff;
            font-size: 16px;
            font-weight: bold;
            background: rgba(0, 0, 0, 0.85);
            padding: 12px 18px;
            border-radius: 10px;
            border: 2px solid #f1c40f;
            direction: rtl;
            z-index: 10;
        }
        .money { color: #2ecc71; font-size: 18px; }
        .health { color: #e74c3c; }
        .status { color: #f1c40f; font-size: 14px; }

        .controls-container {
            position: absolute;
            bottom: 25px;
            width: 100%;
            display: flex;
            justify-content: space-around;
            align-items: center;
            z-index: 10;
            padding: 0 10px;
            box-sizing: border-box;
        }

        .joystick {
            display: grid;
            grid-template-columns: repeat(3, 70px);
            grid-template-rows: repeat(2, 70px);
            gap: 10px;
        }

        .action-container {
            display: flex;
            flex-direction: column;
            gap: 10px;
            align-items: center;
        }

        .btn {
            background: rgba(255, 255, 255, 0.25);
            color: #fff;
            border: 2px solid rgba(255, 255, 255, 0.6);
            border-radius: 50%;
            font-size: 26px;
            display: flex;
            align-items: center;
            justify-content: center;
            backdrop-filter: blur(5px);
        }
        .btn:active {
            background: rgba(241, 196, 15, 0.8);
            border-color: #f1c40f;
        }

        .btn-fire {
            width: 80px;
            height: 80px;
            background: rgba(231, 76, 60, 0.4);
            border: 3px solid #e74c3c;
            font-size: 15px;
            font-weight: bold;
        }

        /* زرار سرقة العربيات الجديد */
        .btn-steal {
            width: 110px;
            height: 50px;
            border-radius: 25px;
            background: rgba(230, 126, 34, 0.8);
            border: 2px solid #e67e22;
            font-size: 14px;
            font-weight: bold;
            display: none; /* بيظهر بس لما تقرب من عربية */
        }

        #btn-up { grid-column: 2; grid-row: 1; }
        #btn-left { grid-column: 1; grid-row: 2; }
        #btn-down { grid-column: 2; grid-row: 2; }
        #btn-right { grid-column: 3; grid-row: 2; }
    </style>
</head>
<body>

    <div id="gameContainer">
        <div id="hud">
            <div>الـفـلـوس: <span id="score" class="money">0</span> ج.م</div>
            <div style="margin-top: 4px;">الـصـحـة: <span id="health" class="health">100%</span></div>
            <div style="margin-top: 4px;" id="carStatus" class="status">الحالة: على رجلك 🏃‍♂️</div>
        </div>
        <canvas id="gameCanvas"></canvas>

        <div class="controls-container">
            <div class="joystick">
                <div class="btn" id="btn-up">▲</div>
                <div class="btn" id="btn-left">◀</div>
                <div class="btn" id="btn-down">▼</div>
                <div class="btn" id="btn-right">▶</div>
            </div>
            
            <div class="action-container">
                <!-- زرار السرقة والنزول -->
                <div class="btn btn-steal" id="btn-steal">اسرق العربية 🚗</div>
                <!-- زرار الضرب -->
                <div class="btn btn-fire" id="btn-fire">اضرب💥</div>
            </div>
        </div>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');

        function initGame() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
            if(player.x === 100 && player.y === 100) {
                player.x = canvas.width / 2 - 22;
                player.y = canvas.height - 260;
            }
            if(!gameInterval) {
                gameInterval = requestAnimationFrame(gameLoop);
            }
        }

        let score = 0;
        let health = 100;
        let gameOver = false;
        let gameInterval = null;

        const player = { x: 100, y: 100, width: 44, height: 44, speed: 7.5 };
        
        // متغيرات قيادة السيارة المخطوفة
        let isDriving = false;
        let stolenCar = { color: '#fff', width: 44, height: 70, type: 'normal' };
        let closeCarIndex = -1; // لتحديد أقرب عربية يمكن سرقتها

        const obstacles = [];
        const coins = [];
        const bullets = [];

        let moveUp = false, moveDown = false, moveLeft = false, moveRight = false;

        bindButton('btn-up', () => moveUp = true, () => moveUp = false);
        bindButton('btn-down', () => moveDown = true, () => moveDown = false);
        bindButton('btn-left', () => moveLeft = true, () => moveLeft = false);
        bindButton('btn-right', () => moveRight = true, () => moveRight = false);

        const fireBtn = document.getElementById('btn-fire');
        fireBtn.addEventListener('touchstart', (e) => { e.preventDefault(); fireBullet(); });
        fireBtn.addEventListener('mousedown', (e) => { fireBullet(); });

        const stealBtn = document.getElementById('btn-steal');
        stealBtn.addEventListener('touchstart', (e) => { e.preventDefault(); toggleSteal(); });
        stealBtn.addEventListener('mousedown', (e) => { toggleSteal(); });

        function bindButton(id, start, end) {
            const el = document.getElementById(id);
            if(el) {
                el.addEventListener('touchstart', (e) => { e.preventDefault(); start(); });
                el.addEventListener('touchend', (e) => { e.preventDefault(); end(); });
            }
        }

        function fireBullet() {
            if (gameOver || isDriving) return; // لا يمكن ضرب النار وأنت جوه العربية
            bullets.push({
                x: player.x + player.width / 2 - 3,
                y: player.y,
                width: 6,
                height: 15,
                speed: 13
            });
        }

        // دالة سرقة أو النزول من العربية
        function toggleSteal() {
            if (gameOver) return;

            if (isDriving) {
                // النزول من العربية
                isDriving = false;
                player.y += 20; // النزول بأمان بجانب السيارة
                document.getElementById('carStatus').innerText = "الحالة: على رجلك 🏃‍♂️";
                stealBtn.innerText = "اسرق العربية 🚗";
                stealBtn.style.display = "none";
            } else if (closeCarIndex !== -1 && obstacles[closeCarIndex]) {
                // سرقة العربية القريبة
                isDriving = true;
                const car = obstacles[closeCarIndex];
                stolenCar.color = car.color;
                stolenCar.width = car.width;
                stolenCar.height = car.height;
                stolenCar.isPolice = car.isPolice;
                
                // حذف العربية من الشارع لأن الأسطورة ركبها
                obstacles.splice(closeCarIndex, 1);
                closeCarIndex = -1;

                document.getElementById('carStatus').innerText = stolenCar.isPolice ? "الحالة: راكب شرطة 🚔" : "الحالة: سايق ميكروباص 🚗";
                stealBtn.innerText = "انزل منها 🏃‍♂️";
            }
        }

        function addObstacle() {
            if (Math.random() < 0.03 && obstacles.length < 5) {
                const isTukTuk = Math.random() > 0.5;
                obstacles.push({
                    x: Math.random() * (canvas.width - 50),
                    y: -90,
                    width: isTukTuk ? 42 : 54,
                    height: isTukTuk ? 55 : 85,
                    speed: 3.5 + Math.random() * 3,
                    color: isTukTuk ? '#e67e22' : '#2980b9',
                    isPolice: Math.random() < 0.25
                });
            }
        }

        function addCoin() {
            if (Math.random() < 0.02 && coins.length < 3) {
                coins.push({
                    x: Math.random() * (canvas.width - 25),
                    y: -30,
                    width: 25,
                    height: 25,
                    speed: 3.5
                });
            }
        }

        function update() {
            if (gameOver) return;

            // سرعة الحركة (تزيد لو راكب عربية)
            let currentSpeed = isDriving ? player.speed * 1.3 : player.speed;

            if (moveUp && player.y > 50) player.y -= currentSpeed;
            if (moveDown && player.y < canvas.height - 210) player.y += currentSpeed;
            if (moveLeft && player.x > 0) player.x -= currentSpeed;
            if (moveRight && player.x < canvas.width - player.width) player.x += currentSpeed;

            // تحديث الرصاص
            for (let b = bullets.length - 1; b >= 0; b--) {
                bullets[b].y -= bullets[b].speed;
                for (let i = obstacles.length - 1; i >= 0; i--) {
                    if (hitTest(bullets[b], obstacles[i])) {
                        score += obstacles[i].isPolice ? 500 : 300;
                        document.getElementById('score').innerText = score;
                        obstacles.splice(i, 1);
                        bullets.splice(b, 1);
                        break;
                    }
                }
                if (bullets[b] && bullets[b].y < -20) bullets.splice(b, 1);
            }

            // فحص العربيات القريبة للسرقة
            closeCarIndex = -1;
            if (!isDriving) {
                for (let i = 0; i < obstacles.length; i++) {
                    // لو المسافة قريبة جداً هندسياً
                    let dist = Math.abs(player.x - obstacles[i].x) + Math.abs(player.y - obstacles[i].y);
                    if (dist < 120) {
                        closeCarIndex = i;
                        break;
                    }
                }
                // إظهار أو إخفاء زرار السرقة بناء على القرب
                stealBtn.style.display = (closeCarIndex !== -1) ? "flex" : "none";
            } else {
                stealBtn.style.display = "flex"; // يفضل ظاهر طول ما أنت سايق عشان تنزل
            }

            // تحديث السيارات في الشارع والاصطدام
            for (let i = obstacles.length - 1; i >= 0; i--) {
                obstacles[i].y += obstacles[i].speed;
                
                if (hitTest(player, obstacles[i])) {
                    if (isDriving) {
                        // لو سايق وخبطت عربية تانية، بتدمرها وتاخد فلوس! (تفجير بـ جاتا)
                        score += 150;
                        document.getElementById('score').innerText = score;
                        health -= 5; // دمج بسيط للعربية اللي أنت راكبها
                        obstacles.splice(i, 1);
                    } else {
                        // لو ماشي على رجلك وخبطتك عربية تدمجك جامد
                        health -= obstacles[i].isPolice ? 25 : 15;
                        obstacles.splice(i, 1);
                    }

                    document.getElementById('health').innerText = health + "%";
                    if (health <= 0) {
                        gameOver = true;
                        alert("الأسطورة خسر المواجهة! مجموع الفلوس: " + score + " ج.م");
                        location.reload();
                    }
                    continue;
                }
                if (obstacles[i].y > canvas.height) obstacles.splice(i, 1);
            }

            // تجميع الفلوس
            for (let i = coins.length - 1; i >= 0; i--) {
                coins[i].y += coins[i].speed;
                if (hitTest(player, coins[i])) {
                    score += 200;
                    document.getElementById('score').innerText = score;
                    coins.splice(i, 1);
                    continue;
                }
                if (coins[i].y > canvas.height) coins.splice(i, 1);
            }

            addObstacle();
            addCoin();
        }

        function hitTest(r1, r2) {
            if(!r1 || !r2) return false;
            return r1.x < r2.x + r2.width && r1.x + r1.width > r2.x &&
                   r1.y < r2.y + r2.height && r1.y + r1.height > r2.y;
        }

        function draw() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // خط الشارع الإسفلتي
            ctx.strokeStyle = '#fff';
            ctx.setLineDash([30, 30]);
            ctx.lineWidth = 6;
            ctx.beginPath();
            ctx.moveTo(canvas.width / 2, 0);
            ctx.lineTo(canvas.width / 2, canvas.height);
            ctx.stroke();

            // الفلوس ج.م
            coins.forEach(c => {
                ctx.fillStyle = '#2ecc71';
                ctx.fillRect(c.x, c.y, c.width, c.height);
                ctx.fillStyle = '#fff';
                ctx.font = 'bold 12px Arial';
                ctx.fillText('ج.م', c.x + 3, c.y + 17);
            });

            // الرصاص
            bullets.forEach(b => {
                ctx.fillStyle = '#f1c40f';
                ctx.fillRect(b.x, b.y, b.width, b.height);
            });

            // رسم العربيات اللي في الشارع
            obstacles.forEach(o => {
                drawCar(o.x, o.y, o.width, o.height, o.color, o.isPolice);
            });

            // ─── رسم اللاعب أو السيارة المخطوفة ───
            if (isDriving) {
                // لو سايق، بنرسم شكل العربية اللي سرقها مكان اللاعب علطول
                drawCar(player.x, player.y, player.width + 6, player.height + 20, stolenCar.color, stolenCar.isPolice);
            } else {
                // لو ماشي على رجله، بنرسم محمد رمضان الكرتوني الفخم بالنظارة والسكسوكة
                let centerX = player.x + player.width / 2;
                
                ctx.fillStyle = '#ffffff'; 
                ctx.fillRect(player.x, player.y + 18, player.width, player.height - 18);
                
                ctx.fillStyle = '#111111';
                ctx.beginPath();
                ctx.moveTo(centerX - 6, player.y + 18);
                ctx.lineTo(centerX + 6, player.y + 18);
                ctx.lineTo(centerX, player.y + 28);
                ctx.closePath();
                ctx.fill();

                ctx.fillStyle = '#dca37a'; 
                ctx.beginPath();
                ctx.arc(centerX, player.y + 12, 11, 0, Math.PI * 2);
                ctx.fill();

                ctx.fillStyle = '#1a1a1a';
                ctx.beginPath();
                ctx.arc(centerX, player.y + 14, 11, 0, Math.PI, false);
                ctx.lineTo(centerX - 11, player.y + 14);
                ctx.fill();
                
                ctx.fillStyle = '#dca37a';
                ctx.beginPath();
                ctx.arc(centerX, player.y + 12, 9, 0, Math.PI, false);
                ctx.fill();

                ctx.fillStyle = '#000000';
                ctx.fillRect(centerX - 9, player.y + 7, 7, 5);
                ctx.fillRect(centerX + 2, player.y + 7, 7, 5);
                ctx.fillRect(centerX - 3, player.y + 8, 6, 2);
            }
        }

        // دالة موحدة لرسم العربيات والشرطة بدقة
        function drawCar(x, y, width, height, color, isPolice) {
            if(isPolice) {
                ctx.fillStyle = '#111111';
                ctx.fillRect(x, y, width, height);
                ctx.fillStyle = '#ffffff';
                ctx.fillRect(x + width/4, y + 10, width/2, height - 20);
                ctx.fillStyle = (Math.floor(Date.now() / 150) % 2 === 0) ? '#e74c3c' : '#3498db';
                ctx.fillRect(x + 4, y + height - 8, width - 8, 6);
            } else {
                ctx.fillStyle = color;
                ctx.fillRect(x, y, width, height);
            }
            ctx.fillStyle = '#e0e0e0';
            ctx.fillRect(x + 4, y + height - 18, width - 8, 12);
        }

        function gameLoop() {
            update();
            draw();
            gameInterval = requestAnimationFrame(gameLoop);
        }

        window.addEventListener('load', initGame);
        setTimeout(initGame, 300);
    </script>
</body>
</html>
<img width="64" height="75" alt="Screenshot_2026-06-24-00-09-11-99_b5a5c5cb02ca09c784c5d88160e2ec24" src="https://github.com/user-attachments/assets/66f8fd0a-44ba-426b-acf3-0a58ad2d5cba" />
<img width="64" height="75" alt="Screenshot_2026-06-24-00-09-11-99_b5a5c5cb02ca09c784c5d88160e2ec24" src="https://github.com/user-attachments/assets/bd44a784-fb66-4b36-a995-99367f3fe108" />
<img width="720" height="1341" alt="Screenshot_2026-06-24-01-33-55-60_40deb401b9ffe8e1df2f1cc5ba480b12" src="https://github.com/user-attachments/assets/b70e0781-c2ea-4d9b-9b06-be81e3f2acfa" />
