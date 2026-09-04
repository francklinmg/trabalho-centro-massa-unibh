<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <title>Simulação 3D: Estabilidade do Caminhão na Rampa</title>
  <style>
    body {
      margin: 0;
      overflow: hidden;
      font-family: Arial, sans-serif;
      background-color: #1a1a1a;
      color: #fff;
    }
    #ui {
      position: absolute;
      top: 20px;
      left: 20px;
      background: rgba(0, 0, 0, 0.85);
      padding: 15px;
      border-radius: 8px;
      box-shadow: 0 4px 10px rgba(0,0,0,0.5);
      width: 300px;
    }
    .control-group {
      margin-bottom: 12px;
    }
    label {
      display: block;
      margin-bottom: 4px;
      font-size: 0.9em;
    }
    input[type="range"], select {
      width: 100%;
      padding: 4px;
    }
    #status {
      font-weight: bold;
      margin-top: 10px;
      padding-top: 8px;
      border-top: 1px solid #444;
    }
    .info-box {
      font-size: 0.85em;
      color: #aaa;
      margin-top: 8px;
    }
  </style>
  <!-- Three.js CDN -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
  <!-- OrbitControls -->
  <script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/controls/OrbitControls.js"></script>
</head>
<body>

  <div id="ui">
    <div class="control-group">
      <label for="rampAngle">Inclinação da Rampa: <span id="angleValue">0</span>°</label>
      <input type="range" id="rampAngle" min="0" max="45" value="0" step="0.5">
    </div>

    <div class="control-group">
      <label for="cargoState">Estado da Carga:</label>
      <select id="cargoState">
        <option value="empty">Vazio (CM Baixo)</option>
        <option value="half">Carga Média</option>
        <option value="full" selected>Carga Total (CM Alto)</option>
      </select>
    </div>

    <div class="info-box">
      Altura do CM ($h_{CM}$): <strong id="cmHeightValue">0.0</strong>m<br>
      Ângulo Limite Teórico: <strong id="criticalAngleValue">0°</strong>
    </div>

    <div id="status">Estado: Estável</div>
  </div>

<script>
  // 1. Cena e Câmera
  const scene = new THREE.Scene();
  scene.background = new THREE.Color(0x1e1e24);

  const camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 1000);
  camera.position.set(15, 10, 18);

  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.shadowMap.enabled = true;
  document.body.appendChild(renderer.domElement);

  const controls = new THREE.OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;

  // 2. Iluminação
  scene.add(new THREE.AmbientLight(0xffffff, 0.6));
  const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
  dirLight.position.set(10, 20, 10);
  dirLight.castShadow = true;
  scene.add(dirLight);

  // 3. Rampa
  const rampGroup = new THREE.Group();
  scene.add(rampGroup);

  const rampGeo = new THREE.BoxGeometry(16, 0.4, 8);
  const rampMat = new THREE.MeshStandardMaterial({ color: 0x444444 });
  const ramp = new THREE.Mesh(rampGeo, rampMat);
  ramp.position.set(0, -0.2, 0);
  ramp.receiveShadow = true;
  rampGroup.add(ramp);

  // 4. Construção do Caminhão 3D
  const truckGroup = new THREE.Group();
  rampGroup.add(truckGroup);

  // Chassi / Base
  const chassisGeo = new THREE.BoxGeometry(7, 0.4, 3);
  const chassisMat = new THREE.MeshStandardMaterial({ color: 0x222222 });
  const chassis = new THREE.Mesh(chassisGeo, chassisMat);
  chassis.position.set(0, 0.6, 0);
  truckGroup.add(chassis);

  // Rodas (Largura de apoio b = 3.2)
  const wheelGeo = new THREE.CylinderGeometry(0.5, 0.5, 0.4, 16);
  const wheelMat = new THREE.MeshStandardMaterial({ color: 0x111111 });
  wheelGeo.rotateX(Math.PI / 2);

  const wheelPositions = [
    [-2, 0.5, 1.5], [-2, 0.5, -1.5],
    [2, 0.5, 1.5], [2, 0.5, -1.5]
  ];
  wheelPositions.forEach(pos => {
    const wheel = new THREE.Mesh(wheelGeo, wheelMat);
    wheel.position.set(...pos);
    truckGroup.add(wheel);
  });

  // Cabine
  const cabinGeo = new THREE.BoxGeometry(2, 2, 2.8);
  const cabinMat = new THREE.MeshStandardMaterial({ color: 0xd90429 });
  const cabin = new THREE.Mesh(cabinGeo, cabinMat);
  cabin.position.set(2.2, 1.8, 0);
  truckGroup.add(cabin);

  // Baú de Carga
  const trunkGeo = new THREE.BoxGeometry(4.5, 3, 2.8);
  const trunkMat = new THREE.MeshStandardMaterial({ color: 0xedf2f4, transparent: true, opacity: 0.6 });
  const trunk = new THREE.Mesh(trunkGeo, trunkMat);
  trunk.position.set(-1.2, 2.3, 0);
  truckGroup.add(trunk);

  // Carga Visível Interna (Ajusta a altura)
  const cargoGeo = new THREE.BoxGeometry(4.3, 1, 2.6);
  const cargoMat = new THREE.MeshStandardMaterial({ color: 0xf77f00 });
  const cargoMesh = new THREE.Mesh(cargoGeo, cargoMat);
  cargoMesh.position.set(-1.2, 1.3, 0);
  truckGroup.add(cargoMesh);

  // Esfera do Centro de Massa (CM)
  const cmGeo = new THREE.SphereGeometry(0.25, 16, 16);
  const cmMat = new THREE.MeshBasicMaterial({ color: 0xffff00 });
  const cmMesh = new THREE.Mesh(cmGeo, cmMat);
  truckGroup.add(cmMesh);

  // Linha da Gravidade (Vetor Vetorial descendo do CM)
  const gravityLineGeo = new THREE.BufferGeometry().setFromPoints([
    new THREE.Vector3(0, 0, 0),
    new THREE.Vector3(0, -10, 0)
  ]);
  const gravityLineMat = new THREE.LineDashedMaterial({ 
    color: 0x06d6a0, 
    dashSize: 0.2, 
    gapSize: 0.1 
  });
  const gravityLine = new THREE.Line(gravityLineGeo, gravityLineMat);
  gravityLine.computeLineDistances();
  scene.add(gravityLine);

  // 5. Interface e Lógica de Física
  const rampAngleInput = document.getElementById('rampAngle');
  const cargoStateSelect = document.getElementById('cargoState');
  const angleLabel = document.getElementById('angleValue');
  const cmHeightLabel = document.getElementById('cmHeightValue');
  const criticalAngleLabel = document.getElementById('criticalAngleValue');
  const statusLabel = document.getElementById('status');

  const halfTrackWidth = 1.5; // Metade da largura entre rodas (b / 2)

  function update() {
    const angleDeg = parseFloat(rampAngleInput.value);
    const cargoState = cargoStateSelect.value;

    let cmHeight = 1.1; // Vazio: CM baixo perto do chassi
    if (cargoState === 'half') {
      cmHeight = 1.8;
      cargoMesh.visible = true;
      cargoMesh.scale.set(1, 1, 1);
      cargoMesh.position.y = 1.3;
    } else if (cargoState === 'full') {
      cmHeight = 2.5; // Cheio: CM alto no meio do baú
      cargoMesh.visible = true;
      cargoMesh.scale.set(1, 2.5, 1);
      cargoMesh.position.y = 2.05;
    } else {
      cargoMesh.visible = false;
    }

    // Posição do CM no caminhão
    cmMesh.position.set(-0.2, cmHeight, 0);

    // Ângulo crítico: arctan((b/2) / h_CM)
    const criticalAngleRad = Math.atan(halfTrackWidth / cmHeight);
    const criticalAngleDeg = (criticalAngleRad * 180) / Math.PI;

    angleLabel.textContent = angleDeg.toFixed(1);
    cmHeightLabel.textContent = cmHeight.toFixed(2);
    criticalAngleLabel.textContent = `${criticalAngleDeg.toFixed(1)}°`;

    // Inclina Rampa
    const angleRad = THREE.MathUtils.degToRad(angleDeg);
    rampGroup.rotation.z = angleRad;

    // Atualiza linha da gravidade no espaço do mundo
    const cmWorldPos = new THREE.Vector3();
    cmMesh.getWorldPosition(cmWorldPos);
    gravityLine.position.copy(cmWorldPos);

    // Condição de Tombamento
    if (angleDeg < criticalAngleDeg) {
      gravityLineMat.color.setHex(0x06d6a0); // Verde
      statusLabel.textContent = "Estado: Estável";
      statusLabel.style.color = "#06d6a0";
      truckGroup.rotation.z = 0;
    } else {
      gravityLineMat.color.setHex(0xef476f); // Vermelho
      statusLabel.textContent = `Estado: TOMBOU! (Limite: ${criticalAngleDeg.toFixed(1)}°)`;
      statusLabel.style.color = "#ef476f";

      // Animação do tombo
      const extraAngle = THREE.MathUtils.degToRad(angleDeg - criticalAngleDeg);
      truckGroup.rotation.z = -extraAngle * 2;
    }
  }

  rampAngleInput.addEventListener('input', update);
  cargoStateSelect.addEventListener('change', update);

  function animate() {
    requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
  }

  window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  });

  update();
  animate();
</script>
</body>
</html>
