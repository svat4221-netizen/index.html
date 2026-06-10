<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Воксельный Онлайн Мир</title>
    <style>
        body { margin: 0; overflow: hidden; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; }
        #ui {
            position: absolute; top: 10px; left: 10px;
            background: rgba(0,0,0,0.75); color: white;
            padding: 15px; border-radius: 8px; pointer-events: auto;
            max-width: 300px;
        }
        .role-player { color: #aaaaaa; font-weight: bold; }
        .role-mod { color: #55ff55; font-weight: bold; }
        .role-admin { color: #ff5555; font-weight: bold; }
        .role-curator { color: #aa00aa; font-weight: bold; }
        
        #online-panel {
            position: absolute; top: 10px; right: 10px;
            background: rgba(0,0,0,0.75); color: white;
            padding: 15px; border-radius: 8px;
        }
        input, button, select {
            background: #222; color: white; border: 1px solid #555;
            padding: 5px; margin-top: 5px; border-radius: 4px; width: 100%;
        }
        button:hover { background: #444; cursor: pointer; }
        #player-list { margin-top: 10px; font-size: 14px; }
        #admin-menu {
            display: none; position: absolute; top: 50%; left: 50%;
            transform: translate(-50%, -50%); background: rgba(0,0,0,0.9);
            color: white; padding: 20px; border-radius: 10px; border: 2px solid #ff5555;
        }
    </style>
    <!-- Библиотеки: Three.js для 3D и PeerJS для бесплатного онлайна без сервера -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script src="https://unpkg.com/peerjs@1.4.7/dist/peerjs.min.js"></script>
</head>
<body>

    <!-- Панель управления и инфо -->
    <div id="ui">
        <h3>Управление:</h3>
        <p>W, A, S, D — Движение<br>Мышь — Обзор (Клик для захвата)<br><b>M</b> — Меню Управления Ролями</p>
        <hr>
        <div>Ваш статус: <span id="my-role-display" class="role-curator">Куратор</span></div>
        <div id="mod-status" style="font-size:12px; color:#ffaa00; margin-top:5px;">Мод: Рубиновая руда активна</div>
    </div>

    <!-- Панель мультиплеера -->
    <div id="online-panel">
        <h3>Мультиплеер</h3>
        <div style="font-size:12px;">Ваш сетевой ID (нажмите, чтобы скопировать):</div>
        <input type="text" id="my-id" readonly onclick="this.select(); document.execCommand('copy');">
        
        <div style="margin-top: 10px; font-size:12px;">Подключиться к другу:</div>
        <input type="text" id="peer-id-input" placeholder="Вставьте ID друга...">
        <button id="connect-btn">Подключиться</button>

        <h4>Игроки в сети:</h4>
        <div id="player-list"></div>
    </div>

    <!-- Меню Администратора / Куратора -->
    <div id="admin-menu">
        <h3>Панель Управления Ролями</h3>
        <label>Выберите игрока:</label>
        <select id="admin-player-select"></select>
        <label style="margin-top:10px; display:block;">Назначить роль:</label>
        <select id="admin-role-select">
            <option value="Игрок">Игрок</option>
            <option value="Модератор">Модератор</option>
            <option value="Администратор">Администратор</option>
            <option value="Куратор">Куратор</option>
        </select>
        <button id="admin-apply-btn" style="background:#ff5555; margin-top:15px;">Применить</button>
        <button onclick="document.getElementById('admin-menu').style.display='none';" style="background:#555; margin-top:5px;">Закрыть</button>
    </div>

    <script>
        // --- 1. НАСТРОЙКА ИЕРАРХИИ РОЛЕЙ ---
        const ROLES = {
            'Игрок': { class: 'role-player', level: 1 },
            'Модератор': { class: 'role-mod', level: 2 },
            'Администратор': { class: 'role-admin', level: 3 },
            'Куратор': { class: 'role-curator', level: 4 }
        };

        // Ты создатель игры, поэтому при запуске у тебя роль Куратора
        let myData = {
            id: '',
            nickname: 'Разработчик_' + Math.floor(Math.random() * 900),
            role: 'Куратор',
            x: 8, y: 2.9, z: 8,
            ry: 0
        };

        // --- 2. НАСТРОЙКА 3D СЦЕНЫ ---
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x87CEEB);

        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.body.appendChild(renderer.domElement);

        scene.add(new THREE.DirectionalLight(0xffffff, 1).position.set(5, 10, 7).normalize());
        scene.add(new THREE.AmbientLight(0x404040));

        // Генерация карты (16х16)
        const geometry = new THREE.BoxGeometry(1, 1, 1);
        const dirtMat = new THREE.MeshLambertMaterial({ color: 0x866043 });
        const grassMat = new THREE.MeshLambertMaterial({ color: 0x559933 });
        const rubyMat = new THREE.MeshLambertMaterial({ color: 0xE0115F }); // Блок из мода

        for (let x = 0; x < 16; x++) {
            for (let z = 0; z < 16; z++) {
                let d = new THREE.Mesh(geometry, dirtMat); d.position.set(x, 0, z); scene.add(d);
                let topMat = Math.random() < 0.05 ? rubyMat : grassMat;
                let t = new THREE.Mesh(geometry, topMat); t.position.set(x, 1, z); scene.add(t);
            }
        }

        // Локальный игрок
        const playerGeom = new THREE.CylinderGeometry(0.4, 0.4, 1.8, 16);
        const playerMat = new THREE.MeshLambertMaterial({ color: 0xCC0000 });
        const playerMesh = new THREE.Mesh(playerGeom, playerMat);
        playerMesh.position.set(myData.x, myData.y, myData.z);
        scene.add(playerMesh);

        // Камера от 3-го лица
        let cameraDistance = 5, cameraPitch = 0.3, cameraYaw = 0;
        const keys = { w: false, a: false, s: false, d: false };

        window.addEventListener('keydown', (e) => { 
            if(keys.hasOwnProperty(e.key.toLowerCase())) keys[e.key.toLowerCase()] = true; 
            if(e.key.toLowerCase() === 'm') toggleAdminMenu();
        });
        window.addEventListener('keyup', (e) => { if(keys.hasOwnProperty(e.key.toLowerCase())) keys[e.key.toLowerCase()] = false; });
        window.addEventListener('mousemove', (e) => {
            if (document.pointerLockElement === document.body) {
                cameraYaw -= e.movementX * 0.003;
                cameraPitch = Math.max(-0.5, Math.min(0.8, cameraPitch - e.movementY * 0.003));
            }
        });
        renderer.domElement.addEventListener('click', () => { renderer.domElement.requestPointerLock(); });

        // --- 3. СЕТЕВАЯ ЛОГИКА (ОНЛАЙН ЧЕРЕЗ PEERJS) ---
        const peer = new Peer(); 
        let connections = {}; 
        let remotePlayers = {}; 

        // Получение личного ID для онлайна
        peer.on('open', (id) => {
            myData.id = id;
            document.getElementById('my-id').value = id;
            updatePlayerList();
        });

        // Когда к нам кто-то подключается
        peer.on('connection', (conn) => {
            setupConnection(conn);
        });

        // Кнопка подключения к другу
        document.getElementById('connect-btn').addEventListener('click', () => {
            const connectId = document.getElementById('peer-id-input').value;
            if(connectId) {
                let conn = peer.connect(connectId);
                setupConnection(conn);
            }
        });

        function setupConnection(conn) {
            conn.on('open', () => {
                connections[conn.peer] = conn;
                // Отправляем свои данные новому игроку
                conn.send({ type: 'init', data: myData });
            });

            conn.on('data', (msg) => {
                handleNetworkMessage(msg, conn);
            });

            conn.on('close', () => {
                if(remotePlayers[conn.peer]) {
                    scene.remove(remotePlayers[conn.peer].mesh);
                    delete remotePlayers[conn.peer];
                }
                delete connections[conn.peer];
                updatePlayerList();
            });
        }

        // Обработка пакетов данных
        function handleNetworkMessage(msg, conn) {
            if (msg.type === 'init' || msg.type === 'update') {
                let pData = msg.data;
                if (!remotePlayers[pData.id]) {
                    // Создаем 3D модель для другого игрока (синий цвет)
                    let m = new THREE.Mesh(playerGeom, new THREE.MeshLambertMaterial({ color: 0x0000CC }));
                    scene.add(m);
                    remotePlayers[pData.id] = { data: pData, mesh: m };
                }
                // Обновляем позицию и роль
                remotePlayers[pData.id].data = pData;
                remotePlayers[pData.id].mesh.position.set(pData.x, pData.y, pData.z);
                remotePlayers[pData.id].mesh.rotation.y = pData.ry;
                updatePlayerList();
            }
            if (msg.type === 'set_role') {
                // Сервер/Админ изменил нашу роль
                myData.role = msg.role;
                document.getElementById('my-role-display').innerText = msg.role;
                document.getElementById('my-role-display').className = ROLES[msg.role].class;
                updatePlayerList();
            }
        }

        // Рассылка своих координат всем в сети
        function broadcastData() {
            myData.x = playerMesh.position.x;
            myData.y = playerMesh.position.y;
            myData.z = playerMesh.position.z;
            myData.ry = playerMesh.rotation.y;

            for(let id in connections) {
                connections[id].send({ type: 'update', data: myData });
            }
        }

        // Обновление списка игроков на экране (Таб)
        function updatePlayerList() {
            let listHtml = `<div><span class="${ROLES[myData.role].class}">[${myData.role}]</span> ${myData.nickname} (Вы)</div>`;
            
            // Очищаем селектор в админ-меню и заполняем заново
            const select = document.getElementById('admin-player-select');
            select.innerHTML = '';

            for(let id in remotePlayers) {
                let p = remotePlayers[id].data;
                listHtml += `<div><span class="${ROLES[p.role].class}">[${p.role}]</span> ${p.nickname}</div>`;
                
                let opt = document.createElement('option');
                opt.value = id;
                opt.innerText = p.nickname;
                select.appendChild(opt);
            }
            document.getElementById('player-list').innerHTML = listHtml;
        }

        // --- 4. МЕНЮ УПРАВЛЕНИЯ РОЛЯМИ (ДЛЯ АДМИНОВ/КУРАТОРОВ) ---
        function toggleAdminMenu() {
            // Проверяем, есть ли права (уровень Модератора = 2, Админа = 3, Куратора = 4)
            if (ROLES[myData.role].level < 2) {
                alert("У вас недостаточно прав! Меню доступно от Модератора и выше.");
                return;
            }
            const menu = document.getElementById('admin-menu');
            menu.style.display = (menu.style.display === 'block') ? 'none' : 'block';
        }

        document.getElementById('admin-apply-btn').addEventListener('click', () => {
            const targetId = document.getElementById('admin-player-select').value;
            const newRole = document.getElementById('admin-role-select').value;

            // Проверка: модератор не может выдать роль выше своей или управлять админами
            if(ROLES[myData.role].level <= ROLES[newRole].level && myData.role !== 'Куратор') {
                alert("Вы не можете выдать статус выше или равный своему!");
                return;
            }

            if(targetId && connections[targetId]) {
                // Отправляем игроку команду изменить роль
                connections[targetId].send({ type: 'set_role', role: newRole });
                // Обновляем у себя локально
                remotePlayers[targetId].data.role = newRole;
                updatePlayerList();
                document.getElementById('admin-menu').style.display = 'none';
            }
        });


        // --- 5. ИГРОВОЙ ЦИКЛ ---
        const clock = new THREE.Clock();
        const speed = 5;

        function animate() {
            requestAnimationFrame(animate);
            const delta = clock.getDelta();

            // Движение игрока
            const moveVector = new THREE.Vector3(0, 0, 0);
            if (keys.w) moveVector.z -= 1; if (keys.s) moveVector.z += 1;
            if (keys.a) moveVector.x -= 1; if (keys.d) moveVector.x += 1;
            moveVector.normalize();

            const forward = new THREE.Vector3(0, 0, -1).applyAxisAngle(new THREE.Vector3(0, 1, 0), cameraYaw);
            const side = new THREE.Vector3(1, 0, 0).applyAxisAngle(new THREE.Vector3(0, 1, 0), cameraYaw);
            const finalMovement = new THREE.Vector3().addScaledVector(forward, -moveVector.z).addScaledVector(side, moveVector.x).normalize().multiplyScalar(speed * delta);
            
            playerMesh.position.add(finalMovement);
            if (moveVector.lengthSq() > 0) playerMesh.rotation.y = cameraYaw + Math.atan2(moveVector.x, moveVector.z);

            // Камера от третьего лица
            camera.position.set(
                playerMesh.position.x + cameraDistance * Math.sin(cameraYaw) * Math.cos(cameraPitch),
                playerMesh.position.y + cameraDistance * Math.sin(cameraPitch) + 1,
                playerMesh.position.z + cameraDistance * Math.cos(cameraYaw) * Math.cos(cameraPitch)
            );
            camera.lookAt(playerMesh.position.x, playerMesh.position.y + 0.5, playerMesh.position.z);

            // Отправка координат в сеть
            if(myData.id) broadcastData();

            renderer.render(scene, camera);
        }

        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight; camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });

        animate();
    </script>
</body>
</html>
