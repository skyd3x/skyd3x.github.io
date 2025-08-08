<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Cloud Jump</title>
    <style>
        /* style.css - Styling for UI, responsive design */
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(180deg, #87CEEB 0%, #E0F6FF 50%, #B8E6FF 100%);
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            overflow: hidden;
            user-select: none;
        }

        .game-container {
            position: relative;
            width: min(400px, 90vw);
            height: min(600px, 90vh);
            border: 3px solid #4A90E2;
            border-radius: 20px;
            background: linear-gradient(180deg, #87CEEB 0%, #E0F6FF 100%);
            box-shadow: 0 15px 35px rgba(0, 0, 0, 0.2);
            overflow: hidden;
        }

        #gameCanvas {
            display: block;
            background: transparent;
            width: 100%;
            height: 100%;
        }

        .ui-overlay {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            pointer-events: none;
            z-index: 10;
        }

        .score {
            position: absolute;
            top: 20px;
            left: 20px;
            color: #2C3E50;
            font-size: clamp(18px, 4vw, 24px);
            font-weight: bold;
            text-shadow: 2px 2px 4px rgba(255, 255, 255, 0.9);
            background: rgba(255, 255, 255, 0.3);
            padding: 8px 16px;
            border-radius: 20px;
            backdrop-filter: blur(10px);
        }

        .game-over {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: rgba(255, 255, 255, 0.95);
            padding: 30px;
            border-radius: 20px;
            text-align: center;
            box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
            display: none;
            pointer-events: all;
            backdrop-filter: blur(15px);
            border: 2px solid rgba(74, 144, 226, 0.3);
        }

        .game-over h2 {
            color: #E74C3C;
            margin-bottom: 15px;
            font-size: clamp(22px, 5vw, 28px);
            text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.1);
        }

        .game-over p {
            color: #2C3E50;
            margin-bottom: 25px;
            font-size: clamp(14px, 3vw, 18px);
        }

        .restart-btn {
            background: linear-gradient(135deg, #4A90E2, #357ABD);
            color: white;
            border: none;
            padding: 12px 24px;
            border-radius: 12px;
            font-size: clamp(14px, 3vw, 16px);
            font-weight: bold;
            cursor: pointer;
            transition: all 0.3s ease;
            box-shadow: 0 4px 15px rgba(74, 144, 226, 0.3);
        }

        .restart-btn:hover {
            background: linear-gradient(135deg, #357ABD, #2E6DA4);
            transform: translateY(-2px);
            box-shadow: 0 6px 20px rgba(74, 144, 226, 0.4);
        }

        .instructions {
            position: absolute;
            bottom: 20px;
            left: 20px;
            right: 20px;
            color: #2C3E50;
            font-size: clamp(11px, 2.5vw, 14px);
            text-align: center;
            text-shadow: 1px 1px 2px rgba(255, 255, 255, 0.9);
            background: rgba(255, 255, 255, 0.2);
            padding: 10px;
            border-radius: 15px;
            backdrop-filter: blur(5px);
        }

        /* Hidden images for preloading */
        .preload-images {
            position: absolute;
            left: -9999px;
            top: -9999px;
            opacity: 0;
        }
    </style>
</head>
<body>
    <!-- index.html - Basic structure, canvas, game container -->
    <div class="game-container">
        <canvas id="gameCanvas" width="400" height="600"></canvas>
        <div class="ui-overlay">
            <div class="score" id="score">Score: 0</div>
            <div class="game-over" id="gameOver">
                <h2>Game Over!</h2>
                <p>Final Score: <span id="finalScore">0</span></p>
                <button class="restart-btn" onclick="game.restart()">Play Again</button>
            </div>
            <div class="instructions">
                Use ← → arrow keys to steer during jumps<br>
                Land on cloud platforms to keep climbing!
            </div>
        </div>
    </div>

    <!-- Preload the provided images -->
    <div class="preload-images">
        <img id="cloudImg" crossorigin="anonymous">
        <img id="characterImg" crossorigin="anonymous">
    </div>

    <script>
        // game.js - Core game logic, physics, rendering
        class CloudJumpGame {
            constructor() {
                this.canvas = document.getElementById('gameCanvas');
                this.ctx = this.canvas.getContext('2d');
                this.scoreElement = document.getElementById('score');
                this.gameOverElement = document.getElementById('gameOver');
                this.finalScoreElement = document.getElementById('finalScore');

                // Load images
                this.cloudImg = document.getElementById('cloudImg');
                this.characterImg = document.getElementById('characterImg');
                this.imagesLoaded = 0;
                this.totalImages = 2;

                this.loadImages();

                // Game state
                this.gameState = {
                    isPlaying: false,
                    score: 0,
                    cameraY: 0,
                    worldHeight: 0,
                    difficulty: 1
                };

                // Player object
                this.player = {
                    x: 200,
                    y: 500,
                    width: 50,
                    height: 50,
                    velocityX: 0,
                    velocityY: 0,
                    isJumping: false,
                    jumpStartTime: 0,
                    jumpDuration: 2000, // 2 seconds
                    jumpStartY: 0,
                    jumpTargetY: 0,
                    onPlatform: true,
                    direction: 1
                };

                // Platform generation settings
                this.platformSettings = {
                    minGap: 80,
                    maxGap: 140,
                    width: 120,
                    height: 60,
                    destroyDistance: 700 // Destroy platforms this far below camera
                };

                // Platforms array
                this.platforms = [];

                // Input handling
                this.keys = {
                    left: false,
                    right: false
                };

                // Audio context for sound effects
                this.audioContext = null;
                this.initAudio();
            }

            loadImages() {
                // Create cloud platform from the provided image data
                this.cloudImg.onload = () => {
                    this.imagesLoaded++;
                    this.checkImagesLoaded();
                };

                // Create character from the provided image data  
                this.characterImg.onload = () => {
                    this.imagesLoaded++;
                    this.checkImagesLoaded();
                };

                // Convert the provided images to data URLs
                this.createCloudImage();
                this.createCharacterImage();
            }

            createCloudImage() {
                // Сохраняем все существующие переменные
                const canvas = document.createElement('canvas');
                canvas.width = 120;  // сохраняем размеры
                canvas.height = 60;  // как в оригинале
                const ctx = canvas.getContext('2d');
                
                // Вместо рисования - загружаем изображение
                const img = new Image();
                img.crossOrigin = "Anonymous";
                img.src = "cloud__1_.png";
                
                img.onload = () => {
                    // Рисуем загруженное изображение на canvas
                    ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
                    
                    // Сохраняем в this.cloudImg как было раньше
                    this.cloudImg.src = canvas.toDataURL();
                    this.imagesLoaded++;
                    this.checkImagesLoaded();
                };
                
                img.onerror = () => {
                    // Fallback - рисуем простое облако как в оригинале
                    ctx.fillStyle = '#C8E6FF';
                    ctx.strokeStyle = '#4A90E2';
                    ctx.lineWidth = 6;
                    
                    ctx.beginPath();
                    ctx.arc(30, 40, 25, Math.PI, 0, false);
                    ctx.arc(60, 30, 30, Math.PI, 0, false);
                    ctx.arc(90, 35, 25, Math.PI, 0, false);
                    ctx.lineTo(15, 45);
                    ctx.closePath();
                    ctx.fill();
                    ctx.stroke();
                    
                    this.cloudImg.src = canvas.toDataURL();
                    this.imagesLoaded++;
                    this.checkImagesLoaded();
                };
            }

            createCharacterImage() {
                // Create the character based on the blue square with stripes
                const canvas = document.createElement('canvas');
                canvas.width = 50;
                canvas.height = 50;
                const ctx = canvas.getContext('2d');

                // Blue square background
                ctx.fillStyle = '#4A90E2';
                ctx.fillRect(5, 5, 40, 35);

                // White diagonal stripes
                ctx.strokeStyle = '#FFFFFF';
                ctx.lineWidth = 3;
                ctx.lineCap = 'round';

                // Three diagonal stripes
                for (let i = 0; i < 3; i++) {
                    const x = 19 + (i * 8);
                    ctx.beginPath();
                    ctx.moveTo(x, 17);
                    ctx.lineTo(x - 2, 27);
                    ctx.stroke();
                }

                // Simple legs
                ctx.fillStyle = '#2C3E50';
                ctx.fillRect(15, 40, 4, 8);
                ctx.fillRect(31, 40, 4, 8);

                this.characterImg.src = canvas.toDataURL();
            }

            checkImagesLoaded() {
                if (this.imagesLoaded >= this.totalImages) {
                    this.init();
                    this.setupEventListeners();
                    this.gameLoop();
                }
            }

            initAudio() {
                try {
                    this.audioContext = new (window.AudioContext || window.webkitAudioContext)();
                } catch (e) {
                    console.log('Audio not supported');
                }
            }

            playSound(frequency, duration = 0.1, type = 'sine') {
                if (!this.audioContext) return;
                
                const oscillator = this.audioContext.createOscillator();
                const gainNode = this.audioContext.createGain();
                
                oscillator.connect(gainNode);
                gainNode.connect(this.audioContext.destination);
                
                oscillator.frequency.value = frequency;
                oscillator.type = type;
                
                gainNode.gain.setValueAtTime(0.1, this.audioContext.currentTime);
                gainNode.gain.exponentialRampToValueAtTime(0.01, this.audioContext.currentTime + duration);
                
                oscillator.start(this.audioContext.currentTime);
                oscillator.stop(this.audioContext.currentTime + duration);
            }

            init() {
                // Create initial platforms
                this.platforms = [];
                
                // Starting platform
                this.platforms.push({
                    x: 140,
                    y: 550,
                    width: this.platformSettings.width,
                    height: this.platformSettings.height,
                    type: 'normal'
                });

                // Generate initial platforms going up
                this.generateInitialPlatforms();

                // Reset player position
                this.player.x = 175;
                this.player.y = 480;
                this.player.velocityX = 0;
                this.player.velocityY = 0;
                this.player.isJumping = false;
                this.player.onPlatform = true;
                this.player.direction = 1;

                // Reset game state
                this.gameState.isPlaying = true;
                this.gameState.score = 0;
                this.gameState.cameraY = 0;
                this.gameState.worldHeight = 0;
                this.gameState.difficulty = 1;

                this.gameOverElement.style.display = 'none';
                this.updateScore();
            }

            generateInitialPlatforms() {
                let currentY = 550;
                for (let i = 1; i < 15; i++) {
                    const gap = this.platformSettings.minGap + 
                               Math.random() * (this.platformSettings.maxGap - this.platformSettings.minGap);
                    currentY -= gap;
                    
                    this.platforms.push({
                        x: Math.random() * (this.canvas.width - this.platformSettings.width),
                        y: currentY,
                        width: this.platformSettings.width,
                        height: this.platformSettings.height,
                        type: 'normal'
                    });
                }
                this.gameState.worldHeight = Math.abs(currentY);
            }

            generateNewPlatform() {
                if (this.platforms.length === 0) return;

                const highestPlatform = this.platforms.reduce((highest, platform) => 
                    platform.y < highest.y ? platform : highest
                );

                // Calculate gap based on difficulty
                const baseGap = this.platformSettings.minGap + 
                               Math.random() * (this.platformSettings.maxGap - this.platformSettings.minGap);
                const difficultyGap = Math.min(30, this.gameState.difficulty * 2);
                const gap = baseGap + difficultyGap;

                const newY = highestPlatform.y - gap;
                
                this.platforms.push({
                    x: Math.random() * (this.canvas.width - this.platformSettings.width),
                    y: newY,
                    width: this.platformSettings.width,
                    height: this.platformSettings.height,
                    type: 'normal'
                });

                this.gameState.worldHeight = Math.abs(newY);
            }

            destroyPlatformsBelowCamera() {
                const destroyY = this.gameState.cameraY + this.platformSettings.destroyDistance;
                this.platforms = this.platforms.filter(platform => platform.y < destroyY);
            }

            setupEventListeners() {
                document.addEventListener('keydown', (e) => {
                    switch(e.code) {
                        case 'ArrowLeft':
                            this.keys.left = true;
                            this.player.direction = -1;
                            e.preventDefault();
                            break;
                        case 'ArrowRight':
                            this.keys.right = true;
                            this.player.direction = 1;
                            e.preventDefault();
                            break;
                        case 'Space':
                            if (this.gameState.isPlaying && this.player.onPlatform && !this.player.isJumping) {
                                this.jumpToNextPlatform();
                            }
                            e.preventDefault();
                            break;
                    }
                });

                document.addEventListener('keyup', (e) => {
                    switch(e.code) {
                        case 'ArrowLeft':
                            this.keys.left = false;
                            break;
                        case 'ArrowRight':
                            this.keys.right = false;
                            break;
                    }
                });
            }

            drawPlayer() {
                const drawX = this.player.x;
                const drawY = this.player.y - this.gameState.cameraY;
                
                // Skip if player is off screen
                if (drawY < -100 || drawY > this.canvas.height + 100) return;

                this.ctx.save();
                
                
                // Draw character image
                if (this.characterImg.complete) {
                    this.ctx.drawImage(this.characterImg, drawX, drawY, this.player.width, this.player.height);
                }

                this.ctx.restore();
            }

            drawPlatforms() {
                this.platforms.forEach(platform => {
                    const drawY = platform.y - this.gameState.cameraY;
                    
                    // Only draw platforms that are visible
                    if (drawY > -100 && drawY < this.canvas.height + 100) {
                        this.ctx.save();
                        

                        // Draw platform image
                        if (this.cloudImg.complete) {
                            this.ctx.drawImage(this.cloudImg, platform.x, drawY, platform.width, platform.height);
                        }
                        
                        this.ctx.restore();
                    }
                });
            }

            drawBackground() {
                // Dynamic sky gradient background
                const gradient = this.ctx.createLinearGradient(0, 0, 0, this.canvas.height);
                gradient.addColorStop(0, '#87CEEB');
                gradient.addColorStop(0.5, '#E0F6FF');
                gradient.addColorStop(1, '#B8E6FF');
                
                this.ctx.fillStyle = gradient;
                this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);

                // Animated cloud background with parallax
                this.ctx.save();
                
                const cloudOffset = (this.gameState.cameraY * 0.03) % 200;
                for (let layer = 0; layer < 2; layer++) {
                    const layerOffset = cloudOffset * (layer + 1);
                    const opacity = 0.05 + layer * 0.05;
                    this.ctx.fillStyle = `rgba(255, 255, 255, ${opacity})`;
                    
                    for (let i = 0; i < 8; i++) {
                        const x = (i % 3) * 150 + 50;
                        const y = Math.floor(i / 3) * 200 + layerOffset - 100;
                        
                        // Simple cloud shapes
                        this.ctx.beginPath();
                        this.ctx.arc(x, y, 15, 0, Math.PI * 2);
                        this.ctx.arc(x + 20, y, 20, 0, Math.PI * 2);
                        this.ctx.arc(x + 40, y, 15, 0, Math.PI * 2);
                        this.ctx.fill();
                    }
                }
                
                this.ctx.restore();
            }

            checkPlatformCollision() {
                const playerBottom = this.player.y + this.player.height;
                const playerLeft = this.player.x + 5;
                const playerRight = this.player.x + this.player.width - 5;
                
                for (let platform of this.platforms) {
                    if (playerBottom >= platform.y && 
                        playerBottom <= platform.y + platform.height + 10 &&
                        playerRight > platform.x && 
                        playerLeft < platform.x + platform.width) {
                        return platform;
                    }
                }
                return null;
            }

            jumpToNextPlatform() {
                if (this.player.isJumping) return;
                
                // Find the next reachable platform above
                let nextPlatform = null;
                let minDistance = Infinity;
                
                for (let platform of this.platforms) {
                    const verticalDistance = this.player.y - platform.y;
                    const horizontalDistance = Math.abs((platform.x + platform.width/2) - (this.player.x + this.player.width/2));
                    
                    if (verticalDistance > 30 && verticalDistance < 200 && horizontalDistance < 250) {
                        if (verticalDistance < minDistance) {
                            minDistance = verticalDistance;
                            nextPlatform = platform;
                        }
                    }
                }
                
                if (nextPlatform) {
                    this.player.isJumping = true;
                    this.player.jumpStartTime = Date.now();
                    this.player.jumpStartY = this.player.y;
                    this.player.jumpTargetY = nextPlatform.y - this.player.height + 10;
                    this.player.onPlatform = false;
                    
                    // Increase score on jump
                    this.gameState.score += 10;
                    this.updateScore();
                    
                    // Play jump sound
                    this.playSound(400, 0.1, 'sine');
                }
            }

            updateCamera() {
                // Screen scrolls upward only when player reaches top 1/3 of screen
                const playerScreenY = this.player.y - this.gameState.cameraY;
                const scrollThreshold = this.canvas.height / 3;
                
                if (playerScreenY < scrollThreshold) {
                    const targetCameraY = this.player.y - scrollThreshold;
                    this.gameState.cameraY += (targetCameraY - this.gameState.cameraY) * 0.1;
                }
            }

            update() {
                if (!this.gameState.isPlaying) return;

                const currentTime = Date.now();

                // Update difficulty based on score
                this.gameState.difficulty = Math.floor(this.gameState.score / 200) + 1;

                if (this.player.isJumping) {
                    const elapsed = currentTime - this.player.jumpStartTime;
                    const progress = Math.min(elapsed / this.player.jumpDuration, 1);
                    
                    // Smooth jump animation with easing
                    const easeProgress = 1 - Math.pow(1 - progress, 2);
                    this.player.y = this.player.jumpStartY + (this.player.jumpTargetY - this.player.jumpStartY) * easeProgress;
                    
                    // Handle horizontal movement during jump
                    const moveSpeed = 4;
                    if (this.keys.left) {
                        this.player.x = Math.max(0, this.player.x - moveSpeed);
                    }
                    if (this.keys.right) {
                        this.player.x = Math.min(this.canvas.width - this.player.width, this.player.x + moveSpeed);
                    }
                    
                    // Check if jump is complete
                    if (progress >= 1) {
                        this.player.isJumping = false;
                        
                        // Check if landed on a platform
                        const landedPlatform = this.checkPlatformCollision();
                        if (landedPlatform) {
                            this.player.onPlatform = true;
                            this.player.y = landedPlatform.y - this.player.height + 10;
                            
                            // Play landing sound
                            this.playSound(300, 0.05, 'triangle');
                            
                            // Auto-jump to next platform after a brief pause
                            setTimeout(() => {
                                if (this.gameState.isPlaying && this.player.onPlatform) {
                                    this.jumpToNextPlatform();
                                }
                            }, 300);
                        } else {
                            // Game over - didn't land on platform
                            this.gameOver();
                        }
                    }
                }

                // Update camera
                this.updateCamera();

                // Generate new platforms as needed
                const highestVisibleY = this.gameState.cameraY - 200;
                if (this.platforms.length === 0 || this.platforms.every(p => p.y > highestVisibleY)) {
                    for (let i = 0; i < 5; i++) {
                        this.generateNewPlatform();
                    }
                }

                // Destroy platforms below camera
                this.destroyPlatformsBelowCamera();
            }

            updateScore() {
                this.scoreElement.textContent = `Score: ${this.gameState.score}`;
            }

            gameOver() {
                this.gameState.isPlaying = false;
                this.finalScoreElement.textContent = this.gameState.score;
                this.gameOverElement.style.display = 'block';
                
                // Play game over sound
                this.playSound(200, 0.5, 'sawtooth');
            }

            restart() {
                this.init();
                
                // Auto-start first jump
                setTimeout(() => {
                    if (this.gameState.isPlaying && this.player.onPlatform) {
                        this.jumpToNextPlatform();
                    }
                }, 1000);
            }

            gameLoop() {
                // Clear canvas
                this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
                
                // Draw background
                this.drawBackground();
                
                // Update game state
                this.update();
                
                // Draw game objects
                this.drawPlatforms();
                this.drawPlayer();
                
                requestAnimationFrame(() => this.gameLoop());
            }
        }

        // Initialize the game when DOM is loaded
        let game;
        document.addEventListener('DOMContentLoaded', () => {
            game = new CloudJumpGame();
        });
    </script>
</body>
</html>
