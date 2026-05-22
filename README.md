<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0">
    <title>Ghost Recon: Tactical Multi-Ops</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <style>
        body, html { 
            margin: 0; padding: 0; overflow: hidden; width: 100%; height: 100%; 
            font-family: monospace; background-color: #0c0a09; 
            -webkit-user-select: none; user-select: none; 
        }
        #canvas-container { width: 100%; height: 100%; position: absolute; top: 0; left: 0; z-index: 10; }
        .touch-zone { touch-action: none; position: absolute; top: 0; bottom: 0; height: 100%; z-index: 20; }
        #move-zone { left: 0; width: 45%; }
        #look-zone { right: 0; width: 55%; }
        
        /* Tactical HUD Header */
        .tactical-hud {
            position: absolute; top: 16px; left: 16px; z-index: 30;
            background: rgba(15, 23, 42, 0.95); padding: 14px; border-radius: 6px;
            border: 1px solid rgba(34, 211, 238, 0.4); width: 260px;
            pointer-events: none; color: #fff; box-shadow: 0 10px 30px rgba(0,0,0,0.5);
        }
        .hud-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 6px; }
        .health-bar-bg { width: 100%; height: 6px; background: #1e293b; border-radius: 3px; overflow: hidden; margin-top: 4px; }
        .health-bar-fill { width: 100%; height: 100%; background: #10b981; transition: width 0.1s ease-out; }
        .stats-row { display: flex; justify-content: space-between; margin-top: 8px; font-size: 11px; }

        /* Draggable & Scrollable Leaderboard UI */
        .draggable-leaderboard {
            position: absolute; top: 16px; right: 16px; z-index: 35;
            width: 250px; background: rgba(15, 23, 42, 0.95);
            border: 1px solid rgba(245, 158, 11, 0.5); border-radius: 6px;
            color: #fff; box-shadow: 0 15px 35px rgba(0,0,0,0.6);
            overflow: hidden; display: flex; flex-direction: column;
        }
        .leaderboard-title-bar {
            background: rgba(30, 41, 59, 0.9); padding: 8px 12px;
            font-weight: bold; font-size: 11px; color: #f59e0b;
            cursor: move; display: flex; justify-content: space-between; align-items: center;
            border-bottom: 1px solid rgba(245, 158, 11, 0.3);
        }
        .leaderboard-scroll-area {
            max-height: 150px; overflow-y: auto; padding: 6px 0;
            scrollbar-width: thin; scrollbar-color: #f59e0b #1e293b;
        }
        .leaderboard-scroll-area::-webkit-scrollbar { width: 4px; }
        .leaderboard-scroll-area::-webkit-scrollbar-track { background: #1e293b; }
        .leaderboard-scroll-area::-webkit-scrollbar-thumb { background: #f59e0b; border-radius: 2px; }
        
        .score-entry {
            display: flex; justify-content: space-between; padding: 4px 12px;
            font-size: 11px; border-bottom: 1px solid rgba(51, 65, 85, 0.3);
        }
        .team-blue-text { color: #38bdf8; }
        .team-red-text { color: #f43f5e; }
        .team-ffa-text { color: #e2e8f0; }
        
        /* Controls & Joysticks */
        .v-joystick {
            position: absolute; width: 80px; height: 80px; background: rgba(15, 23, 42, 0.5);
            border: 2px solid rgba(34, 211, 238, 0.4); border-radius: 50%;
            display: none; align-items: center; justify-content: center; pointer-events: none;
        }
        .v-knob { width: 34px; height: 34px; border-radius: 50%; background: rgba(244, 63, 94, 0.7); border: 1px solid #f43f5e; }
        .crosshair {
            position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);
            width: 8px; height: 8px; border: 2px solid rgba(34, 211, 238, 0.9);
            border-radius: 50%; pointer-events: none; z-index: 25;
        }

        .utility-panel { position: absolute; bottom: 20px; right: 20px; z-index: 30; display: flex; gap: 8px; }
        .tactical-btn {
            background: rgba(15, 23, 42, 0.9); border: 1px solid #22d3ee; color: #fff;
            padding: 10px 14px; font-family: monospace; border-radius: 6px; font-weight: bold;
            font-size: 11px; text-transform: uppercase; letter-spacing: 0.5px;
            box-shadow: 0 4px 12px rgba(0,0,0,0.4);
        }
        .tactical-btn:active { background: #1e293b; }
        .mode-btn { border-color: #f59e0b; color: #f59e0b; }
        
        #damage-overlay {
            position: absolute; inset: 0; width: 100%; height: 100%;
            background: rgba(244, 63, 94, 0); pointer-events: none; z-index: 15;
            transition: background 0.1s ease;
        }
        #kia-overlay {
            position: absolute; inset: 0; width: 100%; height: 100%;
            background: rgba(15, 23, 42, 0.98); display: none; flex-direction: column;
            align-items: center; justify-content: center; z-index: 40; color: #f43f5e;
            font-family: monospace; text-align: center;
        }
    </style>
</head>
<body>

    <div id="canvas-container"></div>
    <div id="damage-overlay"></div>
    
    <div id="kia-overlay">
        <h1 style="letter-spacing: 4px; margin-bottom: 4px; font-size: 22px;">⚡ OPERATOR ELIMINATED</h1>
        <p style="color: #94a3b8; font-size: 11px; margin-bottom: 20px;">RE-ROUTING NEURAL NETWORK TO RE-SPAWN REPLICANT</p>
        <div style="font-size: 13px; background: #1e293b; color: #38bdf8; padding: 6px 12px; border-radius: 4px; border: 1px solid #0284c7;" id="respawn-timer">RESPAWN IN 3.0s</div>
    </div>
    
    <div class="crosshair"></div>

    <div class="tactical-hud">
        <div class="hud-header">
            <span id="hud-mode-title" style="color: #38bdf8; font-weight: bold; font-size: 11px; letter-spacing: 0.5px;">⚡ TEAM DEATHMATCH</span>
            <span id="hud-status-tag" style="background: #065f46; color: #34d399; font-size: 9px; padding: 1px 4px; font-weight: bold; border-radius: 2px;">GHOST</span>
        </div>
        <div style="border-top: 1px solid #334155; margin-bottom: 6px;"></div>
        
        <div style="display: flex; justify-content: space-between; font-size: 11px;">
            <span style="color: #94a3b8;">OPERATOR REPLICANT VITALS</span>
            <span id="player-health-percent" style="color: #10b981; font-weight: bold;">100%</span>
        </div>
        <div class="health-bar-bg">
            <div id="player-health-bar" class="health-bar-fill"></div>
        </div>

        <div class="stats-row">
            <div><span style="color: #64748b; font-size: 9px; text-transform: uppercase;">YOUR SCORE</span><br><span id="score-counter" style="color: #f59e0b; font-weight: bold; font-size: 13px;">0</span></div>
            <div style="text-align: right;"><span style="color: #64748b; font-size: 9px; text-transform: uppercase;">MATCH RULESET</span><br><span id="hud-match-rule" style="color: #22d3ee; font-size: 11px; font-weight: bold;">AUTO-FIRE TRIGGER</span></div>
        </div>
    </div>

    <div id="draggable-board" class="draggable-leaderboard">
        <div id="leaderboard-handle" class="leaderboard-title-bar">
            <span>📊 NET LOBBY ROSTER</span>
            <span style="font-size: 9px; color: #64748b;">[DRAG HUD]</span>
        </div>
        <div id="leaderboard-entries" class="leaderboard-scroll-area">
            </div>
    </div>

    <div id="move-zone" class="touch-zone">
        <div id="move-base" class="v-joystick"><div id="move-knob" class="v-knob"></div></div>
    </div>
    <div id="look-zone" class="touch-zone"></div>

    <div class="utility-panel">
        <button id="btn-toggle-mode" class="tactical-btn mode-btn">Switch Ruleset</button>
        <button id="btn-jump" class="tactical-btn">Vault / Jump</button>
    </div>

    <script>
        // --- 1. GAME ENVIRONMENT ENGINE SETUP ---
        const container = document.getElementById('canvas-container');
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0xf59e0b); 
        scene.fog = new THREE.FogExp2(0xd97706, 0.015);

        const camera = new THREE.PerspectiveCamera(65, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true, powerPreference: "high-performance" });
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.shadowMap.enabled = true;
        renderer.shadowMap.type = THREE.PCFSoftShadowMap;
        container.appendChild(renderer.domElement);

        const sunLight = new THREE.DirectionalLight(0xffedd5, 1.4); 
        sunLight.position.set(40, 60, 30);
        sunLight.castShadow = true;
        sunLight.shadow.mapSize.width = 1048; sunLight.shadow.mapSize.height = 1048;
        scene.add(sunLight);
        scene.add(new THREE.HemisphereLight(0x0284c7, 0x7c2d12, 0.8));

        // Global Structural and Match Allocation Matrices
        const collidableMeshes = []; 
        const solidObstacles = [];   
        const bulletTracers = [];
        const activeDragons = [];
        const simulationPlayers = []; 

        let matchMode = "TDM"; // Alternative: "FFA" (Free For All)
        let playerHealth = 100;
        let isPlayerDead = false;
        let weaponCooldownTime = 0;

        // Shared Lobby Database Object
        const lobbyRegistry = [
            { id: "player_local", name: "Dave (You)", score: 0, team: "BLUE", isLocal: true },
            { id: "bot_1", name: "Ghost_Recon_01", score: 0, team: "BLUE", isLocal: false },
            { id: "bot_2", name: "Spectre_Tactical", score: 0, team: "BLUE", isLocal: false },
            { id: "bot_3", name: "Apex_Hunter", score: 0, team: "BLUE", isLocal: false },
            { id: "bot_4", name: "Viper_Zero", score: 0, team: "RED", isLocal: false },
            { id: "bot_5", name: "Desert_Fox_Tac", score: 0, team: "RED", isLocal: false },
            { id: "bot_6", name: "Krypton_Striker", score: 0, team: "RED", isLocal: false },
            { id: "bot_7", name: "Outpost_Stalker", score: 0, team: "RED", isLocal: false }
        ];

        // --- 2. PROCEDURAL TEXTURES & REALISTIC MATERIALS ---
        function createNoiseTexture(baseHex, noiseHex) {
            const canvas = document.createElement('canvas'); canvas.width = 256; canvas.height = 256;
            const ctx = canvas.getContext('2d'); ctx.fillStyle = baseHex; ctx.fillRect(0, 0, 256, 256);
            for (let i = 0; i < 4000; i++) {
                ctx.fillStyle = Math.random() > 0.5 ? noiseHex : 'rgba(0,0,0,0.1)';
                ctx.fillRect(Math.random()*256, Math.random()*256, 2, 2);
            }
            const t = new THREE.CanvasTexture(canvas); t.wrapS = THREE.RepeatWrapping; t.wrapT = THREE.RepeatWrapping;
            return t;
        }
        const sandTxt = createNoiseTexture('#d97706', '#b45309'); sandTxt.repeat.set(20, 20);
        const wallTxt = createNoiseTexture('#b45309', '#78350f'); 
        const cementTxt = createNoiseTexture('#94a3b8', '#64748b'); cementTxt.repeat.set(3, 3);

        const ground = new THREE.Mesh(new THREE.PlaneGeometry(500, 500), new THREE.MeshStandardMaterial({ map: sandTxt, roughness: 0.95 }));
        ground.rotation.x = -Math.PI / 2; ground.receiveShadow = true; scene.add(ground);

        // --- 3. ARCHITECTURAL MAP SYSTEMS (CONCRETE FLOORS, UNIQUE STAIRS, ROOF OPENINGS) ---
        function buildAccessibleStructure(x, z, w, d, storyCount) {
            const floorHeight = 4.6; const wallThick = 0.4;
            const group = new THREE.Group();
            const wallMat = new THREE.MeshStandardMaterial({ map: wallTxt, roughness: 0.8, side: THREE.DoubleSide });
            const floorMat = new THREE.MeshStandardMaterial({ map: cementTxt, roughness: 0.6 }); 
            const stairMat = new THREE.MeshStandardMaterial({ color: 0x475569, roughness: 0.9 }); 
            const roofMat = new THREE.MeshStandardMaterial({ color: 0x334155, roughness: 0.8 }); 

            for (let f = 0; f < storyCount; f++) {
                const y = f * floorHeight;
                
                // Solid Polished Cement Floor Slabs
                const flMesh = new THREE.Mesh(new THREE.BoxGeometry(w, 0.15, d), floorMat);
                flMesh.position.set(w/2, y, d/2); flMesh.receiveShadow = true;
                group.add(flMesh); collidableMeshes.push(flMesh);

                // Back Wall Core Block
                const bw = new THREE.Mesh(new THREE.BoxGeometry(w, floorHeight, wallThick), wallMat);
                bw.position.set(w/2, y + floorHeight/2, wallThick/2); group.add(bw); solidObstacles.push(bw);

                // Left Wall segments housing true structural window cavities
                const wW = 2.6; const wH = 1.5; const wSill = 1.1;
                const lw1 = new THREE.Mesh(new THREE.BoxGeometry(wallThick, floorHeight, (d - wW)/2), wallMat);
                lw1.position.set(wallThick/2, y + floorHeight/2, (d - wW)/4);
                const lw2 = new THREE.Mesh(new THREE.BoxGeometry(wallThick, floorHeight, (d - wW)/2), wallMat);
                lw2.position.set(wallThick/2, y + floorHeight/2, d - (d - wW)/4);
                const lSill = new THREE.Mesh(new THREE.BoxGeometry(wallThick, wSill, wW), wallMat);
                lSill.position.set(wallThick/2, y + wSill/2, d/2);
                const lHead = new THREE.Mesh(new THREE.BoxGeometry(wallThick, floorHeight - (wSill + wH), wW), wallMat);
                lHead.position.set(wallThick/2, y + floorHeight - (floorHeight - (wSill + wH))/2, d/2);
                group.add(lw1, lw2, lSill, lHead); solidObstacles.push(lw1, lw2, lSill, lHead);

                // Right Wall Core Block
                const rw = new THREE.Mesh(new THREE.BoxGeometry(wallThick, floorHeight, d), wallMat);
                rw.position.set(w - wallThick/2, y + floorHeight/2, d/2); group.add(rw); solidObstacles.push(rw);

                // Front Wall Segment Entry Openings
                if (f === 0) {
                    const dW = 2.6; const dH = 3.5;
                    const fw1 = new THREE.Mesh(new THREE.BoxGeometry((w - dW)/2, floorHeight, wallThick), wallMat);
                    fw1.position.set((w - dW)/4, y + floorHeight/2, d - wallThick/2);
                    const fw2 = new THREE.Mesh(new THREE.BoxGeometry((w - dW)/2, floorHeight, wallThick), wallMat);
                    fw2.position.set(w - (w - dW)/4, y + floorHeight/2, d - wallThick/2);
                    const fHead = new THREE.Mesh(new THREE.BoxGeometry(dW, floorHeight - dH, wallThick), wallMat);
                    fHead.position.set(w/2, y + floorHeight - (floorHeight - dH)/2, d - wallThick/2);
                    group.add(fw1, fw2, fHead); solidObstacles.push(fw1, fw2, fHead);
                } else {
                    const fws = new THREE.Mesh(new THREE.BoxGeometry(w, floorHeight, wallThick), wallMat);
                    fws.position.set(w/2, y + floorHeight/2, d - wallThick/2); group.add(fws); solidObstacles.push(fws);
                }

                // Chiseled Gray Slate Granite Steps Component Loop
                if (storyCount > 1 && f < storyCount) {
                    const steps = 16; const stW = 1.8;
                    for (let s = 0; s < steps; s++) {
                        const p = s / steps;
                        const stMesh = new THREE.Mesh(new THREE.BoxGeometry(stW, 0.28, (d * 0.75)/steps), stairMat);
                        stMesh.position.set(w - stW/2 - wallThick, y + (p * floorHeight) + 0.14, d * 0.12 + (p * d * 0.75));
                        group.add(stMesh); collidableMeshes.push(stMesh);
                    }
                    if(f === 0) { flMesh.scale.setX(0.65); flMesh.position.setX((w * 0.65)/2); }
                }
            }

            // High Precision Unblocked Roof Assembly Outlets
            const rfW = w * 0.65;
            const rf1 = new THREE.Mesh(new THREE.BoxGeometry(rfW, 0.2, d + 0.2), roofMat);
            rf1.position.set(rfW/2, storyCount * floorHeight, d/2);
            const rf2 = new THREE.Mesh(new THREE.BoxGeometry(w - rfW, 0.2, d * 0.25), roofMat);
            rf2.position.set(w - (w - rfW)/2, storyCount * floorHeight, d * 0.875);
            group.add(rf1, rf2); collidableMeshes.push(rf1, rf2); solidObstacles.push(rf1, rf2);

            group.position.set(x, 0, z); scene.add(group);
        }

        // Deploy Tactical Network Arenas
        buildAccessibleStructure(-6, -20, 12, 12, 2);
        buildAccessibleStructure(16, -18, 10, 10, 1);
        buildAccessibleStructure(-26, -32, 12, 12, 2);

        // --- 4. LIFELIKE APEX DRAGON AGENTS ---
        function spawnLifelikeDragon(x, z) {
            const drag = new THREE.Group();
            const dMat = new THREE.MeshStandardMaterial({ color: 0xb45309, roughness: 0.6 });
            const vMat = new THREE.MeshStandardMaterial({ color: 0xf59e0b, roughness: 0.4 });
            const mMat = new THREE.MeshStandardMaterial({ color: 0x991b1b, roughness: 0.7, side: THREE.DoubleSide, transparent:true, opacity:0.85 });
            const bMat = new THREE.MeshStandardMaterial({ color: 0x78350f });

            const torso = new THREE.Mesh(new THREE.BoxGeometry(1.5, 1.2, 3.2), dMat); drag.add(torso);
            const neck = new THREE.Mesh(new THREE.BoxGeometry(0.6, 0.6, 0.9), dMat); neck.position.set(0, 0.6, -1.5); torso.add(neck);
            const head = new THREE.Mesh(new THREE.BoxGeometry(0.7, 0.6, 1.4), dMat); head.position.set(0, 0.2, -0.7); neck.add(head);
            
            const wingL = new THREE.Group(); wingL.position.set(-0.75, 0.4, -0.2);
            const wBoneL = new THREE.Mesh(new THREE.BoxGeometry(2.0, 0.12, 0.2), bMat); wBoneL.geometry.translate(-1.0, 0, 0); wingL.add(wBoneL);
            const wMemL = new THREE.Mesh(new THREE.BoxGeometry(4.0, 0.02, 2.0), mMat); wMemL.position.set(-2.0, 0, 0.9); wingL.add(wMemL);
            torso.add(wingL);

            const wingR = new THREE.Group(); wingR.position.set(0.75, 0.4, -0.2);
            const wBoneR = new THREE.Mesh(new THREE.BoxGeometry(2.0, 0.12, 0.2), bMat); wBoneR.geometry.translate(1.0, 0, 0); wingR.add(wBoneR);
            const wMemR = new THREE.Mesh(new THREE.BoxGeometry(4.0, 0.02, 2.0), mMat); wMemR.position.set(2.0, 0, 0.9); wingR.add(wMemR);
            torso.add(wingR);

            drag.position.set(x, 22, z); scene.add(drag);
            drag.userData = { 
                oX: x, oZ: z, rad: 25+Math.random()*10, speed: 0.3+Math.random()*0.2, wL: wingL, wR: wingR,
                hitbox: torso, isDead: false, fall: 0, health: 100, lastFire: 0 
            };
            activeDragons.push(drag);
        }
        spawnLifelikeDragon(-20, -35); spawnLifelikeDragon(20, -45);

        // --- 5. NETWORK SIMULATION OPERATORS MOCKING LOGIC ---
        function createNetworkAgent(profile) {
            const agentGroup = new THREE.Group();
            const colorCode = profile.team === "BLUE" ? 0x0284c7 : 0xf43f5e;
            
            // Operator Mesh Rig
            const body = new THREE.Mesh(new THREE.CylinderGeometry(0.35, 0.35, 1.7, 8), new THREE.MeshStandardMaterial({ color: colorCode, roughness: 0.5 }));
            body.position.y = 0.85; body.castShadow = true; agentGroup.add(body);
            
            const visor = new THREE.Mesh(new THREE.BoxGeometry(0.5, 0.15, 0.4), new THREE.MeshBasicMaterial({ color: 0x22d3ee }));
            visor.position.set(0, 0.6, -0.2); body.add(visor);

            const sX = (Math.random() - 0.5) * 80, sZ = (Math.random() - 0.5) * 80;
            agentGroup.position.set(sX, 0, sZ); scene.add(agentGroup);

            agentGroup.userData = {
                id: profile.id, name: profile.name, team: profile.team, health: 100, isDead: false,
                hitbox: body, targetHeadingX: 0, targetHeadingZ: 0, tacticalTimer: 0, fireCooldown: 0
            };
            simulationPlayers.push(agentGroup);
        }

        // Instantiate Sim-Network Assets
        lobbyRegistry.forEach(p => { if(!p.isLocal) createNetworkAgent(p); });

        // --- 6. USER WEAPON PROFILE ARRAYS ---
        const tacticalGun = new THREE.Group();
        tacticalGun.add(new THREE.Mesh(new THREE.BoxGeometry(0.04, 0.06, 0.38), new THREE.MeshStandardMaterial({ color: 0x0f172a })));
        const gunBarrel = new THREE.Mesh(new THREE.CylinderGeometry(0.012, 0.012, 0.32), new THREE.MeshStandardMaterial({ color: 0x22d3ee }));
        gunBarrel.position.set(0, 0.015, -0.36); gunBarrel.geometry.rotateX(Math.PI/2);
        tacticalGun.add(gunBarrel); scene.add(tacticalGun);

        // --- 7. DYNAMIC LEADERBOARD RENDERING & INTERACTION LAYER ---
        const dragContainer = document.getElementById('draggable-board');
        const dragHandle = document.getElementById('leaderboard-handle');
        let isDragging = false, dragStartX = 0, dragStartY = 0, winStartX = 0, winStartY = 0;

        // Mouse Drag Mechanics
        dragHandle.addEventListener('mousedown', (e) => {
            isDragging = true; dragStartX = e.clientX; dragStartY = e.clientY;
            winStartX = dragContainer.offsetLeft; winStartY = dragContainer.offsetTop;
            e.preventDefault();
        });
        window.addEventListener('mousemove', (e) => {
            if (!isDragging) return;
            dragContainer.style.left = `${winStartX + (e.clientX - dragStartX)}px`;
            dragContainer.style.top = `${winStartY + (e.clientY - dragStartY)}px`;
            dragContainer.style.right = 'auto';
        });
        window.addEventListener('mouseup', () => isDragging = false);

        // Touch Drag Mechanics
        dragHandle.addEventListener('touchstart', (e) => {
            isDragging = true; const t = e.touches[0];
            dragStartX = t.clientX; dragStartY = t.clientY;
            winStartX = dragContainer.offsetLeft; winStartY = dragContainer.offsetTop;
        }, { passive: true });
        window.addEventListener('touchmove', (e) => {
            if (!isDragging) return; const t = e.touches[0];
            dragContainer.style.left = `${winStartX + (t.clientX - dragStartX)}px`;
            dragContainer.style.top = `${winStartY + (t.clientY - dragStartY)}px`;
            dragContainer.style.right = 'auto';
        }, { passive: true });
        window.addEventListener('touchend', () => isDragging = false);

        function refreshLeaderboardHTML() {
            // Re-sort current lobby statistics by total aggregate kills
            lobbyRegistry.sort((a, b) => b.score - a.score);
            const area = document.getElementById('leaderboard-entries');
            area.innerHTML = '';
            
            lobbyRegistry.forEach((p, idx) => {
                const row = document.createElement('div'); row.className = 'score-entry';
                let teamStyle = 'team-ffa-text';
                if (matchMode === "TDM") teamStyle = p.team === "BLUE" ? 'team-blue-text' : 'team-red-text';
                
                row.innerHTML = `<span class="${teamStyle}">#${idx+1} ${p.name}</span><span style="font-weight:bold;">${p.score} KLS</span>`;
                area.appendChild(row);
            });
        }
        refreshLeaderboardHTML();

        // Ruleset Toggle System
        document.getElementById('btn-toggle-mode').addEventListener('touchstart', (e) => {
            e.stopPropagation(); e.preventDefault();
            matchMode = matchMode === "TDM" ? "FFA" : "TDM";
            document.getElementById('hud-mode-title').innerText = matchMode === "TDM" ? "⚡ TEAM DEATHMATCH" : "⚡ FREE FOR ALL";
            document.getElementById('hud-status-tag').innerText = matchMode === "TDM" ? "GHOST" : "ROGUE";
            refreshLeaderboardHTML();
        });

        // --- 8. TOUCH INTERFACE MOTION MATRICES ---
        camera.position.set(0, 1.8, 12);
        let cameraYaw = 0, cameraPitch = 0, playerVertVel = 0, isGrounded = true;
        const currentGunPos = new THREE.Vector3(), targetGunPos = new THREE.Vector3(), hipOffset = new THREE.Vector3(0.15, -0.18, -0.42);

        let leftActive = false, leftId = null, leftX = 0, leftY = 0, moveAngle = 0, moveMag = 0;
        const uiBase = document.getElementById('move-base'), uiKnob = document.getElementById('move-knob');
        let rightActive = false, rightId = null, rLastX = null, rLastY = null;

        document.getElementById('move-zone').addEventListener('touchstart', (e) => {
            if (isPlayerDead) return; const t = e.changedTouches[0]; leftId = t.identifier; leftActive = true;
            leftX = t.clientX; leftY = t.clientY; uiBase.style.left = `${leftX - 40}px`; uiBase.style.top = `${leftY - 40}px`;
            uiBase.style.display = 'flex'; uiKnob.style.transform = 'translate(0px,0px)';
        }, { passive: true });

        document.getElementById('look-zone').addEventListener('touchstart', (e) => {
            if (isPlayerDead || e.target.closest('button') || e.target.closest('#draggable-board')) return;
            const t = e.changedTouches[0]; rightId = t.identifier; rightActive = true;
            rLastX = t.clientX; rLastY = t.clientY;
        }, { passive: true });

        window.addEventListener('touchmove', (e) => {
            if (isPlayerDead) return;
            for (let i = 0; i < e.touches.length; i++) {
                const t = e.touches[i];
                if (t.identifier === leftId && leftActive) {
                    const dX = t.clientX - leftX, dY = t.clientY - leftY;
                    const dist = Math.min(Math.sqrt(dX*dX + dY*dY), 40);
                    moveAngle = Math.atan2(-dX, -dY); moveMag = dist / 40;
                    uiKnob.style.transform = `translate(${Math.sin(moveAngle)*-dist}px, ${Math.cos(moveAngle)*-dist}px)`;
                }
                if (t.identifier === rightId && rightActive) {
                    if (rLastX !== null) {
                        cameraYaw -= ((t.clientX - rLastX) / window.innerWidth) * 5.0;
                        cameraPitch -= ((t.clientY - rLastY) / window.innerHeight) * 5.0;
                        cameraPitch = Math.max(-Math.PI/2.4, Math.min(Math.PI/2.4, cameraPitch));
                    }
                    rLastX = t.clientX; rLastY = t.clientY;
                }
            }
        }, { passive: true });

        const endTouch = (e) => {
            for (let i = 0; i < e.changedTouches.length; i++) {
                const id = e.changedTouches[i].identifier;
                if (id === leftId) { leftActive = false; moveMag = 0; uiBase.style.display = 'none'; }
                if (id === rightId) { rightActive = false; rLastX = null; rLastY = null; }
            }
        };
        window.addEventListener('touchend', endTouch); window.addEventListener('touchcancel', endTouch);
        document.getElementById('btn-jump').addEventListener('touchstart', (e) => {
            e.stopPropagation(); if (isGrounded && !isPlayerDead) { playerVertVel = 5.5; isGrounded = false; }
        });

        // --- 9. TARGET ACQUISITION & TEAM TRACER INTERSECTIONS ---
        const hitscanRay = new THREE.Raycaster();
        const staticEnvRay = new THREE.Raycaster();

        function commitLocalWeaponFire(time) {
            if (time - weaponCooldownTime < 0.12 || isPlayerDead) return;

            const lookDir = new THREE.Vector3(0, 0, -1).applyQuaternion(camera.quaternion).normalize();
            
            // Process architecture blocks to establish absolute ray occlusion distances
            staticEnvRay.set(camera.position, lookDir);
            const wallsIntersect = staticEnvRay.intersectObjects(solidObstacles);
            let blockDist = Infinity;
            if (wallsIntersect.length > 0) blockDist = wallsIntersect[0].distance;

            hitscanRay.set(camera.position, lookDir);

            // Assemble list of viable targets matching the active operational ruleset
            const targetPool = [];
            simulationPlayers.forEach(bot => {
                if (bot.userData.isDead) return;
                // In Team Deathmatch, explicitly skip firing at teammates on your blue side
                if (matchMode === "TDM" && bot.userData.team === "BLUE") return;
                targetPool.push(bot.userData.hitbox);
            });
            activeDragons.forEach(d => { if(!d.userData.isDead) targetPool.push(d.userData.hitbox); });

            const structuralHits = hitscanRay.intersectObjects(targetPool);

            // TARGET LOCK RULE: Operator weapon loop strictly unlocks ONLY if reticle overlays an un-occluded enemy target
            if (structuralHits.length > 0 && structuralHits[0].distance < blockDist) {
                weaponCooldownTime = time;
                tacticalGun.position.z += 0.04;

                // Render High Brightness Cyan Tracer Beam Lines
                const barrelPoint = new THREE.Vector3(0, 0.015, -0.45).applyMatrix4(tacticalGun.matrixWorld);
                const cyGeo = new THREE.CylinderGeometry(0.015, 0.015, 0.4); cyGeo.rotateX(Math.PI/2);
                const trMesh = new THREE.Mesh(cyGeo, new THREE.MeshBasicMaterial({ color: 0x22d3ee }));
                trMesh.position.copy(barrelPoint); trMesh.lookAt(barrelPoint.clone().add(lookDir.clone().multiplyScalar(80)));
                scene.add(trMesh); bulletTracers.push({ mesh: trMesh, delta: lookDir.clone().multiplyScalar(130.0), life: 0.7 });

                const hitObj = structuralHits[0].object;

                // Test hits against simulated bot layers
                simulationPlayers.forEach(bot => {
                    if (bot.userData.hitbox === hitObj) {
                        bot.userData.health -= 34;
                        if(bot.userData.health <= 0 && !bot.userData.isDead) {
                            bot.userData.isDead = true; 
                            const reg = lobbyRegistry.find(r => r.id === bot.userData.id);
                            if(reg) { reg.score++; refreshLeaderboardHTML(); }
                            killCount++; document.getElementById('score-counter').innerText = killCount;
                            const mainReg = lobbyRegistry.find(r => r.isLocal); if(mainReg) mainReg.score = killCount;
                            refreshLeaderboardHTML();
                        }
                    }
                });

                // Test hits against dragon layers
                activeDragons.forEach(drag => {
                    if (drag.userData.hitbox === hitObj) {
                        drag.userData.health -= 25;
                        if (drag.userData.health <= 0) {
                            drag.userData.isDead = true; killCount++;
                            document.getElementById('score-counter').innerText = killCount;
                            const mainReg = lobbyRegistry.find(r => r.isLocal); if(mainReg) mainReg.score = killCount;
                            refreshLeaderboardHTML();
                        }
                    }
                });
            }
        }

        function triggerPlayerKilled() {
            isPlayerDead = true; playerHealth = 0; leftActive = false; rightActive = false; uiBase.style.display = 'none';
            document.getElementById('player-health-percent').innerText = "0%";
            document.getElementById('player-health-bar').style.width = "0%";
            document.getElementById('kia-overlay').style.display = 'flex';

            let count = 3.0;
            const timer = setInterval(() => {
                count -= 0.1;
                if (count <= 0) { 
                    clearInterval(timer); isPlayerDead = false; playerHealth = 100;
                    document.getElementById('player-health-percent').innerText = "100%";
                    document.getElementById('player-health-bar').style.width = "100%";
                    document.getElementById('kia-overlay').style.display = 'none';
                    camera.position.set((Math.random() - 0.5) * 30, 1.8, (Math.random() - 0.5) * 30);
                } else {
                    document.getElementById('respawn-timer').innerText = `RESPAWN IN ${Math.max(0, count).toFixed(1)}s`;
                }
            }, 100);
        }

        // --- 10. REAL-TIME MULTIPLAYER SIMULATION ENGINE & COLLISION LOOP ---
        const clock = new THREE.Clock();

        function animate() {
            requestAnimationFrame(animate);
            const dt = clock.getDelta(); const elapsed = clock.getElapsedTime();

            camera.quaternion.setFromEuler(new THREE.Euler(cameraPitch, cameraYaw, 0, 'YXZ'));

            // Core Kinematics Gravitational Systems
            playerVertVel -= 14.5 * dt; camera.position.y += playerVertVel * dt;
            let ceilingY = 1.8;
            const ray = new THREE.Raycaster(new THREE.Vector3(camera.position.x, camera.position.y+0.1, camera.position.z), new THREE.Vector3(0,-1,0));
            const floors = ray.intersectObjects(collidableMeshes);
            if (floors.length > 0) ceilingY = floors[0].point.y + 1.8;
            if (camera.position.y <= ceilingY) { camera.position.y = ceilingY; playerVertVel = 0; isGrounded = true; }

            // Spatial Translation Vector Splitting
            if (leftActive && moveMag > 0 && !isPlayerDead) {
                const heading = cameraYaw + moveAngle;
                const sX = Math.sin(heading) * (5.0 * moveMag) * dt, sZ = Math.cos(heading) * (5.0 * moveMag) * dt;
                const nextPos = camera.position.clone().setX(camera.position.x - sX).setZ(camera.position.z - sZ);
                
                let isBlocked = false;
                solidObstacles.forEach(m => {
                    const box = new THREE.Box3().setFromObject(m).expandByScalar(0.4);
                    if (nextPos.y >= box.min.y && nextPos.y <= box.max.y + 0.5 && box.containsPoint(nextPos)) isBlocked = true;
                });
                if (!isBlocked) { camera.position.x -= sX; camera.position.z -= sZ; }
            }

            if (!isPlayerDead) commitLocalWeaponFire(elapsed);

            // SIMULATED MULTIPLAYER BEHAVIOR TREES LAYER
            simulationPlayers.forEach(bot => {
                const bData = bot.userData;
                if (bData.isDead) {
                    bot.position.y = -10; // Hide underground on death
                    bData.tacticalTimer += dt;
                    if (bData.tacticalTimer > 3.0) { // Re-spawn bot loop
                        bData.isDead = false; bData.health = 100; bData.tacticalTimer = 0;
                        bot.position.set((Math.random() - 0.5) * 70, 0, (Math.random() - 0.5) * 70);
                    }
                    return;
                }

                // Simulate bot navigation loops
                bData.tacticalTimer -= dt;
                if (bData.tacticalTimer <= 0) {
                    bData.tacticalTimer = 2.0 + Math.random() * 3.0;
                    bData.targetHeadingX = (Math.random() - 0.5) * 0.4;
                    bData.targetHeadingZ = (Math.random() - 0.5) * 0.4;
                }

                // Check environment collisions for simulated players
                const nextBotPos = bot.position.clone().add(new THREE.Vector3(bData.targetHeadingX, 0, bData.targetHeadingZ).multiplyScalar(dt * 10));
                let botBlocked = false;
                solidObstacles.forEach(m => {
                    if (new THREE.Box3().setFromObject(m).expandByScalar(0.5).containsPoint(nextBotPos)) botBlocked = true;
                });
                if (!botBlocked) { bot.position.copy(nextBotPos); }

                // Simulated combat interaction ticks
                bData.fireCooldown -= dt;
                if (bData.fireCooldown <= 0) {
                    bData.fireCooldown = 0.8 + Math.random() * 1.2;
                    
                    // Determine enemy engagement logic profiles
                    let targetFound = false;
                    const enemyVec = new THREE.Vector3();

                    if (matchMode === "TDM" && bData.team === "RED") {
                        enemyVec.copy(camera.position); targetFound = !isPlayerDead;
                    } else if (matchMode === "FFA") {
                        if (Math.random() > 0.5 && !isPlayerDead) { enemyVec.copy(camera.position); targetFound = true; }
                    }

                    if (targetFound && bot.position.distanceTo(enemyVec) < 45) {
                        // Spawn physical neon crimson bot tracers into simulation grid
                        const bGeo = new THREE.CylinderGeometry(0.015, 0.015, 0.4); bGeo.rotateX(Math.PI/2);
                        const bTracer = new THREE.Mesh(bGeo, new THREE.MeshBasicMaterial({ color: 0xf43f5e }));
                        bTracer.position.copy(bot.position).setY(0.9);
                        
                        const dir = new THREE.Vector3().copy(enemyVec).sub(bot.position).normalize();
                        bTracer.lookAt(bot.position.clone().add(dir.clone().multiplyScalar(20)));
                        scene.add(bTracer); bulletTracers.push({ mesh: bTracer, delta: dir.multiplyScalar(100.0), life: 0.6 });

                        // If user is inside trajectory radius, register hit event
                        if (enemyVec.equals(camera.position) && Math.random() > 0.4 && !isPlayerDead) {
                            playerHealth = Math.max(0, playerHealth - 15);
                            document.getElementById('player-health-percent').innerText = playerHealth + "%";
                            document.getElementById('player-health-bar').style.width = playerHealth + "%";
                            document.getElementById('damage-overlay').style.background = "rgba(244, 63, 94, 0.4)";
                            setTimeout(() => { document.getElementById('damage-overlay').style.background = "rgba(244, 63, 94, 0)"; }, 80);
                            if (playerHealth <= 0) triggerPlayerKilled();
                        }
                    }
                }
            });

            // Lifelike Apex Dragon Animations
            activeDragons.forEach(dragon => {
                const dData = dragon.userData;
                if (!dData.isDead) {
                    const rotAngle = elapsed * dData.speed;
                    dragon.position.set(dData.oX + Math.sin(rotAngle)*dData.rad, 22 + Math.sin(elapsed)*1.5, dData.oZ + Math.cos(rotAngle)*dData.rad);
                    dragon.rotation.y = rotAngle - Math.PI/2;
                    const flap = Math.sin(elapsed * 6.0);
                    dData.wL.rotation.z = flap * 0.45; dData.wR.rotation.z = -flap * 0.45;
                } else {
                    if (dragon.position.y > 0.4) { dData.fall += 9.8 * dt; dragon.position.y -= dData.fall * dt; }
                    else {
                        dragon.position.y = 0.4;
                        setTimeout(() => {
                            dData.health = 100; dData.isDead = false; dData.fall = 0;
                            dragon.position.set((Math.random()-0.5)*50, 22, -30-Math.random()*20);
                        }, 7000);
                    }
                }
            });

            // Clean & cycle flying bullet tracers
            for (let i = bulletTracers.length - 1; i >= 0; i--) {
                const b = bulletTracers[i]; b.mesh.position.add(b.delta.clone().multiplyScalar(dt));
                b.life -= dt; if (b.life <= 0) { scene.remove(b.mesh); bulletTracers.splice(i, 1); }
            }

            // Gun sway interpolation matrix
            let sX = Math.sin(elapsed * 1.4) * 0.002, sY = Math.cos(elapsed * 2.6) * 0.001;
            if (leftActive && moveMag > 0) { sX = Math.sin(elapsed * 6.5) * 0.006; sY = Math.abs(Math.sin(elapsed * 6.5)) * 0.006; }
            targetGunPos.copy(hipOffset).add(new THREE.Vector3(sX, sY, isPlayerDead ? -1.0 : 0));
            currentGunPos.lerp(targetGunPos, dt * 12.0);
            tacticalGun.position.copy(camera.position).add(currentGunPos.clone().applyQuaternion(camera.quaternion));
            tacticalGun.quaternion.copy(camera.quaternion);

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
