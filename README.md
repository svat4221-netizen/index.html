<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <title>3D Craft Sandbox</title>
  <style>
    body {
      margin: 0;
      overflow: hidden;
      font-family: sans-serif;
      user-select: none;
    }
    #ui {
      position: absolute;
      top: 10px;
      left: 10px;
      color: white;
      background: rgba(0, 0, 0, 0.6);
      padding: 12px;
      border-radius: 8px;
      pointer-events: none;
      z-index: 10;
    }
    .badge {
      padding: 2px 6px;
      border-radius: 4px;
      font-weight: bold;
    }
    #crosshair {
      position: absolute;
      top: 50%;
      left: 50%;
      width: 10px;
      height: 10px;
      color: white;
      transform: translate(-50%, -50%);
      font-size: 20px;
      pointer-events: none;
      z-index: 10;
    }
    #instructions {
      position: absolute;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      color: white;
      background: rgba(0,0,0,0.8);
      padding: 20px;
      text-align: center;
      border-radius: 8px;
      cursor: pointer;
      z-index: 20;
    }
  </style>
  <!-- Подключение Three.js и контроллера управления -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/controls/PointerLockControls.js"></script>
</head>
<body>

<div id="instructions">
  <h2>Кликните, чтобы начать игру</h2>
  <p>Управление: WASD — перемещение, Мышь — осмотр<br>ЛКМ — сломать блок</p>
</div>

<div id="crosshair">+</div>

<div id="ui">
  <div>Ранг: <span id="player-rank" class="badge"></span></div>
  <div>Инвентарь: <span id="inventory">Пусто</span></div>
</div>

<script>
  // --- 1. СИСТЕМА РАНГОВ ---
  const RANKS = {
    PLAYER: { name: 'Обычный игрок', color: '#888888' },
    MODERATOR: { name: 'Модератор', color: '#1E90FF' },
    ADMIN: { name: 'Администратор', color: '#FF4500' },
    CURATOR: { name: 'Куратор', color: '#9370DB' },
    CREATOR: { name: 'Создатель', color: '#FFD700' }
  };

  const player = {
    rank: RANKS.CREATOR,
    inventory: {}
  };

  // --- 2. СЦЕНА, КАМЕРА И ОСВЕЩЕНИЕ ---
  const scene = new THREE.Scene();
  scene.background = new THREE.Color(0x87CEEB); // Небо

  const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  document.body.appendChild(renderer.domElement);

  // Свет
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
  scene.add(ambientLight);

  const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
  dirLight.position.set(20, 40, 20);
  scene.add(dirLight);

  // --- 3. РЕЕСТР БЛОКОВ И МАТЕРИАЛЫ ---
  const blockGeometry = new THREE.BoxGeometry(1, 1, 1);
  const BLOCKS = {
    GRASS: { name: 'Трава', material: new THREE.MeshLamberMaterial({ color: 0x4CA64C }) },
    DIRT: { name: 'Земля', material: new THREE.MeshLamberMaterial({ color: 0x8B4513 }) },
    STONE: { name: 'Камень', material: new THREE.MeshLamberMaterial({ color: 0x808080 }) },
    DIAMOND_ORE: { name: 'Алмазная руда', material: new THREE.MeshLamberMaterial({ color: 0x00FFFF }) }
  };

  // Исправление метода материала для Three.js
  BLOCKS.GRASS.material = new THREE.MeshLambertMaterial({ color: 0x4CA64C });
  BLOCKS.DIRT.material = new THREE.MeshLambertMaterial({ color: 0x8B4513 });
  BLOCKS.STONE.material = new THREE.MeshLambertMaterial({ color: 0x808080 });
  BLOCKS.DIAMOND_ORE.material = new THREE.MeshLambertMaterial({ color: 0x00FFFF });

  const objects = []; // Массив блоков для проверки кликов

  // --- 4. ГЕНЕРАЦИЯ 3D-МИРА ---
  const worldSize = 16;
  for (let x = -worldSize / 2; x < worldSize / 2; x++) {
    for (let z = -worldSize / 2; z < worldSize / 2; z++) {
      for (let y = 0; y < 5; y++) {
        let blockType = BLOCKS.STONE;
        if (y === 4) blockType = BLOCKS.GRASS;
        else if (y >= 2) blockType = BLOCKS.DIRT;
        else if (Math.random() < 0.15) blockType = BLOCKS.DIAMOND_ORE;

        const cube = new THREE.Mesh(blockGeometry, blockType.material);
        cube.position.set(x, y, z);
        cube.userData = { name: blockType.name };
        scene.add(cube);
        objects.push(cube);
      }
    }
  }

  // --- 5. СИСТЕМА МОДОВ ---
  const ModLoader = {
    registerBlock: function(key, name, colorHex) {
      BLOCKS[key] = {
        name: name,
        material: new THREE.MeshLambertMaterial({ color: colorHex })
      };
      console.log(`[ModLoader] Добавлен блок: ${name}`);
    }
  };

  // Пример работы мода: добавляем Рубин
  ModLoader.registerBlock('RUBY_ORE', 'Рубиновая руда', 0xE0115F);

  // --- 6. УПРАВЛЕНИЕ И ИГРОВОЙ ЦИКЛ ---
  const controls = new THREE.PointerLockControls(camera, document.body);
  const instructions = document.getElementById('instructions');

  instructions.addEventListener('click', () => {
    controls.lock();
  });

  controls.addEventListener('lock', () => {
    instructions.style.display = 'none';
  });

  controls.addEventListener('unlock', () => {
    instructions.style.display = '';
  });

  camera.position.set(0, 7, 0);

  // Движение (WASD)
  let moveForward = false, moveBackward = false, moveLeft = false, moveRight = false;
  const velocity = new THREE.Vector3();
  const direction = new THREE.Vector3();

  document.addEventListener('keydown', (e) => {
    if (e.code === 'KeyW') moveForward = true;
    if (e.code === 'KeyS') moveBackward = true;
    if (e.code === 'KeyA') moveLeft = true;
    if (e.code === 'KeyD') moveRight = true;
  });

  document.addEventListener('keyup', (e) => {
    if (e.code === 'KeyW') moveForward = false;
    if (e.code === 'KeyS') moveBackward = false;
    if (e.code === 'KeyA') moveLeft = false;
    if (e.code === 'KeyD') moveRight = false;
  });

  // Логика разрушения блоков (Raycasting)
  const raycaster = new THREE.Raycaster();
  const mouse = new THREE.Vector2(0, 0); // Центр экрана

  window.addEventListener('click', (e) => {
    if (!controls.isLocked) return;

    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(objects);

    if (intersects.length > 0) {
      const hitBlock = intersects[0].object;
      
      // Добавляем предмет в инвентарь
      const blockName = hitBlock.userData.name;
      player.inventory[blockName] = (player.inventory[blockName] || 0) + 1;
      updateUI();

      // Удаляем блок со сцены
      scene.remove(hitBlock);
      objects.splice(objects.indexOf(hitBlock), 1);
    }
  });

  // --- 7. UI И АНИМАЦИЯ ---
  function updateUI() {
    const rankEl = document.getElementById('player-rank');
    rankEl.textContent = player.rank.name;
    rankEl.style.backgroundColor = player.rank.color;

    const invItems = Object.entries(player.inventory)
      .map(([name, count]) => `${name}: ${count}`)
      .join(', ');
    document.getElementById('inventory').textContent = invItems || 'Пусто';
  }

  let prevTime = performance.now();

  function animate() {
    requestAnimationFrame(animate);

    const time = performance.now();
    if (controls.isLocked) {
      const delta = (time - prevTime) / 1000;

      velocity.x -= velocity.x * 10.0 * delta;
      velocity.z -= velocity.z * 10.0 * delta;

      direction.z = Number(moveForward) - Number(moveBackward);
      direction.x = Number(moveRight) - Number(moveLeft);
      direction.normalize();

      if (moveForward || moveBackward) velocity.z -= direction.z * 40.0 * delta;
      if (moveLeft || moveRight) velocity.x -= direction.x * 40.0 * delta;

      controls.moveRight(-velocity.x * delta);
      controls.moveForward(-velocity.z * delta);
    }
    prevTime = time;

    renderer.render(scene, camera);
  }

  window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  });

  updateUI();
  animate();
</script>

</body>
</html>


