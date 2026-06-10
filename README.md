<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Minecraft HTML5 Edition</title>
    <style>
        body { margin: 0; overflow: hidden; font-family: 'Courier New', monospace; user-select: none; background: #000; }
        #canvas { display: block; width: 100vw; height: 100vh; }
        
        /* Интерфейс (UI) */
        #ui-container { position: absolute; top: 0; left: 0; width: 100%; height: 100%; pointer-events: none; }
        .panel { background: rgba(0,0,0,0.6); color: #fff; padding: 15px; border: 2px solid #555; position: absolute; pointer-events: auto; }
        
        #controls { top: 10px; left: 10px; font-size: 12px; line-height: 1.5; width: 280px; }
        #stats { top: 10px; right: 10px; text-align: right; }
        
        /* Шкала Энергии */
        .energy-bar-container { width: 150px; height: 15px; background: #333; border: 2px solid #fff; margin-top: 5px; display: inline-block;}
        #energy-bar { width: 100%; height: 100%; background: #ffaa00; transition: width 0.1s; }
        
        /* Хотбар предметов */
        #hotbar { bottom: 20px; left: 50%; transform: translateX(-50%); display: flex; background: rgba(0,0,0,0.8); border: 3px solid #444; padding: 4px; }
        .slot { width: 50px; height: 50px; border: 2px solid #555; margin: 0 4px; display: flex; align-items: center; justify-content: center; font-size: 10px; text-align: center; color: white; font-weight: bold;}
        .slot.active { border-color: #fff; background: rgba(255,255,255,0.2); }
        
        /* Прицел для 1-го лица */
        #crosshair { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); color: white; font-size: 20px; font-weight: bold; display: none; }
        
        /* Цвета ролей */
        .role-Игрок { color: #aaa; }
        .role-Модератор { color: #55ff55; }
        .role-Администратор { color: #ff5555; }
        .role-Куратор { color: #aa00aa; font-weight: bold; }
    </style>
</head>
<body>

    <canvas id="canvas"></canvas>

    <div id="ui-container">
        <div id="crosshair">+</div>

        <div class="panel" id="controls">
            <h3 style="margin:0 0 10px 0; color:#55ff55;">MINECRAFT JS</h3>
            <b>W, A, S, D</b> — Движение<br>
            <b>Мышь</b> — Обзор (Кликните на экран)<br>
            <b>Кнопка V</b> — Смена вида (1-е / 3-е лицо)<br>
            <b>Цифры 1-5</b> — Выбор предмета<br>
            <b>ЛКМ</b> — Сломать блок<br>
            <b>ПКМ</b> — Поставить блок<br>
            <hr style="border-color:#444;">
            <div id="view-mode-text">Вид: От 3-го лица</div>
        </div>

        <div class="panel" id="stats">
            <div>Ваш статус: <span class="role-Куратор">[Куратор] Разработчик</span></div>
            <div>Энергия: <div class="energy-bar-container"><div id="energy-bar"></div></div></div>
            <h4 style="margin: 10px 0 5px 0; text-align:center;">Игроки онлайн (Sim):</h4>
            <div id="player-list" style="font-size:11px; text-align:left;"></div>
        </div>

        <div id="hotbar">
            <div class="slot active" id="slot-1" style="background:#559933;">Трава<br>(1)</div>
            <div class="slot" id="slot-2" style="background:#866043;">Земля<br>(2)</div>
            <div class="slot" id="slot-3" style="background:#777777;">Камень<br>(3)</div>
            <div class="slot" id="slot-4" style="background:#44ddff; color:#000;">Алмаз<br>(4)</div>
            <div class="slot" id="slot-5" style="background:#ffdd44; color:#000;">Золото<br>(5)</div>
        </div>
    </div>

    <script>
        // --- 1. ИНИЦИАЛИЗАЦИЯ И НАСТРОЙКА ДВИЖКА ---
        const canvas = document.getElementById('canvas');
        const ctx = canvas.getContext('2d');

        function resizeCanvas() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
        }
        window.addEventListener('resize', resizeCanvas);
        resizeCanvas();

        // Игровые параметры
        let isFirstPerson = false; // По умолчанию 3-е лицо
        let energy = 100;
        let activeSlot = 1;
        
        // Список доступных блоков из Майнкрафта
        const BLOCKS = {
            1: { name: 'Трава', color: '#559933', topColor: '#77cc44' },
            2: { name: 'Земля', color: '#866043', topColor: '#866043' },
            3: { name: 'Камень', color: '#777777', topColor: '#888888' },
            4: { name: 'Алмазная руда', color: '#557788', topColor: '#44ddff' },
            5: { name: 'Золотая руда', color: '#666655', topColor: '#ffdd44' }
        };

        // --- 2. ИГРОК, КАМЕРА И СЕТЬ ---
        let player = {
            x: 400, z: 400, y: 150,
            yaw: 0, pitch: 0.4,
            speed: 4,
            radius: 15, height: 40
        };

        // Симуляция онлайн игроков с ролями
        let onlinePlayers = [
            { id: 1, name: 'Stevka_Pro', role: 'Игрок', x: 350, z: 450, color: '#3366cc' },
            { id: 2, name: 'Admin_Vanya', role: 'Администратор', x: 500, z: 380, color: '#cc3333' },
            { id: 3, name: 'Moder_Dima', role: 'Модератор', x: 420, z: 520, color: '#33cc33' }
        ];

        // Обновление таб-листа игроков на экране
        function updateTabList() {
            let html = `<div><span class="role-Куратор">[Куратор]</span> Вы</div>`;
            onlinePlayers.forEach(p => {
                html += `<div><span class="role-${p.role}">[${p.role}]</span> ${p.name}</div>`;
            });
            document.getElementById('player-list').innerHTML = html;
        }
        updateTabList();

        // --- 3. ГЕНЕРАЦИЯ МИРА (ПСЕВДО-3D МАТРИЦА БЛОКОВ) ---
        let world = [];
        const worldSize = 16;
        const blockSize = 40;

        // Создаем рельеф Майнкрафта
        for(let x=0; x<worldSize; x++) {
            world[x] = [];
            for(let z=0; z<worldSize; z++) {
                // Изменяем высоту для имитации холмов
                let height = 2 + Math.floor(Math.sin(x*0.5) * 1.5 + Math.cos(z*0.5) * 1.5);
                world[x][z] = [];
                for(let y=0; y<6; y++) {
                    let blockType = 0; // Воздух
                    if (y < height - 1) {
                        // Глубокие слои: камень или случайная руда
                        let rand = Math.random();
                        if (rand < 0.05) blockType = 4; // Алмаз
                        else if (rand < 0.12) blockType = 5; // Золото
                        else blockType = 3; // Камень
                    } else if (y < height) {
                        blockType = 2; // Земля
                    } else if (y === height) {
                        blockType = 1; // Трава
                    }
                    world[x][z][y] = blockType;
                }
            }
        }

        // --- 4. СИСТЕМА УПРАВЛЕНИЯ КЛАВИАТУРОЙ И МЫШЬЮ ---
        let keys = {};
        window.addEventListener('keydown', (e) => {
            keys[e.key.toLowerCase()] = true;
            // Смена вида (кнопка V)
            if(e.key.toLowerCase() === 'v') {
                isFirstPerson = !isFirstPerson;
                document.getElementById('view-mode-text').innerText = isFirstPerson ? "Вид: От 1-го лица" : "Вид: От 3-го лица";
                document.getElementById('crosshair').style.display = isFirstPerson ? "block" : "none";
            }
            // Хотбар предметов (1-5)
            if(e.key >= '1' && e.key <= '5') {
                document.getElementById(`slot-${activeSlot}`).classList.remove('active');
                activeSlot = parseInt(e.key);
                document.getElementById(`slot-${activeSlot}`).classList.add('active');
            }
        });
        window.addEventListener('keyup', (e) => { keys[e.key.toLowerCase()] = false; });

        // Захват мышки
        let mouseLocked = false;
        canvas.addEventListener('click', () => {
            canvas.requestPointerLock();
        });
        document.addEventListener('pointerlockchange', () => {
            mouseLocked = document.pointerLockElement === canvas;
        });

        // Движение мыши (вращение камеры вокруг игрока)
        window.addEventListener('mousemove', (e) => {
            if (mouseLocked) {
                player.yaw += e.movementX * 0.005;
                player.pitch += e.movementY * 0.005;
                player.pitch = Math.max(-0.5, Math.min(1.2, player.pitch)); // Ограничение наклона
            }
        });

        // Клик мыши: Действия с блоками и трата ЭНЕРГИИ
        window.addEventListener('mousedown', (e) => {
            if (!mouseLocked) return;

            if (energy < 15) {
                alert("Недостаточно энергии! Отдохните (подождите пару секунд).");
                return;
            }

            // Находим блок перед игроком
            let targetX = Math.floor((player.x + Math.sin(player.yaw)*60) / blockSize);
            let targetZ = Math.floor((player.z - Math.cos(player.yaw)*60) / blockSize);
            
            if(targetX >= 0 && targetX < worldSize && targetZ >= 0 && targetZ < worldSize) {
                let chunk = world[targetX][targetZ];
                let topY = chunk.length - 1;
                while(topY >= 0 && chunk[topY] === 0) topY--;

                if (e.button === 0) { // ЛКМ — Ломаем верхний блок
                    if(topY >= 0) {
                        chunk[topY] = 0;
                        energy -= 15; // Тратим энергию
                    }
                } else if (e.button === 2) { // ПКМ — Ставим выбранный предмет
                    if(topY < 5) {
                        chunk[topY + 1] = activeSlot;
                        energy -= 10; // Тратим энергию
                    }
                }
            }
        });
        // Отключаем стандартное меню по ПКМ
        window.addEventListener('contextmenu', e => e.preventDefault());


        // --- 5. ОТРИСОВКА ИЗОМЕТРИЧЕСКОГО/3D МИРА ---
        function drawCube(x, y, z, type) {
            let block = BLOCKS[type];
            if (!block) return;

            // Проекция 3D координат на 2D экран относительно камеры игрока
            let dx = x * blockSize - player.x;
            let dz = z * blockSize - player.z;
            let dy = y * (blockSize * 0.7) - player.y;

            // Вращение сцены вокруг камеры
            let cos = Math.cos(-player.yaw);
            let sin = Math.sin(-player.yaw);
            let rx = dx * cos - dz * sin;
            let rz = dx * sin + dz * cos;

            // Если блок сзади нас — не рисуем его
            if (rz < 10) return;

            // Перспектива
            let fov = 400 / (rz * player.pitch);
            let screenX = canvas.width / 2 + rx * fov;
            let screenY = canvas.height / 2 - dy * fov;
            let size = blockSize * fov;

            if(isFirstPerson) {
                // Корректировка высоты для 1-го лица
                screenY += 120;
            }

            // Рисуем переднюю грань куба
            ctx.fillStyle = block.color;
            ctx.fillRect(screenX - size/2, screenY - size/2, size, size);
            ctx.strokeStyle = 'rgba(0,0,0,0.2)';
            ctx.strokeRect(screenX - size/2, screenY - size/2, size, size);

            // Рисуем верхнюю грань (крышку блока)
            ctx.fillStyle = block.topColor;
            ctx.beginPath();
            ctx.moveTo(screenX - size/2, screenY - size/2);
            ctx.lineTo(screenX, screenY - size*0.8);
            ctx.lineTo(screenX + size/2, screenY - size/2);
            ctx.fill();
        }

        // Отрисовка других онлайн игроков
        function drawRemotePlayer(p) {
            let dx = p.x - player.x;
            let dz = p.z - player.z;
            let dy = 0 - player.y + 40;

            let cos = Math.cos(-player.yaw);
            let sin = Math.sin(-player.yaw);
            let rx = dx * cos - dz * sin;
            let rz = dx * sin + dz * cos;

            if (rz < 10) return;

            let fov = 400 / (rz * player.pitch);
            let screenX = canvas.width / 2 + rx * fov;
            let screenY = canvas.height / 2 - dy * fov;
            let size = player.radius * 2 * fov;

            // Моделька игрока (капсула)
            ctx.fillStyle = p.color;
            ctx.beginPath();
            ctx.arc(screenX, screenY, size, 0, Math.PI*2);
            ctx.fill();

            // Никнейм над головой игрока
            ctx.fillStyle = "white";
            ctx.font = "12px Arial";
            ctx.textAlign = "center";
            ctx.fillText(`[${p.role}] ${p.name}`, screenX, screenY - size - 5);
        }

        // --- 6. ОСНОВНОЙ КУРСОР/ЦИКЛ ИГРЫ ---
        let lastTime = performance.now();

        function gameLoop() {
            let now = performance.now();
            let dt = (now - lastTime) / 1000;
            lastTime = now;

            // 1. Восстановление энергии со временем
            if (energy < 100) {
                energy = Math.min(100, energy + dt * 12); // +12 единиц в секунду
            }
            document.getElementById('energy-bar').style.width = energy + '%';

            // 2. Движение игрока
            let moveX = 0;
            let moveZ = 0;
            if (keys['w']) { moveX += Math.sin(player.yaw); moveZ -= Math.cos(player.yaw); }
            if (keys['s']) { moveX -= Math.sin(player.yaw); moveZ += Math.cos(player.yaw); }
            if (keys['a']) { moveX -= Math.cos(player.yaw); moveZ -= Math.sin(player.yaw); }
            if (keys['d']) { moveX += Math.cos(player.yaw); moveZ += Math.sin(player.yaw); }

            player.x += moveX * player.speed;
            player.z += moveZ * player.speed;

            // Симуляция движения других онлайн-игроков (чтобы мир казался живым)
            onlinePlayers.forEach(p => {
                p.x += Math.sin(now * 0.001 + p.id) * 0.5;
                p.z += Math.cos(now * 0.001 * p.id) * 0.5;
            });

            // 3. РЕНДЕРИНГ СЦЕНЫ
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // Небо
            ctx.fillStyle = '#87CEEB';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            // Отрисовка блоков (сзади наперед для корректного наложения слоев)
            for (let y = 0; y < 6; y++) {
                for (let x = 0; x < worldSize; x++) {
                    for (let z = 0; z < worldSize; z++) {
                        let type = world[x][z][y];
                        if (type !== 0) {
                            drawCube(x, y, z, type);
                        }
                    }
                }
            }

            // Отрисовка других игроков в онлайне
            onlinePlayers.forEach(p => drawRemotePlayer(p));

            // Отрисовка собственного персонажа (Только если вид от 3-го лица!)
            if (!isFirstPerson) {
                ctx.fillStyle = '#ff0000'; // Наш скин — Красный
                ctx.beginPath();
                // По центру экрана, так как камера привязана к нам
                let screenY = canvas.height / 2 + 50;
                ctx.arc(canvas.width / 2, screenY, 20, 0, Math.PI * 2);
                ctx.fill();
                
                ctx.fillStyle = "white";
                ctx.font = "12px Arial";
                ctx.textAlign = "center";
                ctx.fillText("[Куратор] Разработчик (Вы)", canvas.width / 2, screenY - 25);
            }

            requestAnimationFrame(gameLoop);
        }

        // Старт игры
        gameLoop();
    </script>
</body>
</html>

