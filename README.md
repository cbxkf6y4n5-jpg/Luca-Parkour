# Luca-Parkour
Runner 
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>128-Bit Runner</title>
    <!-- Подключаем Tailwind CSS для стилизации контейнера и UI -->
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://fonts.googleapis.com/css2?family=VT323&display=swap" rel="stylesheet">
    <style>
        /* Стиль для эмуляции ретро-графики */
        body {
            font-family: 'VT323', monospace;
            background-color: #2c3e50; /* Темный фон */
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
        }
        #game-container {
            background-color: #34495e;
            box-shadow: 0 10px 20px rgba(0, 0, 0, 0.5);
            border: 8px solid #f39c12; /* Золотая рамка */
            border-radius: 12px;
            padding: 1rem;
            max-width: 95vw;
            width: 500px;
        }
        #game-canvas {
            display: block;
            background-color: #ecf0f1; /* Светлый фон неба/земли */
            border: 4px solid #1abc9c; /* Зеленая граница */
            border-radius: 4px;
            touch-action: manipulation; /* Улучшение обработки касаний */
        }
        .info-box {
            background-color: #2c3e50;
            color: #ecf0f1;
            padding: 0.5rem;
            border-radius: 6px;
            font-size: 1.25rem;
        }
        .pixel-button {
            background-color: #e74c3c; /* Красный для рестарта */
            color: white;
            padding: 10px 20px;
            border: 4px solid #c0392b;
            border-radius: 8px;
            box-shadow: 4px 4px 0 #922b21;
            transition: all 0.1s;
        }
        .pixel-button:active {
            box-shadow: 0 0 0 #922b21;
            transform: translate(4px, 4px);
        }
    </style>
</head>
<body class="p-4">

    <div id="game-container" class="flex flex-col items-center gap-4">
        <h1 class="text-3xl font-bold text-white text-center">Побег Светловолосого Мальчика</h1>

        <!-- Информационная панель -->
        <div class="flex justify-between w-full gap-2 text-center">
            <div id="score-display" class="info-box flex-1">Счет: 0</div>
            <div id="coins-display" class="info-box flex-1">Монеты: 0</div>
            <div id="hits-display" class="info-box flex-1">Столкновения: 1/1</div>
        </div>
        
        <!-- Canvas для игры -->
        <canvas id="game-canvas" width="480" height="240"></canvas>

        <!-- Сообщение о конце игры -->
        <div id="game-over-message" class="hidden absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 bg-black bg-opacity-80 p-6 rounded-lg text-white text-center z-10 w-4/5 max-w-sm">
            <h2 class="text-4xl font-bold mb-4">ИГРА ОКОНЧЕНА</h2>
            <p id="final-score" class="text-2xl mb-4"></p>
            <button id="restart-button" class="pixel-button">Перезапустить (Space/Нажать)</button>
        </div>

        <!-- Инструкции для мобильных -->
        <p class="text-sm text-gray-400 mt-2">
            Используйте <span class="text-yellow-400">Space</span> или <span class="text-yellow-400">тап/свайп вверх</span> для прыжка.
        </p>
    </div>

    <script type="module">
        // Получаем элементы DOM
        const canvas = document.getElementById('game-canvas');
        const ctx = canvas.getContext('2d');
        const scoreDisplay = document.getElementById('score-display');
        const coinsDisplay = document.getElementById('coins-display');
        const hitsDisplay = document.getElementById('hits-display');
        const gameOverMessage = document.getElementById('game-over-message');
        const finalScore = document.getElementById('final-score');
        const restartButton = document.getElementById('restart-button');

        // --- Глобальные настройки игры ---
        const GROUND_Y = canvas.height - 30; // Уровень земли
        const GRAVITY = 0.8; // Гравитация
        const INITIAL_JUMP_VELOCITY = -14; // Начальная скорость прыжка

        // --- Состояние игры ---
        let isRunning = false;
        let gameSpeed = 5;
        let score = 0;
        let coinsCollected = 0;
        let hitsRemaining = 1;
        let lastTime = 0;
        let obstacleTimer = 0;
        let coinTimer = 0;
        let highJumpActive = false;
        let animationFrameId;
        
        // --- Звуки (используем Tone.js для генерации, чтобы избежать внешних ссылок) ---
        // Инициализируем Tone.js, если возможно
        let synth;
        try {
            // Подключаем Tone.js
            const toneScript = document.createElement('script');
            toneScript.src = 'https://cdnjs.cloudflare.com/ajax/libs/tone/14.8.49/Tone.min.js';
            document.head.appendChild(toneScript);
            toneScript.onload = () => {
                if (window.Tone) {
                    synth = new Tone.Synth().toDestination();
                }
            };
        } catch (e) {
            console.error("Tone.js failed to load or initialize:", e);
        }

        /**
         * Воспроизводит простой звук для сбора монеты
         */
        function playCoinSound() {
            if (synth) {
                Tone.start();
                synth.triggerAttackRelease("C5", "8n");
            }
        }

        /**
         * Воспроизводит звук столкновения
         */
        function playHitSound() {
            if (synth) {
                Tone.start();
                const noise = new Tone.NoiseSynth({
                    envelope: { attack: 0.005, decay: 0.1, sustain: 0, release: 0.05 },
                    noise: { type: "white" }
                }).toDestination();
                noise.triggerAttackRelease("4n");
            }
        }

        /**
         * Воспроизводит звук бонуса
         */
        function playBonusSound() {
            if (synth) {
                Tone.start();
                const polySynth = new Tone.PolySynth(Tone.Synth, {
                    oscillator: { type: "triangle" },
                    envelope: { attack: 0.01, decay: 0.1, sustain: 0.2, release: 0.5 }
                }).toDestination();
                polySynth.triggerAttackRelease(["C6", "E6", "G6"], "4n");
            }
        }

        // --- Классы игровых объектов ---

        /** Класс Игрока (Мальчик) */
        class Player {
            constructor() {
                this.width = 20;
                this.height = 40;
                this.x = 50;
                this.y = GROUND_Y - this.height;
                this.velocityY = 0;
                this.isJumping = false;
                this.color = '#3498db'; // Цвет одежды (синий)
                this.hairColor = '#f1c40f'; // Цвет волос (светлый)
                this.isInvincible = false; // Неуязвимость после попадания
                this.invincibilityTimer = 0;
            }

            update() {
                // Применяем гравитацию
                if (this.isJumping) {
                    this.y += this.velocityY;
                    this.velocityY += GRAVITY;
                }

                // Столкновение с землей
                if (this.y + this.height > GROUND_Y) {
                    this.y = GROUND_Y - this.height;
                    this.isJumping = false;
                    this.velocityY = 0;
                }

                // Управление таймером неуязвимости
                if (this.isInvincible) {
                    this.invincibilityTimer--;
                    if (this.invincibilityTimer <= 0) {
                        this.isInvincible = false;
                    }
                }
            }

            draw() {
                // Анимация бега (простая смена кадров для 128-бит)
                const frame = Math.floor(score / 5) % 2; 
                
                // Эффект мигания при неуязвимости
                if (this.isInvincible && this.invincibilityTimer % 6 < 3) return; 

                // --- Рисуем Мальчика (Стилизация "прорисованный, светловолосый") ---
                ctx.save();
                ctx.translate(this.x, this.y);

                // 1. Тело и Ноги (Пиксельные формы)
                ctx.fillStyle = this.color; 
                ctx.fillRect(0, 10, this.width, this.height - 10); // Тело
                ctx.fillStyle = '#2c3e50'; // Цвет обуви
                ctx.fillRect(0, this.height - 5, this.width / 2, 5);
                ctx.fillRect(this.width / 2, this.height - 5, this.width / 2, 5);

                // 2. Голова (Светлые волосы)
                ctx.fillStyle = '#e0b48e'; // Цвет кожи
                ctx.fillRect(5, 0, 10, 10); // Голова
                ctx.fillStyle = this.hairColor; // Светлые волосы
                ctx.fillRect(3, -2, 14, 5); // Прическа
                ctx.fillRect(0, 0, 5, 3); // Челка

                // 3. Руки (Простая анимация)
                ctx.fillStyle = this.color;
                if (!this.isJumping) {
                    // Бег
                    ctx.fillRect(-5, 10, 5, 20); // Задняя рука
                    ctx.fillRect(this.width, 10, 5, 20); // Передняя рука
                } else {
                    // Прыжок
                    ctx.fillRect(-5, 15, 5, 10);
                    ctx.fillRect(this.width, 15, 5, 10);
                }

                ctx.restore();
            }

            jump() {
                if (!this.isJumping) {
                    this.isJumping = true;
                    // Бонус "Выше прыгает"
                    const jumpBoost = highJumpActive ? 1.5 : 1; 
                    this.velocityY = INITIAL_JUMP_VELOCITY * jumpBoost;
                }
            }
        }

        /** Класс Препятствия */
        class Obstacle {
            constructor(type) {
                this.width = type === 'small' ? 15 : 30;
                this.height = type === 'small' ? 20 : 35;
                this.x = canvas.width;
                this.y = GROUND_Y - this.height;
                this.color = '#e74c3c'; // Красный
            }

            update() {
                this.x -= gameSpeed;
            }

            draw() {
                ctx.fillStyle = this.color;
                // Рисуем как пиксельный блок
                ctx.fillRect(this.x, this.y, this.width, this.height);
            }
        }

        /** Класс Монеты */
        class Coin {
            constructor() {
                this.size = 10;
                this.x = canvas.width;
                // Случайная высота для монеты
                this.y = GROUND_Y - 40 - Math.random() * 60; 
                this.color = '#f39c12'; // Золотой
                this.collected = false;
            }

            update() {
                this.x -= gameSpeed;
            }

            draw() {
                if (this.collected) return;
                ctx.fillStyle = this.color;
                // Рисуем монету как желтый квадрат с белой точкой (эффект блеска)
                ctx.fillRect(this.x, this.y, this.size, this.size); 
                ctx.fillStyle = '#f9f9f9';
                ctx.fillRect(this.x + 3, this.y + 3, 4, 4);
            }
        }

        // --- Инициализация объектов ---
        let player;
        let obstacles = [];
        let coins = [];

        /** Инициализация или сброс игры */
        function initGame() {
            player = new Player();
            obstacles = [];
            coins = [];
            isRunning = true;
            score = 0;
            coinsCollected = 0;
            hitsRemaining = 1; // Один раз может столкнуться
            gameSpeed = 5;
            highJumpActive = false;
            gameOverMessage.classList.add('hidden');
            
            // Сброс таймеров для генерации
            obstacleTimer = 0;
            coinTimer = 0;

            updateUI();
            
            if (!animationFrameId) {
                gameLoop(0); // Запускаем цикл
            }
        }
        
        // --- Обновление UI ---
        function updateUI() {
            scoreDisplay.textContent = `Счет: ${Math.floor(score)}`;
            coinsDisplay.textContent = `Монеты: ${coinsCollected}`;
            hitsDisplay.textContent = `Столкновения: ${hitsRemaining}/1 ${highJumpActive ? '(БОНУС)' : ''}`;
            
            // Обновляем цвет, если активен бонус
            if (highJumpActive) {
                hitsDisplay.style.color = '#2ecc71'; // Зеленый цвет
            } else {
                hitsDisplay.style.color = ''; // Стандартный цвет
            }
            
            // Показываем, что игрок получил удар
            if (hitsRemaining === 0 && player.isInvincible) {
                hitsDisplay.style.backgroundColor = '#e74c3c'; // Красный фон
            } else {
                 hitsDisplay.style.backgroundColor = ''; 
            }
        }

        // --- Обработка коллизий ---

        /** * Проверка коллизии AABB (Bounding Box)
         * @param {Object} a - Объект A
         * @param {Object} b - Объект B
         * @returns {boolean} - true, если есть столкновение
         */
        function checkCollision(a, b) {
            return a.x < b.x + b.width &&
                   a.x + a.width > b.x &&
                   a.y < b.y + b.height &&
                   a.y + a.height > b.y;
        }

        /** * Обработка сбора монеты (упрощенная коллизия)
         * @param {Object} a - Игрок
         * @param {Object} b - Монета
         * @returns {boolean} - true, если собрана
         */
        function checkCoinCollision(a, b) {
            // Для монеты используем размер this.size, а не this.width/height
            return a.x < b.x + b.size &&
                   a.x + a.width > b.x &&
                   a.y < b.y + b.size &&
                   a.y + a.height > b.y;
        }


        // --- Основной игровой цикл ---

        function gameLoop(currentTime) {
            if (!isRunning) {
                // Если игра остановлена, не запрашиваем следующий кадр
                cancelAnimationFrame(animationFrameId);
                animationFrameId = null;
                return; 
            }

            // Delta time (для более плавного движения, но в этом стиле можно и без него)
            const deltaTime = currentTime - lastTime;
            lastTime = currentTime;

            // 1. Очистка и отрисовка земли
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.fillStyle = '#1abc9c'; // Зеленая земля
            ctx.fillRect(0, GROUND_Y, canvas.width, 30);

            // 2. Обновление скорости и счета
            score += gameSpeed * 0.01;
            // Скорость увеличивается по мере прохождения (постепенно)
            gameSpeed = Math.min(gameSpeed + 0.001, 15); 

            // 3. Генерация препятствий
            obstacleTimer += gameSpeed;
            const minObstacleInterval = 1000;
            const maxObstacleInterval = 2000;
            
            if (obstacleTimer > maxObstacleInterval / gameSpeed) {
                const type = Math.random() < 0.7 ? 'small' : 'large';
                obstacles.push(new Obstacle(type));
                obstacleTimer = 0;
            }
            
            // 4. Генерация монет
            coinTimer += gameSpeed;
            const coinInterval = 300; 
            if (coinTimer > coinInterval) {
                 if (Math.random() < 0.8) { // 80% шанс создать монету
                    coins.push(new Coin());
                }
                coinTimer = -Math.random() * 50; // Небольшая задержка между монетами
            }

            // 5. Обновление и отрисовка объектов

            // Игрок
            player.update();
            player.draw();

            // Препятствия
            obstacles.forEach((obstacle, index) => {
                obstacle.update();
                obstacle.draw();

                // Проверка коллизии с препятствиями
                if (!player.isInvincible && checkCollision(player, obstacle)) {
                    playHitSound();
                    hitsRemaining--;
                    
                    if (hitsRemaining <= 0) {
                        // КОНЕЦ ИГРЫ
                        isRunning = false;
                        finalScore.textContent = `Ваш финальный счет: ${Math.floor(score)}`;
                        gameOverMessage.classList.remove('hidden');
                    } else {
                        // Первое столкновение - активация неуязвимости и сброс бонуса
                        player.isInvincible = true;
                        player.invincibilityTimer = 180; // Неуязвимость на 3 секунды (60 кадров * 3)
                        
                        // Сброс бонуса (если был)
                        if (highJumpActive) {
                            highJumpActive = false;
                        }
                    }
                }

                // Удаление вышедших за экран
                if (obstacle.x + obstacle.width < 0) {
                    obstacles.splice(index, 1);
                }
            });

            // Монеты
            coins.forEach((coin, index) => {
                if (coin.collected) return;
                
                coin.update();
                coin.draw();

                // Проверка коллизии с монетами
                if (checkCoinCollision(player, coin)) {
                    playCoinSound();
                    coin.collected = true;
                    coinsCollected++;
                    
                    // Удаление монеты
                    coins.splice(index, 1); 

                    // Бонус за 10 монет
                    if (coinsCollected % 10 === 0) {
                        playBonusSound();
                        highJumpActive = true;
                        // Бонус длится 500 кадров (около 8 секунд)
                        setTimeout(() => {
                             highJumpActive = false;
                             updateUI();
                        }, 8000); 
                    }
                }

                // Удаление вышедших за экран
                if (coin.x + coin.size < 0) {
                    coins.splice(index, 1);
                }
            });


            // 6. Обновление UI
            updateUI();

            // Запрашиваем следующий кадр анимации
            animationFrameId = requestAnimationFrame(gameLoop);
        }

        // --- Управление ---

        /** Обработчик прыжка */
        function handleJump(event) {
            // Предотвращаем дефолтное поведение (прокрутка)
            event.preventDefault(); 
            if (isRunning) {
                player.jump();
            } else if (!gameOverMessage.classList.contains('hidden')) {
                // Если игра окончена, инициируем перезапуск
                initGame();
            }
        }

        // Клавиатура (Space)
        document.addEventListener('keydown', (e) => {
            if (e.code === 'Space') {
                handleJump(e);
            }
        });
        
        // Кнопка перезапуска
        restartButton.addEventListener('click', initGame);

        // --- Управление касанием для мобильных ---
        let touchstartY = 0;
        
        // Начинаем касание (запоминаем начальную Y-координату)
        canvas.addEventListener('touchstart', (e) => {
            e.preventDefault();
            touchstartY = e.touches[0].clientY;
        }, { passive: false });

        // Конец касания (проверяем, был ли это свайп вверх)
        canvas.addEventListener('touchend', (e) => {
            // Если игра окончена, просто тапаем для перезапуска
            if (!isRunning) {
                initGame();
                return;
            }
            
            // Если это был быстрый тап (нет свайпа), прыгаем
            if (Math.abs(touchstartY - e.changedTouches[0].clientY) < 20) {
                 player.jump();
            }
        }, { passive: true });

        // Движение касания (проверяем, был ли это свайп вверх)
        canvas.addEventListener('touchmove', (e) => {
            const touchmoveY = e.touches[0].clientY;
            // Если свайп вверх (Y уменьшается)
            if (touchstartY - touchmoveY > 30) { 
                player.jump();
                // Сбрасываем touchstartY, чтобы избежать повторных прыжков во время одного свайпа
                touchstartY = Infinity; 
            }
        }, { passive: true });


        // --- Запуск игры при загрузке ---
        window.onload = initGame;

    </script>
</body>
</html>

