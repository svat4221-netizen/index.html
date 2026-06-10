<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Мой Воксельный Мир с Модами</title>
    <style>
        body { margin: 0; overflow: hidden; font-family: sans-serif; }
        #ui {
            position: absolute;
            top: 10px;
            left: 10px;
            background: rgba(0,0,0,0.7);
            color: white;
            padding: 10px;
            border-radius: 5px;
            pointer-events: none;
        }
    </style>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
</head>
<body>

    <div id="ui">
        <h3>Управление:</h3>
        <p>W, A, S, D — Движение персонажа</p>
        <p>Мышь — Вращение камеры вокруг игрока</p>
        <p>Вид: От третьего лица</p>
        <div id="mod-status">Загрузка модов...</div>
    </div>

    <script>
        // --- 1. НАСТРОЙКА СЦЕНЫ И ДВИЖКА ---
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x87CEEB); // Голубое небо

        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.body.appendChild(renderer.domElement);

        // Освещение
        const light = new THREE.DirectionalLight(0xffffff, 1);
        light.position.set(5, 10, 7).normalize();
        scene.add(light);
        scene.add(new THREE.AmbientLight(0x404040)); // Мягкий общий свет

        // --- 2. СИСТЕМА МОДОВ (РЕГИСТР БЛОКОВ) ---
        // Базовая игра знает только про траву и землю.
        const BlockRegistry = {
            'core:grass': { color: 0x559933 },
            'core:dirt': { color: 0x866043 }
        };

        // Функция для симуляции загрузки мода
        function loadMod(modJson) {
            try {
                const mod = JSON.parse(modJson);
                BlockRegistry[mod.id] = { color: parseInt(mod.color, 16) };
                document.getElementById('mod-status').innerText = `Мод успешно загружен: добавлено ${mod.id}`;
            } catch (e) {
                console.error("Ошибка загрузки мода:", e);
            }
        }

        // Пример твоего собственного мода (добавляем Рубиновую руду)
        const myRubyMod = `{
            "id": "my_mod:ruby_ore",
            "color": "0xE0115F"
        }`;
        
        // Активируем мод
        loadMod(myRubyMod);


        // --- 3. ГЕНЕРАЦИЯ МИРА (ЧАНК) ---
        const geometry = new THREE.BoxGeometry(1, 1, 1);
        const worldSize = 16;

        for (let x = 0; x < worldSize; x++) {
            for (let z = 0; z < worldSize; z++) {
                // Базовый слой земли
                const dirtMaterial = new THREE.MeshLambertMaterial({ color: BlockRegistry['core:dirt'].color });
                const dirtMesh = new THREE.Mesh(geometry, dirtMaterial);
                dirtMesh.position.set(x, 0, z);
                scene.add(dirtMesh);

                // Верхний слой (трава или руда из мода)
                // Случайно спавним рубин из нашего мода вместо травы с шансом 5%
                const isRuby = Math.random() < 0.05;
                const blockType = isRuby ? 'my_mod:ruby_ore' : 'core:grass';

                const topMaterial = new THREE.MeshLambertMaterial({ color: BlockRegistry[blockType].color });
                const topMesh = new THREE.Mesh(geometry, topMaterial);
                topMesh.position.set(x, 1, z);
                scene.add(topMesh);
            }
        }


        // --- 4. ПЕРСОНАЖ И КАМЕРА ОТ ТРЕТЬЕГО ЛИЦА ---
        // Создаем игрока (вместо модельки пока побудет красный цилиндр/капсула)
        const playerGeometry = new THREE.CylinderGeometry(0.4, 0.4, 1.8, 16);
        const playerMaterial = new THREE.MeshLambertMaterial({ color: 0xCC0000 });
        const player = new THREE.Mesh(playerGeometry, playerMaterial);
        player.position.set(8, 2.9, 8); // Спавним в центре карты сверху
        scene.add(player);

        // Настройки камеры от третьего лица
        let cameraDistance = 5;
        let cameraPitch = 0.3; // Угол наклона вверх/вниз
        let cameraYaw = 0;     // Поворот влево/вправо

        // Управление клавиатурой
        const keys = { w: false, a: false, s: false, d: false };
        window.addEventListener('keydown', (e) => { if(keys.hasOwnProperty(e.key.toLowerCase())) keys[e.key.toLowerCase()] = true; });
        window.addEventListener('keyup', (e) => { if(keys.hasOwnProperty(e.key.toLowerCase())) keys[e.key.toLowerCase()] = false; });

        // Управление мышью (вращение камеры)
        window.addEventListener('mousemove', (e) => {
            if (document.pointerLockElement === document.body) {
                cameraYaw -= e.movementX * 0.003;
                cameraPitch -= e.movementY * 0.003;
                // Ограничиваем наклон камеры, чтобы не перевернуться
                cameraPitch = Math.max(-0.5, Math.min(0.8, cameraPitch));
            }
        });

        // Блокировка курсора при клике на экран
        window.addEventListener('click', () => {
            document.body.requestPointerLock();
        });


        // --- 5. ИГРОВОЙ ЦИКЛ (ОБНОВЛЕНИЕ КАДРОВ) ---
        const clock = new THREE.Clock();
        const speed = 5;

        function animate() {
            requestAnimationFrame(animate);
            const delta = clock.getDelta();

            // 1. Движение персонажа относительно направления камеры
            const moveVector = new THREE.Vector3(0, 0, 0);
            if (keys.w) moveVector.z -= 1;
            if (keys.s) moveVector.z += 1;
            if (keys.a) moveVector.x -= 1;
            if (keys.d) moveVector.x += 1;
            moveVector.normalize();

            // Разворачиваем вектор движения туда, куда смотрит камера
            const forward = new THREE.Vector3(0, 0, -1).applyAxisAngle(new THREE.Vector3(0, 1, 0), cameraYaw);
            const side = new THREE.Vector3(1, 0, 0).applyAxisAngle(new THREE.Vector3(0, 1, 0), cameraYaw);
            
            const finalMovement = new THREE.Vector3()
                .addScaledVector(forward, -moveVector.z)
                .addScaledVector(side, moveVector.x)
                .normalize()
                .multiplyScalar(speed * delta);

            player.position.add(finalMovement);

            // Поворачиваем игрока лицом в сторону движения (если он двигается)
            if (moveVector.lengthSq() > 0) {
                player.rotation.y = cameraYaw + Math.atan2(moveVector.x, moveVector.z);
            }

            // 2. Логика камеры от третьего лица
            // Вычисляем позицию камеры на сфере вокруг игрока
            const targetCameraPosition = new THREE.Vector3(
                player.position.x + cameraDistance * Math.sin(cameraYaw) * Math.cos(cameraPitch),
                player.position.y + cameraDistance * Math.sin(cameraPitch) + 1, // +1 чтобы смотреть чуть выше плеча
                player.position.z + cameraDistance * Math.cos(cameraYaw) * Math.cos(cameraPitch)
            );

            camera.position.copy(targetCameraPosition);
            // Камера всегда смотрит на персонажа
            camera.lookAt(player.position.x, player.position.y + 0.5, player.position.z);

            renderer.render(scene, camera);
        }

        // Поддержка изменения размеров окна
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });

        // Запуск игры
        animate();
    </script>
</body>
</html>

