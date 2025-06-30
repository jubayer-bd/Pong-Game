<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Adventure Pong Game</title>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap" rel="stylesheet">
    <style>
        body {
            margin: 0;
            overflow: hidden; /* Hide scrollbars */
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            background-color: #1a202c; /* Dark background */
            color: #ffffff; /* White text */
            font-family: 'Inter', sans-serif;
            -webkit-user-select: none; /* Disable text selection */
            -moz-user-select: none;
            -ms-user-select: none;
            user-select: none;
        }

        canvas {
            background-color: #000000; /* Game area background */
            display: block;
            border: 4px solid #4CAF50; /* Green border */
            border-radius: 8px; /* Rounded corners for the game board */
            box-shadow: 0 0 20px rgba(76, 175, 80, 0.7); /* Glow effect */
            touch-action: none; /* Prevent default touch actions like scrolling */
        }

        #game-info {
            margin-bottom: 20px; /* Adjusted margin */
            font-size: 1.5rem;
            font-weight: bold;
            text-align: center;
            color: #64dd17; /* Light green for score */
        }

        #level-info {
            margin-top: 10px; /* New element for level display */
            font-size: 1.2rem;
            color: #FFEB3B; /* Yellow for level */
        }

        #message-box {
            position: fixed;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background-color: rgba(0, 0, 0, 0.85);
            padding: 30px 40px;
            border-radius: 15px;
            box-shadow: 0 0 25px rgba(255, 255, 255, 0.3);
            text-align: center;
            font-size: 1.8rem;
            color: #ffffff;
            z-index: 1000;
            display: none; /* Hidden by default */
            max-width: 80%; /* Responsive width */
        }

        #message-box button {
            background-color: #4CAF50; /* Green button */
            color: white;
            padding: 12px 25px;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            font-size: 1.3rem;
            margin-top: 20px;
            transition: background-color 0.3s ease, transform 0.2s ease;
            box-shadow: 0 4px 10px rgba(0, 0, 0, 0.3);
        }

        #message-box button:hover {
            background-color: #64dd17; /* Lighter green on hover */
            transform: translateY(-2px);
        }

        #message-box button:active {
            transform: translateY(0);
            box-shadow: 0 2px 5px rgba(0, 0, 0, 0.3);
        }
    </style>
</head>
<body>
    <div id="game-info">Score: <span id="score">0</span> | Level: <span id="level">1</span></div>
    <canvas id="pongCanvas"></canvas>

    <!-- Custom Message Box -->
    <div id="message-box">
        <p id="message-content"></p>
        <button id="message-button">Start / Next Level</button>
    </div>

    <script>
        const canvas = document.getElementById('pongCanvas');
        const ctx = canvas.getContext('2d');
        const scoreDisplay = document.getElementById('score');
        const levelDisplay = document.getElementById('level'); // Get level display element
        const messageBox = document.getElementById('message-box');
        const messageContent = document.getElementById('message-content');
        const messageButton = document.getElementById('message-button');

        // Game variables
        let gameRunning = false;
        let score = 0;
        let level = 1; // Current level
        const maxLevels = 3; // Total number of levels
        let paddleWidth = 100;
        let initialPaddleWidth = 100; // Store initial paddle width for reset
        let paddleHeight = 15;
        let paddleX; // X position of the paddle
        let ballRadius = 10;
        let ballX, ballY; // Ball position
        let ballDX, ballDY; // Ball velocity
        let initialBallSpeed = 3; // Base ball speed
        let animationFrameId;
        let blocks = []; // Array to hold block objects

        // --- Block Properties ---
        const blockWidth = 70;
        const blockHeight = 25;
        const blockPadding = 10;
        const blockOffsetTop = 50; // Distance from top of canvas
        const blockColors = ['#FF5722', '#03A9F4', '#8BC34A', '#E91E63']; // Colors for blocks

        // Function to show custom message box
        function showMessageBox(message, buttonText = "OK", callback = null) {
            messageContent.textContent = message;
            messageButton.textContent = buttonText;
            messageBox.style.display = 'block';
            messageButton.onclick = () => {
                messageBox.style.display = 'none';
                if (callback) {
                    callback();
                }
            };
        }

        // --- Setup Level Configuration ---
        function setupLevel(currentLevel) {
            level = currentLevel;
            levelDisplay.textContent = level;
            score = 0; // Reset score for new level
            scoreDisplay.textContent = score;

            // Adjust difficulty based on level
            const speedMultiplier = 1 + (level - 1) * 0.5; // Increase speed by 0.5 for each level
            ballDX = (Math.random() > 0.5 ? 1 : -1) * initialBallSpeed * speedMultiplier;
            ballDY = -initialBallSpeed * speedMultiplier;

            // Paddle width can shrink slightly per level (optional)
            paddleWidth = Math.max(50, initialPaddleWidth - (level - 1) * 10);

            // Reset ball position
            ballX = canvas.width / 2;
            ballY = canvas.height - paddleHeight - ballRadius - 5;
            paddleX = (canvas.width - paddleWidth) / 2;

            // Generate blocks for the current level
            generateBlocks();
        }

        // --- Generate Blocks for Current Level ---
        function generateBlocks() {
            blocks = [];
            const cols = Math.floor((canvas.width - blockPadding) / (blockWidth + blockPadding));
            const rows = level; // More rows of blocks for higher levels

            const totalBlockAreaWidth = cols * (blockWidth + blockPadding) - blockPadding;
            const startX = (canvas.width - totalBlockAreaWidth) / 2;

            for (let r = 0; r < rows; r++) {
                for (let c = 0; c < cols; c++) {
                    // Introduce some randomness or gaps for adventure
                    if (Math.random() > 0.1) { // 10% chance to skip a block for variety
                         blocks.push({
                            x: startX + c * (blockWidth + blockPadding),
                            y: blockOffsetTop + r * (blockHeight + blockPadding),
                            width: blockWidth,
                            height: blockHeight,
                            hits: 1, // Blocks take 1 hit to break
                            color: blockColors[Math.floor(Math.random() * blockColors.length)]
                        });
                    }
                }
            }
        }

        // --- Game Initialization and Reset ---
        function initializeGame() {
             // Set canvas size dynamically based on window size
            canvas.width = Math.min(window.innerWidth * 0.9, 800);
            canvas.height = Math.min(window.innerHeight * 0.7, 600);

            score = 0;
            level = 1;
            initialPaddleWidth = 100; // Reset paddle width
            
            // Set up the first level
            setupLevel(1); 

            gameRunning = false; // Game starts paused with message box
            if (animationFrameId) {
                cancelAnimationFrame(animationFrameId); // Stop any previous animation
            }
            // Do not call gameLoop() here, it will be called by message box callback
        }


        // --- Drawing Functions ---
        function drawPaddle() {
            ctx.fillStyle = '#FFEB3B'; // Yellow paddle
            ctx.roundRect(paddleX, canvas.height - paddleHeight, paddleWidth, paddleHeight, 8); // Rounded rectangle
            ctx.fill();
        }

        function drawBall() {
            ctx.beginPath();
            ctx.arc(ballX, ballY, ballRadius, 0, Math.PI * 2);
            ctx.fillStyle = '#E91E63'; // Pink ball
            ctx.fill();
            ctx.closePath();
        }

        function drawBlocks() {
            blocks.forEach(block => {
                ctx.fillStyle = block.color;
                ctx.roundRect(block.x, block.y, block.width, block.height, 5); // Rounded blocks
                ctx.fill();
            });
        }

        function draw() {
            ctx.clearRect(0, 0, canvas.width, canvas.height); // Clear canvas
            drawBlocks(); // Draw blocks
            drawPaddle();
            drawBall();
        }

        // --- Game Logic ---
        function update() {
            if (!gameRunning) return;

            // Move ball
            ballX += ballDX;
            ballY += ballDY;

            // Wall collision (left/right)
            if (ballX + ballRadius > canvas.width || ballX - ballRadius < 0) {
                ballDX = -ballDX;
            }

            // Wall collision (top)
            if (ballY - ballRadius < 0) {
                ballDY = -ballDY;
            }

            // Paddle collision
            if (
                ballY + ballRadius > canvas.height - paddleHeight && // Ball is at paddle height
                ballX > paddleX && // Ball is within paddle's left edge
                ballX < paddleX + paddleWidth && // Ball is within paddle's right edge
                ballDY > 0 // Ball is moving downwards
            ) {
                ballDY = -ballDY; // Reverse vertical direction
                
                // Add a slight random angle variation on paddle hit
                ballDX += (Math.random() - 0.5) * 0.5; // Small random perturbation
                
                // Ensure ballDX doesn't become too small (stuck) or too large
                if (Math.abs(ballDX) < 1) ballDX = Math.sign(ballDX) * 1;
            }

            // Ball-Block Collision
            for (let i = 0; i < blocks.length; i++) {
                const block = blocks[i];

                // Simple AABB collision detection
                if (ballX + ballRadius > block.x &&
                    ballX - ballRadius < block.x + block.width &&
                    ballY + ballRadius > block.y &&
                    ballY - ballRadius < block.y + block.height) {

                    // Collision detected!
                    block.hits--; // Decrease block's health

                    // Determine which side of the block was hit to reflect ball correctly
                    const overlapX = Math.min(ballX + ballRadius - block.x, block.x + block.width - (ballX - ballRadius));
                    const overlapY = Math.min(ballY + ballRadius - block.y, block.y + block.height - (ballY - ballRadius));

                    if (overlapX < overlapY) {
                        ballDX = -ballDX; // Horizontal collision
                    } else {
                        ballDY = -ballDY; // Vertical collision
                    }

                    if (block.hits <= 0) {
                        blocks.splice(i, 1); // Remove block if no hits left
                        i--; // Adjust index after removal
                        score += 10; // Score for breaking a block
                        scoreDisplay.textContent = score;
                    }
                }
            }

            // Check for level completion
            if (blocks.length === 0) {
                gameRunning = false;
                if (level < maxLevels) {
                    showMessageBox(`Level ${level} Complete! Score: ${score}`, "Next Level", () => {
                        setupLevel(level + 1);
                        gameRunning = true;
                        gameLoop();
                    });
                } else {
                    showMessageBox(`Congratulations! You won the game! Final Score: ${score}`, "Play Again", initializeGame);
                }
                return; // Stop updating for current frame
            }

            // Game Over (ball hits bottom)
            if (ballY + ballRadius > canvas.height) {
                gameRunning = false;
                showMessageBox(`Game Over! Your score: ${score}`, "Play Again", initializeGame);
            }
        }

        // --- Main Game Loop ---
        function gameLoop() {
            if (gameRunning) {
                update();
                draw();
                animationFrameId = requestAnimationFrame(gameLoop);
            }
        }

        // --- Event Listeners for Paddle Control ---

        // Mouse control for desktop
        canvas.addEventListener('mousemove', (e) => {
            if (!gameRunning) return;
            const relativeX = e.clientX - canvas.getBoundingClientRect().left;
            paddleX = relativeX - paddleWidth / 2;

            // Keep paddle within canvas bounds
            if (paddleX < 0) {
                paddleX = 0;
            }
            if (paddleX + paddleWidth > canvas.width) {
                paddleX = canvas.width - paddleWidth;
            }
        });

        // Touch control for mobile
        canvas.addEventListener('touchmove', (e) => {
            if (!gameRunning) return;
            e.preventDefault(); // Prevent scrolling
            const touchX = e.touches[0].clientX - canvas.getBoundingClientRect().left;
            paddleX = touchX - paddleWidth / 2;

            // Keep paddle within canvas bounds
            if (paddleX < 0) {
                paddleX = 0;
            }
            if (paddleX + paddleWidth > canvas.width) {
                paddleX = canvas.width - paddleWidth;
            }
        }, { passive: false }); // Use passive: false to allow preventDefault

        // Resize handler - Re-initializes the game on window resize
        window.addEventListener('resize', initializeGame);

        // Start the game on window load
        window.onload = function() {
            initializeGame();
            // Show initial instruction message
            showMessageBox("Use your mouse or touch to move the paddle and hit all blocks!", "Start Game", () => {
                gameRunning = true;
                gameLoop();
            });
        };

        // Polyfill for CanvasRenderingContext2D.roundRect (if not natively supported)
        if (!('roundRect' in CanvasRenderingContext2D.prototype)) {
            CanvasRenderingContext2D.prototype.roundRect = function(x, y, width, height, radius) {
                if (typeof radius === 'number') {
                    radius = { tl: radius, tr: radius, br: radius, bl: radius };
                } else if (typeof radius === 'object') {
                    radius = { ...{ tl: 0, tr: 0, br: 0, bl: 0 }, ...radius };
                } else {
                    radius = { tl: 0, tr: 0, br: 0, bl: 0 };
                }

                this.beginPath();
                this.moveTo(x + radius.tl, y);
                this.lineTo(x + width - radius.tr, y);
                this.arcTo(x + width, y, x + width, y + radius.tr, radius.tr);
                this.lineTo(x + width, y + height - radius.br);
                this.arcTo(x + width, y + height, x + width - radius.br, y + height, radius.br);
                this.lineTo(x + radius.bl, y + height);
                this.arcTo(x, y + height, x, y + height - radius.bl, radius.bl);
                this.lineTo(x, y + radius.tl);
                this.arcTo(x, y, x + radius.tl, y, radius.tl);
                this.closePath();
                return this;
            };
        }

    </script>
</body>
</html>
