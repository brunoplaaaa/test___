<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>WebXR - Química AR Automática</title>
    
    <!-- Three.js núcleo -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>

    <style>
        body {
            margin: 0;
            overflow: hidden;
            font-family: 'Arial', sans-serif;
            background: black;
            touch-action: none;
        }
        #status {
            position: absolute;
            top: 30px;
            right: 30px;
            background: rgba(0,0,0,0.8);
            color: #0ff;
            padding: 14px 22px;
            border-radius: 40px;
            font-size: 16px;
            border: 1px solid #00ccff;
            backdrop-filter: blur(4px);
            pointer-events: none;
            z-index: 200;
            box-shadow: 0 0 20px rgba(0,200,255,0.3);
        }
        #instructions {
            position: absolute;
            top: 30px;
            left: 30px;
            background: rgba(20,20,30,0.8);
            color: #ddd;
            padding: 16px 22px;
            border-radius: 16px;
            font-size: 15px;
            backdrop-filter: blur(4px);
            border-left: 5px solid #ffaa66;
            pointer-events: none;
            z-index: 200;
            max-width: 260px;
            line-height: 1.6;
        }
        #ar-button {
            position: absolute;
            bottom: 40px;
            right: 40px;
            background: #00ccff;
            color: black;
            padding: 18px 30px;
            border-radius: 50px;
            font-size: 20px;
            font-weight: bold;
            border: none;
            box-shadow: 0 4px 20px rgba(0,200,255,0.6);
            pointer-events: auto;
            z-index: 300;
            cursor: pointer;
            transition: transform 0.2s;
        }
        #ar-button:hover {
            transform: scale(1.05);
        }
        #info-panel {
            position: absolute;
            bottom: 30px;
            left: 30px;
            width: 300px;
            background: rgba(10, 20, 30, 0.9);
            backdrop-filter: blur(10px);
            border-left: 6px solid #00ccff;
            border-radius: 18px;
            padding: 20px 25px;
            color: white;
            font-size: 16px;
            box-shadow: 0 8px 30px rgba(0,0,0,0.7);
            border: 1px solid rgba(255,255,255,0.15);
            transition: opacity 0.4s ease, visibility 0.4s;
            pointer-events: none;
            z-index: 100;
        }
        .hidden-panel {
            opacity: 0;
            visibility: hidden;
        }
        .visible-panel {
            opacity: 1;
            visibility: visible;
        }
        canvas {
            display: block;
        }
    </style>
</head>
<body>
    <div id="info-panel" class="hidden-panel">
        <h3 id="particle-title" style="margin-top:0; color:#88ddff;">Átomo</h3>
        <p><span class="highlight" style="color:#ffaa66;">Información:</span> <span id="extra-info" style="color:#fff;">---</span></p>
    </div>

    <div id="status">🌀 Inicializando AR...</div>
    <div id="instructions">
        <strong style="color:#88ddff;">✨ Modo Automático</strong><br>
        • Los átomos aparecen solos sobre superficies<br>
        • Se mueven de forma autónoma<br>
        • Acerca Oxígeno (rojo) e Hidrógeno (blanco) para crear agua<br>
        🖐️ Arrastra con 1 dedo · ✌️ Escala con 2 dedos<br>
        👆👆 Doble tap para ver detalles
    </div>
    <button id="ar-button">🚀 Iniciar AR</button>

    <script>
        (async function() {
            // --- VARIABLES GLOBALES ---
            const infoPanel = document.getElementById('info-panel');
            const statusDiv = document.getElementById('status');
            const arButton = document.getElementById('ar-button');

            // --- ESTADO DE LA APLICACIÓN ---
            let xrSession = null;
            let xrReferenceSpace = null;
            let xrHitTestSource = null;
            let lastHitPose = null;          // Última superficie detectada
            let automaticSpawnTimer = 0;      // Para espaciado de apariciones

            // --- MODELOS ---
            let oxygenModel = null;
            let hydrogenModel = null;
            let waterModel = null;

            // --- MOVIMIENTO AUTÓNOMO ---
            let movingObjects = [];           // Lista de objetos con sus velocidades
            const MOVE_SPEED = 0.02;          // Velocidad de desplazamiento
            const BOUNDARY_RADIUS = 1.2;      // Área virtual de movimiento

            // --- INTERACCIÓN ---
            let selectedObject = null;
            let isDragging = false;
            let initialPinchDistance = 0;
            let initialScale = 1;
            let lastTapTime = 0;
            let showDetailedInfo = false;

            // --- Three.js setup ---
            const scene = new THREE.Scene();
            scene.background = new THREE.Color(0x050510); // Azul oscuro nocturno
            
            const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.01, 20);
            
            const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: false });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // Calidad óptima
            renderer.xr.enabled = true;
            renderer.shadowMap.enabled = true; // Sombras para más realismo
            renderer.shadowMap.type = THREE.PCFSoftShadowMap;
            document.body.appendChild(renderer.domElement);

            // --- ILUMINACIÓN CINEMATOGRÁFICA ---
            const ambientLight = new THREE.AmbientLight(0x404060);
            scene.add(ambientLight);

            const mainLight = new THREE.DirectionalLight(0xffeedd, 1.5);
            mainLight.position.set(1, 1, 1);
            mainLight.castShadow = true;
            mainLight.receiveShadow = true;
            mainLight.shadow.mapSize.width = 1024;
            mainLight.shadow.mapSize.height = 1024;
            mainLight.shadow.camera.near = 0.5;
            mainLight.shadow.camera.far = 5;
            mainLight.shadow.camera.left = -2;
            mainLight.shadow.camera.right = 2;
            mainLight.shadow.camera.top = 2;
            mainLight.shadow.camera.bottom = -2;
            scene.add(mainLight);

            const fillLight = new THREE.DirectionalLight(0x99aaff, 0.8);
            fillLight.position.set(-1, 0.5, 1);
            scene.add(fillLight);

            const backLight = new THREE.PointLight(0x446688, 0.5);
            backLight.position.set(0, 1, -2);
            scene.add(backLight);

            // Pequeña niebla para profundidad
            scene.fog = new THREE.FogExp2(0x050510, 0.8);

            // --- SPRITE DE ETIQUETA MEJORADO (alta resolución) ---
            function createLabelSprite(text, bgColor = '#1e3a4a', textColor = '#ffffff') {
                const canvas = document.createElement('canvas');
                canvas.width = 512;
                canvas.height = 256;
                const ctx = canvas.getContext('2d');
                
                // Sombra
                ctx.shadowColor = 'rgba(0,0,0,0.8)';
                ctx.shadowBlur = 15;
                ctx.shadowOffsetX = 4;
                ctx.shadowOffsetY = 4;
                
                // Fondo con degradado
                const gradient = ctx.createLinearGradient(0, 0, canvas.width, canvas.height);
                gradient.addColorStop(0, bgColor);
                gradient.addColorStop(1, '#2a4a5a');
                ctx.fillStyle = gradient;
                
                const radius = 40;
                ctx.beginPath();
                ctx.moveTo(radius, 20);
                ctx.lineTo(canvas.width - radius, 20);
                ctx.quadraticCurveTo(canvas.width - 20, 20, canvas.width - 20, radius);
                ctx.lineTo(canvas.width - 20, canvas.height - radius);
                ctx.quadraticCurveTo(canvas.width - 20, canvas.height - 20, canvas.width - radius, canvas.height - 20);
                ctx.lineTo(radius, canvas.height - 20);
                ctx.quadraticCurveTo(20, canvas.height - 20, 20, canvas.height - radius);
                ctx.lineTo(20, radius);
                ctx.quadraticCurveTo(20, 20, radius, 20);
                ctx.closePath();
                ctx.fill();
                
                // Borde brillante
                ctx.shadowBlur = 0;
                ctx.strokeStyle = '#88ddff';
                ctx.lineWidth = 4;
                ctx.stroke();
                
                // Texto con brillo
                ctx.font = 'bold 56px "Segoe UI", Arial, sans-serif';
                ctx.fillStyle = textColor;
                ctx.shadowBlur = 12;
                ctx.shadowColor = '#00ccff';
                ctx.textAlign = 'center';
                ctx.textBaseline = 'middle';
                ctx.fillText(text, canvas.width / 2, canvas.height / 2);
                
                const texture = new THREE.CanvasTexture(canvas);
                texture.anisotropy = 16; // Mejor calidad en ángulos
                const material = new THREE.SpriteMaterial({ 
                    map: texture, 
                    depthTest: false, 
                    depthWrite: false,
                    transparent: true
                });
                const sprite = new THREE.Sprite(material);
                sprite.scale.set(0.5, 0.25, 1);
                return sprite;
            }

            // --- MODELOS DE ALTA CALIDAD ---
            function createAtom(color, radius, orbitalColor, orbitalRadius = radius * 1.8, orbitalTube = 0.025, nombre = 'Átomo') {
                const group = new THREE.Group();
                
                // Núcleo con brillo interno
                const sphereGeo = new THREE.SphereGeometry(radius, 64, 32);
                const sphereMat = new THREE.MeshStandardMaterial({ 
                    color, 
                    roughness: 0.2, 
                    metalness: 0.2, 
                    emissive: color, 
                    emissiveIntensity: 0.25,
                    emissiveMap: null
                });
                const sphere = new THREE.Mesh(sphereGeo, sphereMat);
                sphere.castShadow = true;
                sphere.receiveShadow = true;
                group.add(sphere);
                
                // Orbitales animados (se rotarán en el loop)
                const torusGeo = new THREE.TorusGeometry(orbitalRadius, orbitalTube, 24, 48);
                const torusMat = new THREE.MeshStandardMaterial({ 
                    color: orbitalColor, 
                    emissive: orbitalColor, 
                    emissiveIntensity: 0.3, 
                    transparent: true, 
                    opacity: 0.5, 
                    side: THREE.DoubleSide,
                    roughness: 0.6,
                    metalness: 0.2
                });
                
                const torus1 = new THREE.Mesh(torusGeo, torusMat);
                torus1.rotation.x = Math.PI / 2;
                torus1.rotation.z = 0.3;
                group.add(torus1);
                
                const torus2 = new THREE.Mesh(torusGeo, torusMat.clone());
                torus2.rotation.y = Math.PI / 2;
                torus2.rotation.x = 0.5;
                torus2.scale.set(1.1, 1.1, 1.1);
                group.add(torus2);
                
                // Etiqueta
                const label = createLabelSprite(nombre, '#1e3a4a', '#ffffff');
                label.position.y = radius * 2.8;
                group.add(label);
                
                group.userData.label = label;
                group.userData.type = 'atom';
                group.userData.orbitSpeed1 = 0.005;
                group.userData.orbitSpeed2 = -0.003;
                group.userData.torus1 = torus1;
                group.userData.torus2 = torus2;
                return group;
            }

            function createH2() {
                const group = new THREE.Group();
                const hGeo = new THREE.SphereGeometry(0.1, 32, 16);
                const hMat = new THREE.MeshStandardMaterial({ 
                    color: 0xffffff, 
                    roughness: 0.25, 
                    metalness: 0.2, 
                    emissive: 0x88aaff, 
                    emissiveIntensity: 0.2 
                });
                
                const h1 = new THREE.Mesh(hGeo, hMat);
                h1.position.x = -0.15;
                h1.castShadow = true;
                h1.receiveShadow = true;
                group.add(h1);
                
                const h2 = new THREE.Mesh(hGeo, hMat.clone());
                h2.position.x = 0.15;
                h2.castShadow = true;
                h2.receiveShadow = true;
                group.add(h2);
                
                const bondGeo = new THREE.CylinderGeometry(0.03, 0.03, 0.3, 8);
                const bondMat = new THREE.MeshStandardMaterial({ color: 0xcccccc, roughness: 0.5, metalness: 0.8, emissive: 0x446688, emissiveIntensity: 0.1 });
                const bond = new THREE.Mesh(bondGeo, bondMat);
                bond.rotation.z = Math.PI / 2;
                bond.position.x = 0;
                bond.castShadow = true;
                bond.receiveShadow = true;
                group.add(bond);
                
                const label = createLabelSprite('Hidrógeno (H₂)', '#1e3a4a', '#aaddff');
                label.position.y = 0.4;
                group.add(label);
                
                group.userData.label = label;
                group.userData.type = 'h2';
                return group;
            }

            function createH2O() {
                const group = new THREE.Group();
                
                // Oxígeno
                const oGeo = new THREE.SphereGeometry(0.16, 48, 24);
                const oMat = new THREE.MeshStandardMaterial({ 
                    color: 0xff5555, 
                    roughness: 0.2, 
                    metalness: 0.2, 
                    emissive: 0x442222, 
                    emissiveIntensity: 0.3 
                });
                const oxygen = new THREE.Mesh(oGeo, oMat);
                oxygen.castShadow = true;
                oxygen.receiveShadow = true;
                group.add(oxygen);
                
                // Hidrógenos
                const hGeo = new THREE.SphereGeometry(0.1, 32, 16);
                const hMat = new THREE.MeshStandardMaterial({ 
                    color: 0xffffff, 
                    roughness: 0.25, 
                    metalness: 0.2, 
                    emissive: 0x88aaff, 
                    emissiveIntensity: 0.2 
                });
                
                const h1 = new THREE.Mesh(hGeo, hMat);
                h1.position.set(0.2, 0.15, 0);
                h1.castShadow = true;
                h1.receiveShadow = true;
                group.add(h1);
                
                const h2 = new THREE.Mesh(hGeo, hMat.clone());
                h2.position.set(0.2, -0.15, 0);
                h2.castShadow = true;
                h2.receiveShadow = true;
                group.add(h2);
                
                // Enlaces
                const bondGeo = new THREE.CylinderGeometry(0.035, 0.035, 0.25, 8);
                const bondMat = new THREE.MeshStandardMaterial({ color: 0xcccccc, roughness: 0.4, metalness: 0.9, emissive: 0x446688, emissiveIntensity: 0.1 });
                
                const bond1 = new THREE.Mesh(bondGeo, bondMat);
                bond1.position.set(0.1, 0.075, 0);
                bond1.rotation.z = -0.7;
                bond1.castShadow = true;
                bond1.receiveShadow = true;
                group.add(bond1);
                
                const bond2 = new THREE.Mesh(bondGeo, bondMat);
                bond2.position.set(0.1, -0.075, 0);
                bond2.rotation.z = 0.7;
                bond2.castShadow = true;
                bond2.receiveShadow = true;
                group.add(bond2);
                
                const label = createLabelSprite('Agua (H₂O)', '#1e4a5a', '#88ddff');
                label.position.y = 0.45;
                group.add(label);
                
                group.userData.label = label;
                group.userData.type = 'water';
                return group;
            }

            // --- INICIALIZAR MODELOS Y AÑADIRLOS OCULTOS ---
            function initModels() {
                oxygenModel = createAtom(0xff5555, 0.16, 0x88ccff, 0.28, 0.025, 'Oxígeno (O)');
                oxygenModel.visible = false;
                scene.add(oxygenModel);

                hydrogenModel = createH2();
                hydrogenModel.visible = false;
                scene.add(hydrogenModel);

                waterModel = createH2O();
                waterModel.visible = false;
                scene.add(waterModel);
            }
            initModels();

            // --- SISTEMA DE MOVIMIENTO AUTÓNOMO ---
            function initMovingObject(obj) {
                movingObjects.push({
                    obj: obj,
                    velocity: new THREE.Vector3(
                        (Math.random() - 0.5) * MOVE_SPEED,
                        0,
                        (Math.random() - 0.5) * MOVE_SPEED
                    )
                });
            }

            function updateMovements() {
                movingObjects.forEach(item => {
                    const obj = item.obj;
                    if (!obj.visible) return;
                    
                    // Mover
                    obj.position.x += item.velocity.x;
                    obj.position.z += item.velocity.z;
                    
                    // Rebote en bordes virtuales (centro en el punto de spawn)
                    if (obj.position.x > BOUNDARY_RADIUS) { obj.position.x = BOUNDARY_RADIUS; item.velocity.x *= -1; }
                    if (obj.position.x < -BOUNDARY_RADIUS) { obj.position.x = -BOUNDARY_RADIUS; item.velocity.x *= -1; }
                    if (obj.position.z > BOUNDARY_RADIUS) { obj.position.z = BOUNDARY_RADIUS; item.velocity.z *= -1; }
                    if (obj.position.z < -BOUNDARY_RADIUS) { obj.position.z = -BOUNDARY_RADIUS; item.velocity.z *= -1; }
                    
                    // Pequeña rotación
                    obj.rotation.y += 0.01;
                    obj.rotation.x += 0.005;
                });
                
                // Animar orbitales de los átomos
                if (oxygenModel.visible) {
                    if (oxygenModel.userData.torus1) oxygenModel.userData.torus1.rotation.z += 0.01;
                    if (oxygenModel.userData.torus2) oxygenModel.userData.torus2.rotation.x += 0.008;
                }
            }

            // --- SPAWN AUTOMÁTICO ---
            function trySpawnAtom() {
                if (!lastHitPose || !xrReferenceSpace) return;
                
                const pos = new THREE.Vector3(
                    lastHitPose.transform.position.x,
                    lastHitPose.transform.position.y,
                    lastHitPose.transform.position.z
                );
                
                // Si no hay oxígeno, crear uno
                if (!oxygenModel.visible) {
                    oxygenModel.position.copy(pos);
                    oxygenModel.visible = true;
                    oxygenModel.scale.setScalar(1);
                    initMovingObject(oxygenModel);
                    statusDiv.innerHTML = '🟢 Oxígeno apareció automáticamente';
                } 
                // Si ya hay oxígeno y no hay hidrógeno, crear hidrógeno en otra posición cercana
                else if (!hydrogenModel.visible) {
                    // Desplazar ligeramente para que no coincida
                    pos.x += 0.3;
                    pos.z += 0.2;
                    hydrogenModel.position.copy(pos);
                    hydrogenModel.visible = true;
                    hydrogenModel.scale.setScalar(1);
                    initMovingObject(hydrogenModel);
                    statusDiv.innerHTML = '⚪ Hidrógeno apareció automáticamente';
                }
                // Si ya están ambos, no spawnear más (o podrías reemplazar el más antiguo)
            }

            // --- LÓGICA DE FUSIÓN ---
            function checkFusion() {
                if (oxygenModel.visible && hydrogenModel.visible) {
                    const distance = oxygenModel.position.distanceTo(hydrogenModel.position);
                    if (distance < 0.4) {
                        // Crear agua
                        waterModel.position.copy(oxygenModel.position).add(hydrogenModel.position).multiplyScalar(0.5);
                        waterModel.quaternion.copy(oxygenModel.quaternion).slerp(hydrogenModel.quaternion, 0.5);
                        waterModel.scale.setScalar((oxygenModel.scale.x + hydrogenModel.scale.x) / 2);
                        
                        oxygenModel.visible = false;
                        hydrogenModel.visible = false;
                        waterModel.visible = true;
                        
                        // Quitar de movingObjects y añadir agua
                        movingObjects = movingObjects.filter(item => item.obj !== oxygenModel && item.obj !== hydrogenModel);
                        initMovingObject(waterModel);
                        
                        statusDiv.innerHTML = '💧 ¡Reacción! Se ha formado agua';
                    }
                }
                
                // Si falta alguno, ocultar agua (por si acaso)
                if (!oxygenModel.visible || !hydrogenModel.visible) {
                    waterModel.visible = false;
                }
            }

            // --- PANEL DE INFORMACIÓN MEJORADO ---
            function updateInfoPanelContent() {
                let title = '', info = '';
                if (waterModel.visible) {
                    title = 'Agua (H₂O)';
                    info = 'Molécula polar · Disolvente universal · Punto de ebullición 100°C';
                } else if (oxygenModel.visible) {
                    title = 'Oxígeno (O)';
                    info = 'Nº atómico 8 · Masa 15.999 u · Electronegatividad 3.44 · Esencial para la vida';
                } else if (hydrogenModel.visible) {
                    title = 'Hidrógeno (H₂)';
                    info = 'Nº atómico 1 · Masa 1.008 u · Elemento más abundante · Combustible estelar';
                } else {
                    title = 'Esperando superficie...';
                    info = 'Apunta a una superficie plana para generar átomos automáticamente.';
                }
                document.getElementById('particle-title').innerText = title;
                document.getElementById('extra-info').innerText = info;
            }

            function toggleDetailedInfo() {
                showDetailedInfo = !showDetailedInfo;
                if (showDetailedInfo) {
                    updateInfoPanelContent();
                    infoPanel.classList.remove('hidden-panel');
                    infoPanel.classList.add('visible-panel');
                } else {
                    infoPanel.classList.remove('visible-panel');
                    infoPanel.classList.add('hidden-panel');
                }
            }

            // --- WEBXR: INICIAR SESIÓN AR ---
            async function startAR() {
                if (!navigator.xr) {
                    alert('❌ WebXR no está disponible. Usa Chrome Android con ARCore.');
                    return;
                }

                try {
                    const session = await navigator.xr.requestSession('immersive-ar', {
                        requiredFeatures: ['hit-test', 'local-floor']
                    });
                    xrSession = session;
                    
                    await renderer.xr.setSession(session);
                    
                    xrReferenceSpace = await session.requestReferenceSpace('local-floor');
                    
                    const viewerSpace = await session.requestReferenceSpace('viewer');
                    xrHitTestSource = await session.requestHitTestSource({
                        space: viewerSpace
                    });

                    arButton.style.display = 'none';
                    statusDiv.innerHTML = '✅ AR activo · Detectando superficies...';
                    
                    // Resetear estado
                    movingObjects = [];
                    oxygenModel.visible = false;
                    hydrogenModel.visible = false;
                    waterModel.visible = false;

                    // --- LOOP DE ANIMACIÓN XR ---
                    const onXRFrame = (time, frame) => {
                        if (!xrSession) return;
                        xrSession.requestAnimationFrame(onXRFrame);
                        
                        // Hit test continuo
                        if (frame && xrHitTestSource) {
                            const hitTestResults = frame.getHitTestResults(xrHitTestSource);
                            if (hitTestResults.length > 0) {
                                const pose = hitTestResults[0].getPose(xrReferenceSpace);
                                if (pose) {
                                    lastHitPose = pose;
                                    statusDiv.innerHTML = '📌 Superficie detectada · Generando átomos...';
                                    
                                    // Spawn automático cada cierto tiempo
                                    automaticSpawnTimer++;
                                    if (automaticSpawnTimer > 30) { // ~ cada 30 frames
                                        trySpawnAtom();
                                        automaticSpawnTimer = 0;
                                    }
                                }
                            } else {
                                statusDiv.innerHTML = '🔍 Mueve la cámara para buscar una superficie';
                            }
                        }
                        
                        // Movimiento autónomo
                        updateMovements();
                        
                        // Verificar fusión
                        checkFusion();
                    };
                    
                    xrSession.requestAnimationFrame(onXRFrame);
                    
                } catch (e) {
                    console.error(e);
                    alert('Error al iniciar AR: ' + e.message);
                }
            }

            // --- EVENTOS TÁCTILES PARA INTERACCIÓN MANUAL ---
            function setupTouchEvents() {
                const canvas = renderer.domElement;

                canvas.addEventListener('touchstart', (e) => {
                    if (e.touches.length === 1) {
                        // Seleccionar objeto (si tocan sobre él)
                        const touch = e.touches[0];
                        const coords = new THREE.Vector2(
                            (touch.clientX / window.innerWidth) * 2 - 1,
                            -(touch.clientY / window.innerHeight) * 2 + 1
                        );
                        const raycaster = new THREE.Raycaster();
                        raycaster.setFromCamera(coords, camera);
                        
                        const objects = [];
                        if (oxygenModel.visible) objects.push(oxygenModel);
                        if (hydrogenModel.visible) objects.push(hydrogenModel);
                        if (waterModel.visible) objects.push(waterModel);
                        
                        const intersects = raycaster.intersectObjects(objects, true);
                        
                        if (intersects.length > 0) {
                            let obj = intersects[0].object;
                            while (obj.parent && obj.parent !== scene) {
                                obj = obj.parent;
                            }
                            selectedObject = obj;
                            isDragging = true;
                            
                            // Detener movimiento autónomo mientras se arrastra
                            const movingItem = movingObjects.find(item => item.obj === selectedObject);
                            if (movingItem) {
                                movingItem.velocity.set(0, 0, 0);
                            }
                            e.preventDefault();
                        }
                    } else if (e.touches.length === 2) {
                        const dx = e.touches[0].clientX - e.touches[1].clientX;
                        const dy = e.touches[0].clientY - e.touches[1].clientY;
                        initialPinchDistance = Math.sqrt(dx * dx + dy * dy);
                        
                        if (oxygenModel.visible) initialScale = oxygenModel.scale.x;
                        else if (hydrogenModel.visible) initialScale = hydrogenModel.scale.x;
                        else if (waterModel.visible) initialScale = waterModel.scale.x;
                        e.preventDefault();
                    }
                }, { passive: false });

                canvas.addEventListener('touchmove', (e) => {
                    if (e.touches.length === 1 && isDragging && selectedObject) {
                        const touch = e.touches[0];
                        const coords = new THREE.Vector2(
                            (touch.clientX / window.innerWidth) * 2 - 1,
                            -(touch.clientY / window.innerHeight) * 2 + 1
                        );
                        const raycaster = new THREE.Raycaster();
                        raycaster.setFromCamera(coords, camera);
                        
                        const plane = new THREE.Plane(new THREE.Vector3(0, 1, 0), 0);
                        const target = new THREE.Vector3();
                        if (raycaster.ray.intersectPlane(plane, target)) {
                            selectedObject.position.copy(target);
                            selectedObject.position.y = 0;
                        }
                        e.preventDefault();
                    } else if (e.touches.length === 2) {
                        const dx = e.touches[0].clientX - e.touches[1].clientX;
                        const dy = e.touches[0].clientY - e.touches[1].clientY;
                        const distance = Math.sqrt(dx * dx + dy * dy);
                        const scaleFactor = distance / initialPinchDistance;
                        const newScale = Math.max(0.5, Math.min(2, initialScale * scaleFactor));
                        
                        if (oxygenModel.visible) oxygenModel.scale.setScalar(newScale);
                        if (hydrogenModel.visible) hydrogenModel.scale.setScalar(newScale);
                        if (waterModel.visible) waterModel.scale.setScalar(newScale);
                        e.preventDefault();
                    }
                }, { passive: false });

                canvas.addEventListener('touchend', (e) => {
                    if (e.touches.length === 0) {
                        if (isDragging && selectedObject) {
                            // Restaurar velocidad aleatoria al soltar
                            const movingItem = movingObjects.find(item => item.obj === selectedObject);
                            if (movingItem) {
                                movingItem.velocity.set(
                                    (Math.random() - 0.5) * MOVE_SPEED,
                                    0,
                                    (Math.random() - 0.5) * MOVE_SPEED
                                );
                            }
                        }
                        isDragging = false;
                        selectedObject = null;
                    }
                    
                    const currentTime = new Date().getTime();
                    const tapLength = currentTime - lastTapTime;
                    if (tapLength < 300 && tapLength > 0) {
                        toggleDetailedInfo();
                        e.preventDefault();
                    }
                    lastTapTime = currentTime;
                });
            }
            setupTouchEvents();

            // --- BOTÓN DE INICIO ---
            arButton.addEventListener('click', startAR);

            // --- LOOP DE RENDER (para modo no AR) ---
            function animate() {
                if (!xrSession) {
                    renderer.render(scene, camera);
                }
                requestAnimationFrame(animate);
            }
            animate();

            // --- REDIMENSIONAR ---
            window.addEventListener('resize', () => {
                camera.aspect = window.innerWidth / window.innerHeight;
                camera.updateProjectionMatrix();
                renderer.setSize(window.innerWidth, window.innerHeight);
            });

            // Inicializar panel oculto
            infoPanel.classList.add('hidden-panel');
            statusDiv.innerHTML = '🌍 Presiona "Iniciar AR" para comenzar';
        })();
    </script>
</body>
</html>
