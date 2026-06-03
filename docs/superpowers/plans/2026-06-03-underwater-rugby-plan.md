# Underwater Rugby Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a 3D underwater rugby game with cyberpunk visuals, AI opponents, and player-switching mechanics. First team to score 5 goals wins.

**Architecture:** Single HTML file with Three.js for 3D rendering. Game loop runs at 60fps. Physics handles 3D collision detection with underwater resistance. AI uses state machine for player behavior.

**Tech Stack:** Three.js (CDN), Web Audio API, JavaScript ES6+

---

## File Structure

```
underwater-rugby/
├── index.html          # Main game file (single file contains all code)
└── docs/
    └── superpowers/
        └── specs/
            └── 2026-06-03-underwater-rugby-design.md
```

---

## Task 1: Project Setup and Three.js Scene

**Files:**
- Create: `index.html`

- [ ] **Step 1: Create HTML skeleton with Three.js CDN**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Underwater Rugby</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            background: #0a0a12;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            font-family: 'Courier New', monospace;
            overflow: hidden;
        }
        #game-container {
            position: relative;
            width: 100vw;
            height: 100vh;
        }
        canvas { display: block; }
        #ui {
            position: absolute;
            top: 20px;
            left: 0;
            right: 0;
            display: flex;
            justify-content: center;
            gap: 60px;
            color: #0ff;
            font-size: 32px;
            text-shadow: 0 0 10px #0ff, 0 0 20px #0ff;
            pointer-events: none;
        }
        #energy-bar {
            position: absolute;
            bottom: 20px;
            left: 50%;
            transform: translateX(-50%);
            width: 200px;
            height: 20px;
            background: rgba(0, 255, 255, 0.2);
            border: 2px solid #0ff;
            border-radius: 10px;
            overflow: hidden;
        }
        #energy-fill {
            height: 100%;
            background: linear-gradient(90deg, #0ff, #0f0);
            width: 100%;
            transition: width 0.1s;
        }
        #controls-hint {
            position: absolute;
            bottom: 60px;
            left: 50%;
            transform: translateX(-50%);
            color: #0ff;
            font-size: 14px;
            text-shadow: 0 0 5px #0ff;
            text-align: center;
        }
        #player-indicator {
            position: absolute;
            top: 70px;
            left: 50%;
            transform: translateX(-50%);
            color: #ff0;
            font-size: 18px;
            text-shadow: 0 0 10px #ff0;
        }
    </style>
</head>
<body>
    <div id="game-container">
        <div id="ui">
            <div id="score-a">CYAN: 0</div>
            <div id="score-b">MAGENTA: 0</div>
        </div>
        <div id="player-indicator">Controlling: Player 1</div>
        <div id="energy-bar"><div id="energy-fill"></div></div>
        <div id="controls-hint">WASD: Move | U: Up | I: Down | TAB: Switch Player | SPACE: Sprint</div>
    </div>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script>
// Game constants
const FIELD_RADIUS = 15;
const GOAL_WIDTH = 6;
const GOAL_DEPTH = 3;
const BALL_RADIUS = 0.5;
const PLAYER_RADIUS = 0.8;
const TEAM_A_COLOR = 0x00ffff;
const TEAM_B_COLOR = 0xff00ff;

// Game state
let scoreA = 0;
let scoreB = 0;
let gameRunning = true;
let controlledPlayerIndex = 0;

// Energy system
let energy = 100;
const MAX_ENERGY = 100;
const ENERGY_DRAIN = 30; // per second while sprinting
const ENERGY_REGEN = 15; // per second while not sprinting

// Three.js components
let scene, camera, renderer;
let ball, players = [];
let playerBodies = []; // Physics representations

// Input state
const keys = {};

        document.addEventListener('keydown', e => {
            keys[e.code] = true;
            if (e.code === 'Tab') e.preventDefault();
        });
        document.addEventListener('keyup', e => keys[e.code] = false);

        function init() {
            // Scene
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x0a0a12);
            scene.fog = new THREE.Fog(0x0a0a12, 20, 50);

            // Camera (isometric view)
            camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 100);
            camera.position.set(20, 25, 20);
            camera.lookAt(0, 0, 0);

            // Renderer
            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setPixelRatio(window.devicePixelRatio);
            document.getElementById('game-container').appendChild(renderer.domElement);

            // Lighting
            const ambientLight = new THREE.AmbientLight(0x404060, 0.5);
            scene.add(ambientLight);

            const pointLight1 = new THREE.PointLight(0x00ffff, 1, 50);
            pointLight1.position.set(-10, 10, -10);
            scene.add(pointLight1);

            const pointLight2 = new THREE.PointLight(0xff00ff, 1, 50);
            pointLight2.position.set(10, 10, 10);
            scene.add(pointLight2);

            // Create field (hexagonal)
            createField();

            // Create ball
            createBall();

            // Create players (3 per team)
            createPlayers();

            // Create particles (bubbles)
            createBubbles();

            // Handle resize
            window.addEventListener('resize', onWindowResize);

            // Start game loop
            gameLoop();
        }

        function createField() {
            // Hexagonal field boundary (glowing lines)
            const hexShape = new THREE.Shape();
            for (let i = 0; i < 6; i++) {
                const angle = (i / 6) * Math.PI * 2 - Math.PI / 2;
                const x = Math.cos(angle) * FIELD_RADIUS;
                const z = Math.sin(angle) * FIELD_RADIUS;
                if (i === 0) hexShape.moveTo(x, z);
                else hexShape.lineTo(x, z);
            }
            hexShape.closePath();

            const points = hexShape.getPoints();
            const geometry = new THREE.BufferGeometry().setFromPoints(
                points.map(p => new THREE.Vector3(p.x, 0, p.y))
            );
            const material = new THREE.LineBasicMaterial({
                color: 0x00ffff,
                linewidth: 2
            });
            const hexLine = new THREE.LineLoop(geometry, material);
            hexLine.position.y = 0.1;
            scene.add(hexLine);

            // Goal areas (cyan team left, magenta team right)
            const goalGeometry = new THREE.BoxGeometry(GOAL_WIDTH, 3, GOAL_DEPTH);
            const cyanMaterial = new THREE.MeshBasicMaterial({
                color: 0x00ffff,
                transparent: true,
                opacity: 0.3
            });
            const magentaMaterial = new THREE.MeshBasicMaterial({
                color: 0xff00ff,
                transparent: true,
                opacity: 0.3
            });

            // Left goal (cyan)
            const leftGoal = new THREE.Mesh(goalGeometry, cyanMaterial);
            leftGoal.position.set(-FIELD_RADIUS - GOAL_DEPTH/2, 1.5, 0);
            scene.add(leftGoal);

            // Right goal (magenta)
            const rightGoal = new THREE.Mesh(goalGeometry, magentaMaterial);
            rightGoal.position.set(FIELD_RADIUS + GOAL_DEPTH/2, 1.5, 0);
            scene.add(rightGoal);
        }

        function createBall() {
            const geometry = new THREE.SphereGeometry(BALL_RADIUS, 16, 16);
            const material = new THREE.MeshBasicMaterial({
                color: 0xffffff,
                emissive: 0xffffff,
                emissiveIntensity: 0.5
            });
            ball = new THREE.Mesh(geometry, material);
            ball.position.set(0, 1, 0);
            ball.userData.velocity = new THREE.Vector3();
            scene.add(ball);

            // Glow effect
            const glowGeometry = new THREE.SphereGeometry(BALL_RADIUS * 1.5, 16, 16);
            const glowMaterial = new THREE.MeshBasicMaterial({
                color: 0x00ffff,
                transparent: true,
                opacity: 0.2
            });
            const glow = new THREE.Mesh(glowGeometry, glowMaterial);
            ball.add(glow);
        }

        function createPlayers() {
            const positions = [
                { team: 'A', x: -8, y: 1, z: 0 },    // Cyan team
                { team: 'A', x: -4, y: 2, z: 3 },
                { team: 'A', x: -4, y: 1, z: -3 },
                { team: 'B', x: 8, y: 1, z: 0 },     // Magenta team
                { team: 'B', x: 4, y: 2, z: 3 },
                { team: 'B', x: 4, y: 1, z: -3 },
            ];

            positions.forEach((pos, i) => {
                const color = pos.team === 'A' ? TEAM_A_COLOR : TEAM_B_COLOR;
                const geometry = new THREE.SphereGeometry(PLAYER_RADIUS, 16, 16);
                const material = new THREE.MeshBasicMaterial({ color });
                const player = new THREE.Mesh(geometry, material);
                player.position.set(pos.x, pos.y, pos.z);
                player.userData = {
                    team: pos.team,
                    velocity: new THREE.Vector3(),
                    controlled: i === 0
                };
                scene.add(player);
                players.push(player);

                // Glow ring for controlled player
                const ringGeometry = new THREE.TorusGeometry(PLAYER_RADIUS * 1.3, 0.05, 8, 32);
                const ringMaterial = new THREE.MeshBasicMaterial({ color: 0xffff00 });
                const ring = new THREE.Mesh(ringGeometry, ringMaterial);
                ring.rotation.x = Math.PI / 2;
                ring.visible = i === 0;
                player.add(ring);
                player.userData.ring = ring;
            });
        }

        function createBubbles() {
            const bubbleGeometry = new THREE.SphereGeometry(0.1, 8, 8);
            const bubbleMaterial = new THREE.MeshBasicMaterial({
                color: 0x00ffff,
                transparent: true,
                opacity: 0.5
            });

            for (let i = 0; i < 50; i++) {
                const bubble = new THREE.Mesh(bubbleGeometry, bubbleMaterial.clone());
                bubble.position.set(
                    (Math.random() - 0.5) * FIELD_RADIUS * 2,
                    Math.random() * 10,
                    (Math.random() - 0.5) * FIELD_RADIUS * 2
                );
                bubble.userData.speed = 0.5 + Math.random() * 1;
                bubble.userData.wobble = Math.random() * Math.PI * 2;
                scene.add(bubble);
            }
        }

        function onWindowResize() {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        }

        // Start
        init();
    </script>
</body>
</html>
```

- [ ] **Step 2: Run to verify basic Three.js scene loads**

Open: `open /Users/gaostone/game/underwater-rugby/index.html`

Expected: Browser opens with black background, cyan hexagonal field outline, floating white ball, 6 colored players, UI shows scores.

---

## Task 2: Player Movement System

**Files:**
- Modify: `index.html` (add input handling and movement)

- [ ] **Step 1: Add keyboard input constants**

Add at top of script section after game constants:

```javascript
// Movement constants
const PLAYER_SPEED = 5;
const SPRINT_MULTIPLIER = 2;
const VERTICAL_SPEED = 3;
const UNDERWATER_DRAG = 0.95;
```

- [ ] **Step 2: Replace gameLoop function content**

Replace the `// Start` and `gameLoop` placeholder with:

```javascript
        let lastTime = 0;
        function gameLoop(time = 0) {
            const deltaTime = Math.min((time - lastTime) / 1000, 0.1);
            lastTime = time;

            if (gameRunning) {
                handleInput(deltaTime);
                updatePhysics(deltaTime);
                updateAI(deltaTime);
                updateBubbles(deltaTime);
                checkGoals();
            }

            updateUI();
            renderer.render(scene, camera);
            requestAnimationFrame(gameLoop);
        }

        function handleInput(dt) {
            const player = players[controlledPlayerIndex];
            if (!player) return;

            let moveX = 0, moveY = 0, moveZ = 0;

            // WASD - horizontal movement
            if (keys['KeyW'] || keys['ArrowUp']) moveZ = -1;
            if (keys['KeyS'] || keys['ArrowDown']) moveZ = 1;
            if (keys['KeyA'] || keys['ArrowLeft']) moveX = -1;
            if (keys['KeyD'] || keys['ArrowRight']) moveX = 1;

            // UI - vertical movement
            if (keys['KeyU']) moveY = 1;
            if (keys['KeyI']) moveY = -1;

            // Normalize diagonal movement
            if (moveX !== 0 && moveZ !== 0) {
                moveX *= 0.707;
                moveZ *= 0.707;
            }

            // Apply speed
            let speed = PLAYER_SPEED;
            if (keys['Space'] && energy > 0) {
                speed *= SPRINT_MULTIPLIER;
                energy -= ENERGY_DRAIN * dt;
                energy = Math.max(0, energy);
            } else {
                energy += ENERGY_REGEN * dt;
                energy = Math.min(MAX_ENERGY, energy);
            }

            player.position.x += moveX * speed * dt;
            player.position.y += moveY * VERTICAL_SPEED * dt;
            player.position.z += moveZ * speed * dt;

            // Keep player in bounds (hexagonal)
            clampToField(player.position);

            // Update glow ring
            if (player.userData.ring) {
                player.userData.ring.visible = true;
                player.userData.ring.rotation.z += dt * 2;
            }

            // Tab switching
            if (keys['Tab']) {
                // Show switch hint (handled in UI)
            }
        }

        function clampToField(pos) {
            const maxR = FIELD_RADIUS - PLAYER_RADIUS;
            const dist = Math.sqrt(pos.x * pos.x + pos.z * pos.z);
            if (dist > maxR) {
                const scale = maxR / dist;
                pos.x *= scale;
                pos.z *= scale;
            }
            pos.y = Math.max(0.5, Math.min(8, pos.y));
        }

        function updatePhysics(dt) {
            // Ball physics
            if (ball.userData.velocity.length() > 0.01) {
                ball.position.add(ball.userData.velocity.clone().multiplyScalar(dt));
                ball.userData.velocity.multiplyScalar(UNDERWATER_DRAG);

                // Ball collision with players
                players.forEach(p => {
                    const dist = ball.position.distanceTo(p.position);
                    if (dist < BALL_RADIUS + PLAYER_RADIUS) {
                        const normal = ball.position.clone().sub(p.position).normalize();
                        ball.userData.velocity.add(normal.multiplyScalar(3));
                    }
                });

                // Ball bounds
                clampToField(ball.position);
            }
        }

        function updateBubbles(dt) {
            scene.children.forEach(child => {
                if (child.userData && child.userData.speed) {
                    child.position.y += child.userData.speed * dt;
                    child.userData.wobble += dt;
                    child.position.x += Math.sin(child.userData.wobble) * 0.02;
                    if (child.position.y > 15) {
                        child.position.y = 0;
                    }
                }
            });
        }

        function checkGoals() {
            // Left goal (cyan scores)
            if (ball.position.x < -FIELD_RADIUS - GOAL_DEPTH/2 &&
                Math.abs(ball.position.z) < GOAL_WIDTH/2 &&
                ball.position.y < 3) {
                scoreA++;
                resetPositions();
            }
            // Right goal (magenta scores)
            if (ball.position.x > FIELD_RADIUS + GOAL_DEPTH/2 &&
                Math.abs(ball.position.z) < GOAL_WIDTH/2 &&
                ball.position.y < 3) {
                scoreB++;
                resetPositions();
            }

            if (scoreA >= 5 || scoreB >= 5) {
                gameRunning = false;
                const winner = scoreA >= 5 ? 'CYAN' : 'MAGENTA';
                alert(winner + ' WINS!');
            }
        }

        function resetPositions() {
            ball.position.set(0, 1, 0);
            ball.userData.velocity.set(0, 0, 0);

            players[0].position.set(-8, 1, 0);
            players[1].position.set(-4, 2, 3);
            players[2].position.set(-4, 1, -3);
            players[3].position.set(8, 1, 0);
            players[4].position.set(4, 2, 3);
            players[5].position.set(4, 1, -3);
        }

        function updateUI() {
            document.getElementById('score-a').textContent = 'CYAN: ' + scoreA;
            document.getElementById('score-b').textContent = 'MAGENTA: ' + scoreB;
            document.getElementById('energy-fill').style.width = energy + '%';
            document.getElementById('player-indicator').textContent =
                'Controlling: P' + (controlledPlayerIndex + 1) + ' (' +
                (controlledPlayerIndex < 3 ? 'CYAN' : 'MAGENTA') + ')';
        }
```

- [ ] **Step 3: Test player movement**

Refresh browser.

Expected: WASD moves player, U/I moves up/down, player glows yellow, moves stay within hexagonal bounds.

---

## Task 3: AI System

**Files:**
- Modify: `index.html` (add AI behavior)

- [ ] **Step 1: Add AI state machine**

Add after `updatePhysics` function:

```javascript
        const AI_STATES = {
            CHASE_BALL: 'chase_ball',
            ATTACK: 'attack',
            DEFEND: 'defend',
            SHOOT: 'shoot'
        };

        function updateAI(dt) {
            players.forEach((player, index) => {
                // Skip controlled player
                if (index === controlledPlayerIndex) return;

                const team = player.userData.team;
                const isAI = (team === 'A' && index > 0) || (team === 'B' && index > 3);
                if (!isAI) return;

                // Simple state machine
                const distToBall = player.position.distanceTo(ball.position);
                const distToOwnGoal = team === 'A' ?
                    player.position.x + FIELD_RADIUS :
                    FIELD_RADIUS - player.position.x;

                let targetPos = ball.position.clone();
                let state = AI_STATES.ATTACK;

                // Defend if ball is far and near own goal
                if (distToBall > 10 && distToOwnGoal < 8) {
                    state = AI_STATES.DEFEND;
                    targetPos = new THREE.Vector3(
                        team === 'A' ? -5 : 5,
                        1,
                        ball.position.z * 0.5
                    );
                }

                // Chase ball if far
                if (distToBall > 5) {
                    state = AI_STATES.CHASE_BALL;
                    targetPos = ball.position.clone();
                }

                // Shoot when close to ball
                if (distToBall < 2) {
                    state = AI_STATES.SHOOT;
                    // Aim at opponent's goal
                    const goalX = team === 'A' ? FIELD_RADIUS + 3 : -FIELD_RADIUS - 3;
                    const shootDir = new THREE.Vector3(
                        goalX - player.position.x,
                        0,
                        -player.position.z
                    ).normalize();
                    targetPos = player.position.clone().add(shootDir.multiplyScalar(3));
                }

                // Move toward target
                const direction = targetPos.clone().sub(player.position).normalize();
                const aiSpeed = PLAYER_SPEED * 0.8; // Slightly slower than player

                player.position.add(direction.multiplyScalar(aiSpeed * dt));

                // Vertical behavior - stay near ball's Y
                const targetY = Math.max(0.5, Math.min(5, ball.position.y));
                player.position.y += (targetY - player.position.y) * dt * 2;

                clampToField(player.position);
            });
        }
```

- [ ] **Step 2: Update handleInput to skip AI players**

Add at start of handleInput:

```javascript
        function handleInput(dt) {
            const player = players[controlledPlayerIndex];
            if (!player) return;

            // Tab - player switching (only on keydown, not hold)
            if (keys['Tab'] && !keys['TabUsed']) {
                keys['TabUsed'] = true;
                // Cycle through own team players
                const team = controlledPlayerIndex < 3 ? 'A' : 'B';
                const startIndex = team === 'A' ? 0 : 3;
                controlledPlayerIndex = startIndex + ((controlledPlayerIndex - startIndex + 1) % 3);

                // Update ring visibility
                players.forEach((p, i) => {
                    if (p.userData.ring) p.userData.ring.visible = (i === controlledPlayerIndex);
                });
            }
            if (!keys['Tab']) {
                keys['TabUsed'] = false;
            }
            // ... rest of function
```

- [ ] **Step 3: Test AI**

Refresh browser. After 5 seconds, AI players should move toward ball and try to shoot.

---

## Task 4: Audio System

**Files:**
- Modify: `index.html` (add Web Audio API)

- [ ] **Step 1: Add audio system**

Add after game constants:

```javascript
// Audio system
let audioContext;
let bgMusicGain, sfxGain;

function initAudio() {
    audioContext = new (window.AudioContext || window.webkitAudioContext)();
    bgMusicGain = audioContext.createGain();
    bgMusicGain.gain.value = 0.3;
    bgMusicGain.connect(audioContext.destination);

    sfxGain = audioContext.createGain();
    sfxGain.gain.value = 0.5;
    sfxGain.connect(audioContext.destination);
}

function playBackgroundMusic() {
    if (!audioContext) initAudio();

    // Simple ambient drone
    const osc1 = audioContext.createOscillator();
    const osc2 = audioContext.createOscillator();
    const filter = audioContext.createBiquadFilter();

    osc1.type = 'sine';
    osc1.frequency.value = 55;
    osc2.type = 'sawtooth';
    osc2.frequency.value = 57;

    filter.type = 'lowpass';
    filter.frequency.value = 200;

    osc1.connect(filter);
    osc2.connect(filter);
    filter.connect(bgMusicGain);

    osc1.start();
    osc2.start();
}

function playGoalSound(team) {
    if (!audioContext) initAudio();

    const osc = audioContext.createOscillator();
    const gain = audioContext.createGain();

    osc.type = 'square';
    osc.frequency.setValueAtTime(440, audioContext.currentTime);
    osc.frequency.setValueAtTime(880, audioContext.currentTime + 0.1);
    osc.frequency.setValueAtTime(660, audioContext.currentTime + 0.2);

    gain.gain.setValueAtTime(0.5, audioContext.currentTime);
    gain.gain.exponentialRampToValueAtTime(0.01, audioContext.currentTime + 0.5);

    osc.connect(gain);
    gain.connect(sfxGain);

    osc.start();
    osc.stop(audioContext.currentTime + 0.5);
}

function playCollisionSound() {
    if (!audioContext) initAudio();

    const osc = audioContext.createOscillator();
    const gain = audioContext.createGain();

    osc.type = 'sine';
    osc.frequency.value = 150 + Math.random() * 100;

    gain.gain.setValueAtTime(0.2, audioContext.currentTime);
    gain.gain.exponentialRampToValueAtTime(0.01, audioContext.currentTime + 0.1);

    osc.connect(gain);
    gain.connect(sfxGain);

    osc.start();
    osc.stop(audioContext.currentTime + 0.1);
}

function playSprintSound() {
    if (!audioContext) initAudio();

    const osc = audioContext.createOscillator();
    const gain = audioContext.createGain();

    osc.type = 'sawtooth';
    osc.frequency.setValueAtTime(200, audioContext.currentTime);
    osc.frequency.linearRampToValueAtTime(400, audioContext.currentTime + 0.15);

    gain.gain.setValueAtTime(0.15, audioContext.currentTime);
    gain.gain.exponentialRampToValueAtTime(0.01, audioContext.currentTime + 0.2);

    osc.connect(gain);
    gain.connect(sfxGain);

    osc.start();
    osc.stop(audioContext.currentTime + 0.2);
}
```

- [ ] **Step 2: Call playBackgroundMusic in init()**

Add at end of `init()` function:

```javascript
            // Start audio on first interaction
            document.addEventListener('click', () => {
                if (!audioContext) playBackgroundMusic();
            }, { once: true });
            document.addEventListener('keydown', () => {
                if (!audioContext) playBackgroundMusic();
            }, { once: true });
        }
```

- [ ] **Step 3: Call goal/collision sounds**

In `checkGoals()`, add before `resetPositions()`:

```javascript
                playGoalSound(scoreA >= 5 ? 'A' : 'B');
```

In `updatePhysics()`, add collision detection:

```javascript
        function updatePhysics(dt) {
            let collided = false;
            // Ball collision with players
            players.forEach(p => {
                const dist = ball.position.distanceTo(p.position);
                if (dist < BALL_RADIUS + PLAYER_RADIUS && ball.userData.velocity.length() > 0.5) {
                    collided = true;
                    const normal = ball.position.clone().sub(p.position).normalize();
                    ball.userData.velocity.add(normal.multiplyScalar(3));
                }
            });
            if (collided) playCollisionSound();
            // ... rest
        }
```

In `handleInput()`, add sprint sound:

```javascript
            if (keys['Space'] && energy > 0 && !keys['SpaceUsed']) {
                speed *= SPRINT_MULTIPLIER;
                energy -= ENERGY_DRAIN * dt;
                energy = Math.max(0, energy);
                playSprintSound();
                keys['SpaceUsed'] = true;
            }
            if (!keys['Space']) {
                keys['SpaceUsed'] = false;
            }
```

- [ ] **Step 4: Test audio**

Click or press key to start music. Move fast into ball for collision sound. Score a goal for goal sound.

---

## Task 5: Visual Polish

**Files:**
- Modify: `index.html` (enhance visuals)

- [ ] **Step 1: Add ball trail effect**

Add after ball creation in `createBall()`:

```javascript
            // Add trail particles
            const trailGeometry = new THREE.SphereGeometry(BALL_RADIUS * 0.5, 8, 8);
            ball.userData.trail = [];
            for (let i = 0; i < 10; i++) {
                const trailBall = new THREE.Mesh(trailGeometry,
                    new THREE.MeshBasicMaterial({
                        color: 0x00ffff,
                        transparent: true,
                        opacity: 0.3 - i * 0.03
                    })
                );
                trailBall.visible = false;
                scene.add(trailBall);
                ball.userData.trail.push(trailBall);
            }
```

- [ ] **Step 2: Update physics to show trail**

Add to `updatePhysics()`:

```javascript
            // Update ball trail
            if (ball.userData.velocity.length() > 0.5) {
                ball.userData.trail.forEach((t, i) => {
                    t.visible = true;
                    const trailPos = ball.position.clone().sub(
                        ball.userData.velocity.clone().normalize().multiplyScalar(i * 0.3)
                    );
                    t.position.lerp(trailPos, 0.5);
                });
            } else {
                ball.userData.trail.forEach(t => t.visible = false);
            }
```

- [ ] **Step 3: Add goal celebration effect**

Add in `checkGoals()` before `resetPositions()`:

```javascript
                // Flash effect
                const flashColor = scoreA >= 5 ? 0x00ffff : 0xff00ff;
                const flashLight = new THREE.PointLight(flashColor, 5, 30);
                flashLight.position.copy(ball.position);
                scene.add(flashLight);
                setTimeout(() => scene.remove(flashLight), 500);
```

- [ ] **Step 4: Enhance player glow**

In `createPlayers()`, update glow ring:

```javascript
                const ringMaterial = new THREE.MeshBasicMaterial({
                    color: 0xffff00,
                    transparent: true,
                    opacity: 0.8
                });
```

- [ ] **Step 5: Test visual effects**

Refresh. Ball should leave trail when moving fast. Goals should trigger light flash.

---

## Task 6: Final Testing and Polish

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add camera follow**

Add to `gameLoop` before `updateUI()`:

```javascript
            // Camera follows controlled player smoothly
            const targetCamPos = new THREE.Vector3(
                players[controlledPlayerIndex].position.x + 15,
                20,
                players[controlledPlayerIndex].position.z + 15
            );
            camera.position.lerp(targetCamPos, dt * 2);
            camera.lookAt(players[controlledPlayerIndex].position);
```

- [ ] **Step 2: Add player indicator in 3D**

Add after creating players:

```javascript
        function updateUI() {
            // Update 3D player indicator
            players.forEach((p, i) => {
                if (p.userData.ring) {
                    p.userData.ring.visible = (i === controlledPlayerIndex);
                    if (i === controlledPlayerIndex) {
                        p.userData.ring.rotation.z += 0.05;
                    }
                }
            });
            // ... rest of existing updateUI
        }
```

- [ ] **Step 3: Test full game flow**

1. Open in browser
2. Click/keypress to start music
3. Use WASD+UI to move
4. Ball should collide and move
5. AI should respond
6. Score 5 goals to win

---

## Task 7: Commit and Push

- [ ] **Step 1: Commit changes**

```bash
cd /Users/gaostone/game/underwater-rugby
git add index.html
git commit -m "feat: complete underwater rugby game with Three.js

- 3D isometric view with hexagonal cyberpunk arena
- Player movement (WASD + UI for vertical)
- Ball physics with underwater drag
- AI state machine (chase/attack/defend/shoot)
- Tab switching between team players
- Sprint with energy bar
- Audio: background music, collision, sprint, goal sounds
- Visual effects: ball trail, goal flash, player glow"
```

- [ ] **Step 2: Add remote and push**

```bash
git remote add origin https://github.com/bumpkingstone/underwater-rugby.git
git branch -M main
git push -u origin main
```

---

## Verification Checklist

- [ ] Game loads without errors
- [ ] WASD + U/I controls player in 3D
- [ ] Ball bounces off players
- [ ] AI players move and shoot
- [ ] Tab switches between own team players
- [ ] Sprint works with energy depletion
- [ ] Background music plays on interaction
- [ ] Goal sounds play when scoring
- [ ] Camera follows controlled player
- [ ] First to 5 goals triggers win message
- [ ] All commits pushed to GitHub