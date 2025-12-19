<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Basic Tower Defense</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        body {
            font-family: 'Comic Sans MS', cursive;
            background: linear-gradient(45deg, #ff00ff, #00ffff, #ffff00);
            background-size: 400% 400%;
            animation: gradient 5s ease infinite;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            overflow: hidden;
            padding: 10px;
        }
        @keyframes gradient {
            0% { background-position: 0% 50%; }
            50% { background-position: 100% 50%; }
            100% { background-position: 0% 50%; }
        }
        #gameContainer {
            background: rgba(0, 0, 0, 0.8);
            border: 5px solid #ff00ff;
            border-radius: 20px;
            padding: 20px;
            box-shadow: 0 0 50px rgba(255, 0, 255, 0.8);
            max-width: 100%;
        }
        #gameCanvas {
            border: 3px solid #00ffff;
            display: block;
            cursor: crosshair;
            background: #1a1a1a;
        }
        #ui {
            margin-top: 15px;
            display: flex;
            gap: 15px;
            flex-wrap: wrap;
            align-items: center;
        }
        .stat {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            padding: 10px 20px;
            border-radius: 10px;
            color: white;
            font-weight: bold;
            font-size: 16px;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.5);
            border: 2px solid #fff;
        }
        .tower-btn {
            background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);
            border: 3px solid #fff;
            padding: 12px 20px;
            border-radius: 15px;
            color: white;
            font-weight: bold;
            cursor: pointer;
            transition: all 0.3s;
            font-size: 14px;
            text-shadow: 1px 1px 2px rgba(0,0,0,0.5);
        }
        .tower-btn:hover {
            transform: scale(1.1) rotate(2deg);
            box-shadow: 0 0 20px rgba(255, 255, 255, 0.8);
        }
        .tower-btn:disabled {
            opacity: 0.5;
            cursor: not-allowed;
            transform: none;
        }
        #startWave {
            background: linear-gradient(135deg, #fa709a 0%, #fee140 100%);
            font-size: 18px;
            padding: 15px 30px;
            animation: pulse 1.5s infinite;
        }
        @keyframes pulse {
            0%, 100% { transform: scale(1); }
            50% { transform: scale(1.05); }
        }
        .tower-info {
            font-size: 11px;
            display: block;
            margin-top: 3px;
        }
        #gameOver {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: rgba(0, 0, 0, 0.95);
            padding: 40px;
            border-radius: 20px;
            border: 5px solid #ff0000;
            color: white;
            text-align: center;
            display: none;
            z-index: 1000;
        }
        #gameOver h1 {
            font-size: 48px;
            margin-bottom: 20px;
            color: #ff0000;
            text-shadow: 3px 3px 6px rgba(255, 0, 0, 0.5);
        }
        #restartBtn {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            border: 3px solid #fff;
            padding: 15px 40px;
            font-size: 20px;
            color: white;
            border-radius: 15px;
            cursor: pointer;
            margin-top: 20px;
        }
    </style>
</head>
<body>
    <div id="gameContainer">
        <canvas id="gameCanvas" width="800" height="600"></canvas>
        <div id="ui">
            <div class="stat">💰 Gold: <span id="gold">500</span></div>
            <div class="stat">❤️ Health: <span id="lives">1000</span></div>
            <div class="stat">🌊 Wave: <span id="wave">0</span></div>
            <div class="stat">💀 Kills: <span id="kills">0</span></div>
            <button class="tower-btn" id="basicTower">🗼 Basic ($50)<span class="tower-info">DMG: 20</span></button>
            <button class="tower-btn" id="sniperTower">🎯 Sniper ($100)<span class="tower-info">DMG: 80, Range++</span></button>
            <button class="tower-btn" id="cannonTower">💥 Cannon ($150)<span class="tower-info">AOE DMG: 50</span></button>
            <button class="tower-btn" id="laserTower">⚡ Laser ($200)<span class="tower-info">DMG: 30, Fast</span></button>
            <button class="tower-btn" id="sellTower">❌ Sell (50%)</button>
            <button class="tower-btn" id="upgradeTower">⬆️ Upgrade</button>
            <button class="tower-btn" id="startWave">▶️ START WAVE</button>
            <button class="tower-btn" id="autoStartWave">🔄 Auto Start: OFF</button>
        </div>
    </div>
    <div id="gameOver">
        <h1>GAME OVER! 💀</h1>
        <p style="font-size: 24px; margin: 10px 0;">Wave Reached: <span id="finalWave"></span></p>
        <p style="font-size: 24px; margin: 10px 0;">Enemies Killed: <span id="finalKills"></span></p>
        <button id="restartBtn">🔄 RESTART</button>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');

        // Game state
        let gold = 500;
        let lives = 1000;
        let wave = 0;
        let kills = 0;
        let selectedTowerType = null;
        let selectedTower = null;
        let towers = [];
        let enemies = [];
        let projectiles = [];
        let particles = [];
        let waveInProgress = false;
        let gameRunning = true;
        let autoStartWave = false;

        // Path definition - FIXED coordinates
        const path = [
            {x: 0, y: 100},
            {x: 150, y: 100},
            {x: 200, y: 200},
            {x: 350, y: 250},
            {x: 400, y: 350},
            {x: 250, y: 400},
            {x: 200, y: 500},
            {x: 400, y: 550},
            {x: 600, y: 500},
            {x: 700, y: 400},
            {x: 800, y: 300}
        ];

        // Tower types
        const towerTypes = {
            basic: {
                cost: 50,
                damage: 20,
                range: 100,
                fireRate: 60,
                color: '#00ff00',
                projectileSpeed: 5,
                projectileColor: '#00ff00',
                upgradeCost: 40,
                name: 'Basic'
            },
            sniper: {
                cost: 100,
                damage: 80,
                range: 200,
                fireRate: 120,
                color: '#ff00ff',
                projectileSpeed: 10,
                projectileColor: '#ff00ff',
                upgradeCost: 80,
                name: 'Sniper'
            },
            cannon: {
                cost: 150,
                damage: 50,
                range: 120,
                fireRate: 90,
                color: '#ff8800',
                projectileSpeed: 4,
                projectileColor: '#ff4400',
                aoe: 60,
                upgradeCost: 120,
                name: 'Cannon'
            },
            laser: {
                cost: 200,
                damage: 30,
                range: 110,
                fireRate: 15,
                color: '#00ffff',
                projectileSpeed: 15,
                projectileColor: '#00ffff',
                upgradeCost: 160,
                name: 'Laser'
            }
        };

        // Enemy types
        class Enemy {
            constructor(type, wave) {
                this.type = type;
                this.pathIndex = 0;
                this.progress = 0;
                this.x = path[0].x;
                this.y = path[0].y;
                
                const multiplier = 1 + (wave * 0.3);
                
                if (type === 'basic') {
                    this.maxHealth = 100 * multiplier;
                    this.speed = 1;
                    this.reward = 10;
                    this.color = '#ff0000';
                    this.size = 12;
                    this.damage = 20;
                } else if (type === 'fast') {
                    this.maxHealth = 50 * multiplier;
                    this.speed = 2;
                    this.reward = 15;
                    this.color = '#ffff00';
                    this.size = 10;
                    this.damage = 50;
                } else if (type === 'tank') {
                    this.maxHealth = 300 * multiplier;
                    this.speed = 0.5;
                    this.reward = 30;
                    this.color = '#0000ff';
                    this.size = 16;
                    this.damage = 80;
                } else if (type === 'boss') {
                    this.maxHealth = 1000 * multiplier;
                    this.speed = 0.3;
                    this.reward = 100;
                    this.color = '#ff00ff';
                    this.size = 24;
                    this.damage = 150;
                }
                
                this.health = this.maxHealth;
            }

            update() {
                if (this.pathIndex >= path.length - 1) {
                    lives -= this.damage;
                    if (lives < 0) lives = 0;
                    updateUI();
                    if (lives <= 0) endGame();
                    return false;
                }

                const start = path[this.pathIndex];
                const end = path[this.pathIndex + 1];
                const dx = end.x - start.x;
                const dy = end.y - start.y;
                const dist = Math.sqrt(dx * dx + dy * dy);

                this.progress += this.speed / dist;

                if (this.progress >= 1) {
                    this.progress = 0;
                    this.pathIndex++;
                    if (this.pathIndex >= path.length - 1) {
                        lives -= this.damage;
                        if (lives < 0) lives = 0;
                        updateUI();
                        if (lives <= 0) endGame();
                        return false;
                    }
                }

                const start2 = path[this.pathIndex];
                const end2 = path[this.pathIndex + 1];
                this.x = start2.x + (end2.x - start2.x) * this.progress;
                this.y = start2.y + (end2.y - start2.y) * this.progress;

                return true;
            }

            draw() {
                // Enemy body
                ctx.fillStyle = this.color;
                ctx.beginPath();
                ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2);
                ctx.fill();
                
                // Border
                ctx.strokeStyle = '#fff';
                ctx.lineWidth = 2;
                ctx.stroke();

                // Health bar
                const barWidth = this.size * 2;
                const barHeight = 4;
                const healthPercent = this.health / this.maxHealth;
                
                ctx.fillStyle = '#000';
                ctx.fillRect(this.x - barWidth/2, this.y - this.size - 10, barWidth, barHeight);
                
                ctx.fillStyle = healthPercent > 0.5 ? '#0f0' : healthPercent > 0.25 ? '#ff0' : '#f00';
                ctx.fillRect(this.x - barWidth/2, this.y - this.size - 10, barWidth * healthPercent, barHeight);
            }

            takeDamage(damage) {
                this.health -= damage;
                createParticles(this.x, this.y, this.color);
                return this.health <= 0;
            }
        }

        class Tower {
            constructor(x, y, type) {
                this.x = x;
                this.y = y;
                this.type = type;
                this.stats = {...towerTypes[type]};
                this.cooldown = 0;
                this.target = null;
                this.level = 1;
                this.angle = 0;
            }

            update() {
                this.cooldown--;

                // Find target
                this.target = null;
                let closestDist = this.stats.range;
                
                for (let enemy of enemies) {
                    const dx = enemy.x - this.x;
                    const dy = enemy.y - this.y;
                    const dist = Math.sqrt(dx * dx + dy * dy);
                    
                    if (dist < closestDist) {
                        closestDist = dist;
                        this.target = enemy;
                    }
                }

                // Aim at target
                if (this.target) {
                    const dx = this.target.x - this.x;
                    const dy = this.target.y - this.y;
                    this.angle = Math.atan2(dy, dx);

                    // Fire
                    if (this.cooldown <= 0) {
                        this.fire();
                        this.cooldown = this.stats.fireRate;
                    }
                }
            }

            fire() {
                projectiles.push({
                    x: this.x,
                    y: this.y,
                    target: this.target,
                    damage: this.stats.damage,
                    speed: this.stats.projectileSpeed,
                    color: this.stats.projectileColor,
                    aoe: this.stats.aoe || 0,
                    tower: this
                });
            }

            draw() {
                // Range circle when selected
                if (selectedTower === this) {
                    ctx.strokeStyle = 'rgba(255, 255, 255, 0.3)';
                    ctx.lineWidth = 2;
                    ctx.beginPath();
                    ctx.arc(this.x, this.y, this.stats.range, 0, Math.PI * 2);
                    ctx.stroke();
                }

                // Tower base
                ctx.fillStyle = this.stats.color;
                ctx.beginPath();
                ctx.arc(this.x, this.y, 20, 0, Math.PI * 2);
                ctx.fill();
                ctx.strokeStyle = '#000';
                ctx.lineWidth = 3;
                ctx.stroke();

                // Tower barrel
                ctx.save();
                ctx.translate(this.x, this.y);
                ctx.rotate(this.angle);
                ctx.fillStyle = this.stats.color;
                ctx.fillRect(0, -5, 25, 10);
                ctx.strokeStyle = '#000';
                ctx.lineWidth = 2;
                ctx.strokeRect(0, -5, 25, 10);
                ctx.restore();

                // Level indicator
                ctx.fillStyle = '#fff';
                ctx.font = 'bold 12px Arial';
                ctx.textAlign = 'center';
                ctx.fillText(this.level, this.x, this.y + 5);

                // Selection indicator
                if (selectedTower === this) {
                    ctx.strokeStyle = '#ffff00';
                    ctx.lineWidth = 3;
                    ctx.beginPath();
                    ctx.arc(this.x, this.y, 25, 0, Math.PI * 2);
                    ctx.stroke();
                }
            }

            upgrade() {
                if (this.level >= 10) {
                    return false;
                }
                this.level++;
                this.stats.damage *= 1.5;
                this.stats.range *= 1.1;
                this.stats.fireRate *= 0.9;
                return true;
            }

            getSellValue() {
                let value = this.stats.cost * 0.5;
                for (let i = 1; i < this.level; i++) {
                    value += towerTypes[this.type].upgradeCost * 0.5 * i;
                }
                return Math.floor(value);
            }

            getUpgradeCost() {
                return Math.floor(this.stats.upgradeCost * this.level);
            }
        }

        function createParticles(x, y, color) {
            for (let i = 0; i < 8; i++) {
                particles.push({
                    x: x,
                    y: y,
                    vx: (Math.random() - 0.5) * 4,
                    vy: (Math.random() - 0.5) * 4,
                    life: 30,
                    color: color
                });
            }
        }

        function updateProjectiles() {
            for (let i = projectiles.length - 1; i >= 0; i--) {
                const proj = projectiles[i];
                
                if (!proj.target || proj.target.health <= 0) {
                    projectiles.splice(i, 1);
                    continue;
                }

                const dx = proj.target.x - proj.x;
                const dy = proj.target.y - proj.y;
                const dist = Math.sqrt(dx * dx + dy * dy);

                if (dist < proj.speed + 5) {
                    // Hit
                    if (proj.aoe > 0) {
                        // AOE damage
                        for (let enemy of enemies) {
                            const edx = enemy.x - proj.target.x;
                            const edy = enemy.y - proj.target.y;
                            const edist = Math.sqrt(edx * edx + edy * edy);
                            if (edist < proj.aoe) {
                                if (enemy.takeDamage(proj.damage)) {
                                    gold += enemy.reward;
                                    kills++;
                                    updateUI();
                                }
                            }
                        }
                        // AOE visual
                        createParticles(proj.target.x, proj.target.y, proj.color);
                    } else {
                        if (proj.target.takeDamage(proj.damage)) {
                            gold += proj.target.reward;
                            kills++;
                            updateUI();
                        }
                    }
                    projectiles.splice(i, 1);
                } else {
                    proj.x += (dx / dist) * proj.speed;
                    proj.y += (dy / dist) * proj.speed;
                }
            }
        }

        function drawProjectiles() {
            for (let proj of projectiles) {
                ctx.fillStyle = proj.color;
                ctx.beginPath();
                ctx.arc(proj.x, proj.y, 5, 0, Math.PI * 2);
                ctx.fill();
                ctx.strokeStyle = '#fff';
                ctx.lineWidth = 2;
                ctx.stroke();
            }
        }

        function updateParticles() {
            for (let i = particles.length - 1; i >= 0; i--) {
                const p = particles[i];
                p.x += p.vx;
                p.y += p.vy;
                p.life--;
                if (p.life <= 0) {
                    particles.splice(i, 1);
                }
            }
        }

        function drawParticles() {
            for (let p of particles) {
                ctx.fillStyle = p.color;
                ctx.globalAlpha = p.life / 30;
                ctx.fillRect(p.x - 2, p.y - 2, 4, 4);
            }
            ctx.globalAlpha = 1;
        }

        function drawPath() {
            // Path border
            ctx.strokeStyle = '#222';
            ctx.lineWidth = 44;
            ctx.lineCap = 'round';
            ctx.lineJoin = 'round';
            ctx.beginPath();
            ctx.moveTo(path[0].x, path[0].y);
            for (let i = 1; i < path.length; i++) {
                ctx.lineTo(path[i].x, path[i].y);
            }
            ctx.stroke();

            // Main path
            ctx.strokeStyle = '#444';
            ctx.lineWidth = 40;
            ctx.beginPath();
            ctx.moveTo(path[0].x, path[0].y);
            for (let i = 1; i < path.length; i++) {
                ctx.lineTo(path[i].x, path[i].y);
            }
            ctx.stroke();
        }

        function canPlaceTower(x, y) {
            // Check if on path
            for (let i = 0; i < path.length - 1; i++) {
                const start = path[i];
                const end = path[i + 1];
                const dist = distToSegment({x, y}, start, end);
                if (dist < 40) return false;
            }

            // Check if too close to other towers
            for (let tower of towers) {
                const dx = tower.x - x;
                const dy = tower.y - y;
                if (Math.sqrt(dx * dx + dy * dy) < 50) return false;
            }

            return true;
        }

        function distToSegment(p, v, w) {
            const l2 = (v.x - w.x) ** 2 + (v.y - w.y) ** 2;
            if (l2 === 0) return Math.sqrt((p.x - v.x) ** 2 + (p.y - v.y) ** 2);
            const t = Math.max(0, Math.min(1, ((p.x - v.x) * (w.x - v.x) + (p.y - v.y) * (w.y - v.y)) / l2));
            const projection = {
                x: v.x + t * (w.x - v.x),
                y: v.y + t * (w.y - v.y)
            };
            return Math.sqrt((p.x - projection.x) ** 2 + (p.y - projection.y) ** 2);
        }

        function spawnWave() {
            waveInProgress = true;
            wave++;
            updateUI();

            const waveConfig = [
                { type: 'basic', count: 10 + wave * 2 },
                { type: 'fast', count: Math.floor(wave / 2) },
                { type: 'tank', count: Math.floor(wave / 3) }
            ];

            if (wave % 5 === 0) {
                waveConfig.push({ type: 'boss', count: 1 });
            }

            let totalEnemies = 0;
            for (let config of waveConfig) {
                totalEnemies += config.count;
            }

            let spawnedCount = 0;
            const spawnInterval = setInterval(() => {
                if (!gameRunning) {
                    clearInterval(spawnInterval);
                    return;
                }

                let typeIndex = 0;
                let typeCount = 0;
                let currentType = waveConfig[0].type;

                for (let config of waveConfig) {
                    if (spawnedCount < typeCount + config.count) {
                        currentType = config.type;
                        break;
                    }
                    typeCount += config.count;
                }

                enemies.push(new Enemy(currentType, wave));
                spawnedCount++;

                if (spawnedCount >= totalEnemies) {
                    clearInterval(spawnInterval);
                }
            }, 800);
        }

        function updateUI() {
            document.getElementById('gold').textContent = gold;
            document.getElementById('lives').textContent = lives;
            document.getElementById('wave').textContent = wave;
            document.getElementById('kills').textContent = kills;

            // Update button states
            document.getElementById('basicTower').disabled = gold < towerTypes.basic.cost;
            document.getElementById('sniperTower').disabled = gold < towerTypes.sniper.cost;
            document.getElementById('cannonTower').disabled = gold < towerTypes.cannon.cost;
            document.getElementById('laserTower').disabled = gold < towerTypes.laser.cost;
            document.getElementById('sellTower').disabled = !selectedTower;
            document.getElementById('upgradeTower').disabled = !selectedTower || (selectedTower && (gold < selectedTower.getUpgradeCost() || selectedTower.level >= 10));
            
            if (selectedTower) {
                if (selectedTower.level >= 10) {
                    document.getElementById('upgradeTower').innerHTML = `⬆️ MAX LEVEL<span class="tower-info">Level 10</span>`;
                } else {
                    document.getElementById('upgradeTower').innerHTML = `⬆️ Upgrade ($${selectedTower.getUpgradeCost()})<span class="tower-info">Level ${selectedTower.level}</span>`;
                }
            } else {
                document.getElementById('upgradeTower').innerHTML = `⬆️ Upgrade`;
            }

            document.getElementById('startWave').disabled = waveInProgress;
        }

        function endGame() {
            gameRunning = false;
            document.getElementById('finalWave').textContent = wave;
            document.getElementById('finalKills').textContent = kills;
            document.getElementById('gameOver').style.display = 'block';
        }

        function restartGame() {
            // Reset all game state
            gold = 500;
            lives = 1000;
            wave = 0;
            kills = 0;
            selectedTowerType = null;
            selectedTower = null;
            towers = [];
            enemies = [];
            projectiles = [];
            particles = [];
            waveInProgress = false;
            gameRunning = true;
            autoStartWave = false;
            
            // Reset UI
            document.getElementById('autoStartWave').textContent = '🔄 Auto Start: OFF';
            document.getElementById('autoStartWave').style.background = 'linear-gradient(135deg, #f093fb 0%, #f5576c 100%)';
            document.getElementById('gameOver').style.display = 'none';
            
            // Update display
            updateUI();
            
            // Restart game loop
            gameLoop();
        }

        // Event listeners
        canvas.addEventListener('click', (e) => {
            const rect = canvas.getBoundingClientRect();
            const x = e.clientX - rect.left;
            const y = e.clientY - rect.top;

            // Check if clicking on a tower
            let clickedTower = null;
            for (let tower of towers) {
                const dx = tower.x - x;
                const dy = tower.y - y;
                if (Math.sqrt(dx * dx + dy * dy) < 25) {
                    clickedTower = tower;
                    break;
                }
            }

            if (clickedTower) {
                selectedTower = clickedTower;
                selectedTowerType = null;
                updateUI();
            } else if (selectedTowerType && canPlaceTower(x, y)) {
                const towerData = towerTypes[selectedTowerType];
                if (gold >= towerData.cost) {
                    towers.push(new Tower(x, y, selectedTowerType));
                    gold -= towerData.cost;
                    updateUI();
                    selectedTowerType = null;
                    selectedTower = null;
                }
            } else {
                selectedTower = null;
                updateUI();
            }
        });

        document.getElementById('basicTower').addEventListener('click', () => {
            selectedTowerType = 'basic';
            selectedTower = null;
        });

        document.getElementById('sniperTower').addEventListener('click', () => {
            selectedTowerType = 'sniper';
            selectedTower = null;
        });

        document.getElementById('cannonTower').addEventListener('click', () => {
            selectedTowerType = 'cannon';
            selectedTower = null;
        });

        document.getElementById('laserTower').addEventListener('click', () => {
            selectedTowerType = 'laser';
            selectedTower = null;
        });

        document.getElementById('sellTower').addEventListener('click', () => {
            if (selectedTower) {
                gold += selectedTower.getSellValue();
                towers = towers.filter(t => t !== selectedTower);
                selectedTower = null;
                updateUI();
            }
        });

        document.getElementById('upgradeTower').addEventListener('click', () => {
            if (selectedTower && gold >= selectedTower.getUpgradeCost()) {
                gold -= selectedTower.getUpgradeCost();
                selectedTower.upgrade();
                updateUI();
            }
        });

        document.getElementById('startWave').addEventListener('click', () => {
            if (!waveInProgress) {
                spawnWave();
            }
        });

        document.getElementById('autoStartWave').addEventListener('click', () => {
            autoStartWave = !autoStartWave;
            document.getElementById('autoStartWave').textContent = autoStartWave ? '🔄 Auto Start: ON' : '🔄 Auto Start: OFF';
            document.getElementById('autoStartWave').style.background = autoStartWave 
                ? 'linear-gradient(135deg, #00ff00 0%, #00aa00 100%)' 
                : 'linear-gradient(135deg, #f093fb 0%, #f5576c 100%)';
        });

        document.getElementById('restartBtn').addEventListener('click', restartGame);

        // Game loop
        function gameLoop() {
            if (!gameRunning) return;

            ctx.fillStyle = '#1a1a1a';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            drawPath();

            // Update and draw enemies
            for (let i = enemies.length - 1; i >= 0; i--) {
                if (!enemies[i].update() || enemies[i].health <= 0) {
                    if (enemies[i].health <= 0) {
                        gold += enemies[i].reward;
                        kills++;
                        updateUI();
                    }
                    enemies.splice(i, 1);
                }
            }

            if (enemies.length === 0 && waveInProgress) {
                waveInProgress = false;
                updateUI();
                
                // Auto-start next wave if enabled
                if (autoStartWave && gameRunning) {
                    setTimeout(() => {
                        if (autoStartWave && gameRunning) {
                            spawnWave();
                        }
                    }, 2000);
                }
            }

            for (let enemy of enemies) {
                enemy.draw();
            }

            // Update and draw towers
            for (let tower of towers) {
                tower.update();
                tower.draw();
            }

            updateProjectiles();
            drawProjectiles();
            updateParticles();
            drawParticles();

            requestAnimationFrame(gameLoop);
        }

        updateUI();
        gameLoop();
    </script>
</body>
</html>
