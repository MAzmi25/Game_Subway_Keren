<html>
<head>
<title>Simple Runner Game - Harta!</title>
<style>
    body { margin: 0; display: flex; justify-content: center; align-items: center; height: 100vh; background-color: #d3d3d3; font-family: 'Arial', sans-serif; }
    #gameContainer { display: flex; flex-direction: column; align-items: center; padding: 20px; background-color: #f0f0f0; border-radius: 10px; box-shadow: 0 0 10px rgba(0,0,0,0.2); }
    canvas { border: 2px solid #333; background-color: #6DB5D7; }
    #uiContainer { display: flex; justify-content: space-between; width: 100%; margin-top: 10px; align-items: center; }
    .scoreDisplay { font-size: 20px; color: #333; }
    #restartButton { padding: 10px 15px; font-size: 16px; color: white; background-color: #4CAF50; border: none; border-radius: 5px; cursor: pointer; }
    #restartButton:hover { background-color: #45a049; }
    #instructions { margin-top: 15px; font-size: 14px; color: #555; text-align: center; }
</style>
</head>
<body>

<div id="gameContainer">
    <canvas id="gameCanvas"></canvas>
    <div id="uiContainer">
        <div>
            <div id="scoreBoard" class="scoreDisplay">Skor: 0</div>
            <div id="coinBoard" class="scoreDisplay">Harta: 0</div> 
        </div>
        <button id="restartButton">Restart Game</button>
    </div>
    <div id="instructions">Panah Kiri/Kanan: Gerak | Spasi: Pause/Lanjut | 'R': Restart (saat Game Over)</div>
</div>

<script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    const scoreBoard = document.getElementById('scoreBoard');
    const coinBoard = document.getElementById('coinBoard');
    const restartButton = document.getElementById('restartButton');

    const laneWidth = 80;
    const numLanes = 3;
    canvas.width = laneWidth * numLanes;
    canvas.height = 500;

    // --- Properti Pemain ---
    const playerVisualWidth = 40; const playerVisualHeight = 60; let playerLane = 1;
    const playerBaseY = canvas.height - playerVisualHeight - 30; 
    const playerBodyColor = '#FFFFFF'; const playerLimbTipColor = '#000000'; 
    const playerHeadSkinColor = '#FFDAB9'; const playerHairColor = '#000000'; 
    let playerAnimationFrame = 0; const playerAnimationSpeed = 15; let playerSpriteToggle = false; 

    // --- Properti Rintangan (Kereta) ---
    const trainWidth = 60; const trainHeight = 100; let obstacles = [];
    let obstacleSpeed = 3; let obstacleSpawnIntervalBase = 120;
    let currentObstacleSpawnInterval = obstacleSpawnIntervalBase; const trainColor = '#555E63';

    // --- Properti Koin (Harta) ---
    const coinVisualHeight = 20; const coinColor = '#FFD700'; let coins = [];
    let coinSpawnInterval = 70; let frameCountForCoins = 0;
    const coinSpinWidths = [20, 16, 10, 5, 2, 5, 10, 16]; const coinAnimationSpeed = 8;

    // --- Properti Berlian ---
    const diamondWidth = 22; const diamondHeight = 28; const diamondColor = '#00BFFF'; 
    let diamonds = []; let diamondSpawnInterval = 250; let frameCountForDiamonds = 0; 
    const diamondShineMinAlpha = 0.2; const diamondShineMaxAlpha = 0.7;
    const diamondShineSpeed = 6; const diamondFloatAmplitude = 3; 
    const diamondFloatSpeed = 0.08;

    // --- Properti Rel ---
    const railColor = '#4A3B31'; const sleeperColor = '#7B6E66'; const railVisualWidth = 10;
    const sleeperVisualWidth = laneWidth - 10; const sleeperVisualHeight = 15;
    const sleeperSpacing = 50; let railOffsetY = 0;
    
    // --- Skor & Game State ---
    let score = 0; let treasuresCollected = 0; 
    let gameOver = false;
    let isPaused = false; let globalFrameCount = 0; let animationFrameId;

    // --- Suara Koin (Embedded Base64) ---
    const coinSoundBase64_simple_blip = "data:audio/wav;base64,UklGRkIAAABXQVZFZm10IBAAAAABAAEARKwAAIhYAQACABAAZGF0YQUAAAAHCHN0dWRpbw==";
    let coinSound;
    try { coinSound = new Audio(coinSoundBase64_simple_blip); } catch (e) { console.error("Gagal membuat objek Audio.", e); coinSound = { play: function() {}, currentTime: 0 }; }

    // =================== FUNGSI-FUNGSI GAME ===================
    function resetGame() {
        playerLane = 1; obstacles = []; coins = []; diamonds = []; 
        obstacleSpeed = 3; currentObstacleSpawnInterval = obstacleSpawnIntervalBase; 
        score = 0; treasuresCollected = 0; 
        globalFrameCount = 0; frameCountForCoins = 0; frameCountForDiamonds = 0; 
        playerAnimationFrame = 0; railOffsetY = 0; 
        gameOver = false; isPaused = false;
        scoreBoard.textContent = "Skor: " + score;
        coinBoard.textContent = "Harta: " + treasuresCollected; 
        if (animationFrameId) cancelAnimationFrame(animationFrameId);
        gameLoop();
    }

    function clearCanvas() { ctx.clearRect(0, 0, canvas.width, canvas.height); }
    function drawRails() {
        ctx.fillStyle = sleeperColor;
        for (let y = railOffsetY % sleeperSpacing - sleeperSpacing; y < canvas.height; y += sleeperSpacing) {
            for (let i = 0; i < numLanes; i++) {
                 const sleeperX = i * laneWidth + (laneWidth - sleeperVisualWidth) / 2;
                 ctx.fillRect(sleeperX, y, sleeperVisualWidth, sleeperVisualHeight);
            }
        }
        ctx.fillStyle = railColor;
        for (let i = 0; i < numLanes; i++) {
            ctx.fillRect(i * laneWidth + (laneWidth / 2) - railVisualWidth - 5, 0, railVisualWidth, canvas.height);
            ctx.fillRect(i * laneWidth + (laneWidth / 2) + 5, 0, railVisualWidth, canvas.height);
        }
    }
    function drawPlayer() { 
        const playerX = playerLane * laneWidth + (laneWidth - playerVisualWidth) / 2;
        const hairHeight = playerVisualHeight * 0.15; const headSkinHeight = playerVisualHeight * 0.20;
        const bodyHeight = playerVisualHeight * 0.35; const legTotalVisualHeight = playerVisualHeight * 0.30; 
        const headAndHairWidth = playerVisualWidth * 0.7; const armWidth = playerVisualWidth * 0.18; 
        const armTotalVisualLength = playerVisualHeight * 0.40; const limbTipPortion = 0.20; 
        const shoeHeight = legTotalVisualHeight * limbTipPortion; const upperLegHeight = legTotalVisualHeight - shoeHeight;
        const gloveHeight = armTotalVisualLength * limbTipPortion; const upperArmLength = armTotalVisualLength - gloveHeight;
        const hairTopY = playerBaseY; const headSkinTopY = hairTopY + hairHeight;
        const bodyTopY = headSkinTopY + headSkinHeight; const legsAttachY = bodyTopY + bodyHeight; 
        const shoulderY = bodyTopY + bodyHeight * 0.15; 
        ctx.fillStyle = playerHairColor; ctx.fillRect(playerX + (playerVisualWidth - headAndHairWidth)/2 , hairTopY, headAndHairWidth, hairHeight);
        ctx.fillStyle = playerHeadSkinColor; ctx.fillRect(playerX + (playerVisualWidth - headAndHairWidth)/2 , headSkinTopY, headAndHairWidth, headSkinHeight);
        ctx.fillStyle = playerBodyColor; ctx.fillRect(playerX, bodyTopY, playerVisualWidth, bodyHeight);
        const legSingleWidth = playerVisualWidth * 0.35;
        let leftLegEffectiveTotalHeight = playerSpriteToggle ? legTotalVisualHeight : legTotalVisualHeight * 0.8;
        let rightLegEffectiveTotalHeight = playerSpriteToggle ? legTotalVisualHeight * 0.8 : legTotalVisualHeight;
        let currentUpperLegH_L = leftLegEffectiveTotalHeight * (1 - limbTipPortion); let currentShoeH_L = leftLegEffectiveTotalHeight * limbTipPortion;
        ctx.fillStyle = playerBodyColor; ctx.fillRect(playerX + (playerVisualWidth * 0.1), legsAttachY, legSingleWidth, currentUpperLegH_L);
        ctx.fillStyle = playerLimbTipColor; ctx.fillRect(playerX + (playerVisualWidth * 0.1), legsAttachY + currentUpperLegH_L, legSingleWidth, currentShoeH_L);
        let currentUpperLegH_R = rightLegEffectiveTotalHeight * (1 - limbTipPortion); let currentShoeH_R = rightLegEffectiveTotalHeight * limbTipPortion;
        ctx.fillStyle = playerBodyColor; ctx.fillRect(playerX + playerVisualWidth - legSingleWidth - (playerVisualWidth*0.1), legsAttachY, legSingleWidth, currentUpperLegH_R);
        ctx.fillStyle = playerLimbTipColor; ctx.fillRect(playerX + playerVisualWidth - legSingleWidth - (playerVisualWidth*0.1), legsAttachY + currentUpperLegH_R, legSingleWidth, currentShoeH_R);
        const armSwingOffsetY = armTotalVisualLength * 0.1; 
        let currentLeftArmY = shoulderY + (playerSpriteToggle ? armSwingOffsetY : -armSwingOffsetY * 0.5);
        ctx.fillStyle = playerBodyColor; ctx.fillRect(playerX - armWidth + armWidth*0.3, currentLeftArmY, armWidth, upperArmLength);
        ctx.fillStyle = playerLimbTipColor; ctx.fillRect(playerX - armWidth + armWidth*0.3, currentLeftArmY + upperArmLength, armWidth, gloveHeight);
        let currentRightArmY = shoulderY + (playerSpriteToggle ? -armSwingOffsetY * 0.5 : armSwingOffsetY);
        ctx.fillStyle = playerBodyColor; ctx.fillRect(playerX + playerVisualWidth - armWidth*0.3, currentRightArmY, armWidth, upperArmLength);
        ctx.fillStyle = playerLimbTipColor; ctx.fillRect(playerX + playerVisualWidth - armWidth*0.3, currentRightArmY + upperArmLength, armWidth, gloveHeight);
    }
    function drawObstacles() { obstacles.forEach(obstacle => { const obstacleX = obstacle.lane * laneWidth + (laneWidth - trainWidth) / 2; ctx.fillStyle = trainColor; ctx.fillRect(obstacleX, obstacle.y, trainWidth, trainHeight); ctx.fillStyle = '#FFFF00'; ctx.fillRect(obstacleX + trainWidth/2 - 5, obstacle.y + 10, 10, 10); }); }
    function drawCoins() { ctx.fillStyle = coinColor; coins.forEach(coin => { const coinCenterX = coin.lane * laneWidth + laneWidth / 2; const currentVisualCoinWidth = coinSpinWidths[coin.animFrame]; ctx.beginPath(); ctx.ellipse(coinCenterX, coin.y, currentVisualCoinWidth / 2, coinVisualHeight / 2, 0, 0, Math.PI * 2); ctx.fill(); }); }
    function drawDiamonds() { diamonds.forEach(diamond => { const diamondCenterX = diamond.lane * laneWidth + laneWidth / 2; const floatOffsetY = Math.sin(diamond.floatAngle) * diamondFloatAmplitude; const actualDrawY = diamond.y + floatOffsetY; const dX = diamondCenterX; const dY = actualDrawY; const dW = diamondWidth; const dH = diamondHeight; ctx.fillStyle = diamondColor; ctx.beginPath(); ctx.moveTo(dX, dY - dH / 2); ctx.lineTo(dX + dW / 2, dY); ctx.lineTo(dX, dY + dH / 2); ctx.lineTo(dX - dW / 2, dY); ctx.closePath(); ctx.fill(); ctx.fillStyle = `rgba(255, 255, 255, ${diamond.shineAlpha})`; ctx.beginPath(); ctx.moveTo(dX, dY - dH / 2); ctx.lineTo(dX + dW / 4, dY - dH / 4); ctx.lineTo(dX, dY); ctx.lineTo(dX - dW / 4, dY - dH/4); ctx.closePath(); ctx.fill(); }); }

    function updateEnvironment() { if (!gameOver && !isPaused) railOffsetY += obstacleSpeed; }
    function updatePlayerAnimation() { if (gameOver || isPaused) return; playerAnimationFrame++; if (playerAnimationFrame >= playerAnimationSpeed) { playerAnimationFrame = 0; playerSpriteToggle = !playerSpriteToggle; } }
    function spawnNewObstacle() { const newLane = Math.floor(Math.random() * numLanes); obstacles.push({ lane: newLane, y: -trainHeight }); if (currentObstacleSpawnInterval > 40) currentObstacleSpawnInterval *= 0.99; }
    function spawnNewCoin() { const newLane = Math.floor(Math.random() * numLanes); let canSpawnCoin = true; obstacles.forEach(obs => { if (obs.lane === newLane && obs.y < coinVisualHeight * 2 && obs.y > -trainHeight * 0.7) { canSpawnCoin = false; } }); if (canSpawnCoin) { coins.push({ lane: newLane, y: -coinVisualHeight, animFrame: Math.floor(Math.random() * coinSpinWidths.length), animFrameCounter: 0 }); } }
    function spawnNewDiamond() { const newLane = Math.floor(Math.random() * numLanes); let canSpawnDiamond = true; obstacles.forEach(obs => { if (obs.lane === newLane && obs.y < diamondHeight * 2 && obs.y > -trainHeight * 0.7) canSpawnDiamond = false; }); coins.forEach(cn => { if (cn.lane === newLane && cn.y < diamondHeight * 2 && cn.y > -coinVisualHeight * 0.7) canSpawnDiamond = false; }); if (canSpawnDiamond) { diamonds.push({ lane: newLane, y: -diamondHeight, shineAlpha: diamondShineMinAlpha, shineAlphaDirection: 1, shineFrameCounter: 0, floatAngle: Math.random() * Math.PI * 2 }); } }
    
    function updateGameElements() { 
        if (gameOver || isPaused) return; 
        globalFrameCount++;
        frameCountForCoins++;
        frameCountForDiamonds++; 

        if (globalFrameCount % Math.floor(currentObstacleSpawnInterval) === 0) spawnNewObstacle();
        if (frameCountForCoins % coinSpawnInterval === 0) { spawnNewCoin(); frameCountForCoins = 0; }
        if (frameCountForDiamonds % diamondSpawnInterval === 0) { spawnNewDiamond(); frameCountForDiamonds = 0; }

        obstacles.forEach((obstacle, i, arr) => { obstacle.y += obstacleSpeed; if (obstacle.y > canvas.height) { arr.splice(i, 1); score += 10; scoreBoard.textContent = "Skor: " + score; if (score > 0 && score % 50 === 0) obstacleSpeed = Math.min(obstacleSpeed + 0.2, 10); } });
        coins.forEach((coin, i, arr) => { coin.y += obstacleSpeed; coin.animFrameCounter++; if (coin.animFrameCounter >= coinAnimationSpeed) { coin.animFrameCounter = 0; coin.animFrame = (coin.animFrame + 1) % coinSpinWidths.length; } if (coin.y > canvas.height + coinVisualHeight) { arr.splice(i, 1); } });
        diamonds.forEach((diamond, i, arr) => { diamond.y += obstacleSpeed; diamond.shineFrameCounter++; if (diamond.shineFrameCounter >= diamondShineSpeed) { diamond.shineFrameCounter = 0; diamond.shineAlpha += 0.1 * diamond.shineAlphaDirection; if (diamond.shineAlpha >= diamondShineMaxAlpha) { diamond.shineAlpha = diamondShineMaxAlpha; diamond.shineAlphaDirection = -1; } else if (diamond.shineAlpha <= diamondShineMinAlpha) { diamond.shineAlpha = diamondShineMinAlpha; diamond.shineAlphaDirection = 1; } } diamond.floatAngle += diamondFloatSpeed; if (diamond.floatAngle > Math.PI * 2) { diamond.floatAngle -= Math.PI * 2; } if (diamond.y > canvas.height + diamondHeight + diamondFloatAmplitude) { arr.splice(i, 1); } });
    }

    function getPlayerHitbox() { const playerX = playerLane * laneWidth + (laneWidth - playerVisualWidth) / 2; return { x: playerX, y: playerBaseY, width: playerVisualWidth, height: playerVisualHeight }; }
    function checkCollisions() { if (gameOver || isPaused) return; const playerHitbox = getPlayerHitbox(); obstacles.forEach(obstacle => { const obstacleHitbox = { x: obstacle.lane * laneWidth + (laneWidth - trainWidth) / 2, y: obstacle.y, width: trainWidth, height: trainHeight }; if (playerHitbox.x < obstacleHitbox.x + obstacleHitbox.width && playerHitbox.x + playerHitbox.width > obstacleHitbox.x && playerHitbox.y < obstacleHitbox.y + obstacleHitbox.height && playerHitbox.y + playerHitbox.height > obstacleHitbox.y) { gameOver = true; } }); }
    
    function checkCoinCollection() { 
        if (gameOver || isPaused) return; 
        const playerHitbox = getPlayerHitbox(); 
        for (let i = coins.length - 1; i >= 0; i--) { 
            const coin = coins[i]; const coinCenterX = coin.lane * laneWidth + laneWidth / 2; const currentVisualCoinWidth = coinSpinWidths[coin.animFrame]; 
            const coinHitbox = { x: coinCenterX - currentVisualCoinWidth / 2, y: coin.y - coinVisualHeight / 2, width: currentVisualCoinWidth, height: coinVisualHeight }; 
            if (playerHitbox.x < coinHitbox.x + coinHitbox.width && playerHitbox.x + playerHitbox.width > coinHitbox.x && playerHitbox.y < coinHitbox.y + coinHitbox.height && playerHitbox.y + playerHitbox.height > coinHitbox.y) { 
                coins.splice(i, 1); treasuresCollected += 1; 
                coinBoard.textContent = "Harta: " + treasuresCollected; 
                if (coinSound && typeof coinSound.play === 'function') { coinSound.currentTime = 0; coinSound.play().then(() => {}).catch(error => { console.warn("Gagal play suara koin:", error); }); } 
            } 
        } 
    }

    function checkDiamondCollection() { 
        if (gameOver || isPaused) return;
        const playerHitbox = getPlayerHitbox();
        for (let i = diamonds.length - 1; i >= 0; i--) {
            const diamond = diamonds[i];
            const diamondCenterX = diamond.lane * laneWidth + laneWidth / 2;
            const floatOffsetY = Math.sin(diamond.floatAngle) * diamondFloatAmplitude;
            const actualDiamondDrawY = diamond.y + floatOffsetY;
            const diamondHitbox = { x: diamondCenterX - diamondWidth / 2, y: actualDiamondDrawY - diamondHeight / 2, width: diamondWidth, height: diamondHeight };
            if (playerHitbox.x < diamondHitbox.x + diamondHitbox.width &&
                playerHitbox.x + playerHitbox.width > diamondHitbox.x &&
                playerHitbox.y < diamondHitbox.y + diamondHitbox.height &&
                playerHitbox.y + playerHitbox.height > diamondHitbox.y) {
                diamonds.splice(i, 1); 
                treasuresCollected += 5; 
                coinBoard.textContent = "Harta: " + treasuresCollected; 
            }
        }
    }
    
    function gameLoop() {
        if (gameOver) { 
            ctx.fillStyle = "rgba(0, 0, 0, 0.75)"; ctx.fillRect(0, 0, canvas.width, canvas.height);
            ctx.font = "bold 30px Arial"; ctx.fillStyle = "white"; ctx.textAlign = "center";
            ctx.fillText("GAME OVER", canvas.width / 2, canvas.height / 2 - 60);
            ctx.font = "24px Arial";
            ctx.fillText("Skor Akhir: " + score, canvas.width / 2, canvas.height / 2 - 20);
            ctx.fillText("Harta: " + treasuresCollected, canvas.width / 2, canvas.height/2 + 10);
            ctx.font = "bold 18px Arial"; 
            ctx.fillText("Tekan 'R' Untuk Restart", canvas.width / 2, canvas.height / 2 + 50); 
            return; 
        }
        if (!isPaused) {
            clearCanvas(); updateEnvironment(); drawRails(); updateGameElements(); 
            drawCoins(); drawDiamonds(); drawObstacles();
            updatePlayerAnimation(); drawPlayer();
            checkCollisions(); checkCoinCollection(); checkDiamondCollection(); 
        }
        if (!isPaused || !gameOver) { 
             animationFrameId = requestAnimationFrame(gameLoop);
        }
    }

    document.addEventListener('keydown', function(event) { if (event.key === ' ' || event.code === 'Space') { event.preventDefault(); togglePause(); return; } if (gameOver) { if (event.key === 'r' || event.key === 'R') { resetGame(); } return; } if (isPaused) return; if (event.key === 'ArrowLeft') { if (playerLane > 0) playerLane--; } else if (event.key === 'ArrowRight') { if (playerLane < numLanes - 1) playerLane++; } });
    restartButton.addEventListener('click', function() { if (animationFrameId) cancelAnimationFrame(animationFrameId); resetGame(); });
    function togglePause() { if (gameOver) return; isPaused = !isPaused; if (isPaused) { if (animationFrameId) { cancelAnimationFrame(animationFrameId); } drawPauseScreen(); } else { requestAnimationFrame(gameLoop); } }
    function drawPauseScreen() { ctx.fillStyle = "rgba(0, 0, 0, 0.6)"; ctx.fillRect(0, 0, canvas.width, canvas.height); ctx.font = "bold 40px Arial"; ctx.fillStyle = "white"; ctx.textAlign = "center"; ctx.fillText("PAUSE", canvas.width / 2, canvas.height / 2 - 20); ctx.font = "18px Arial"; ctx.fillText("Tekan Spasi untuk Lanjut", canvas.width / 2, canvas.height / 2 + 30); }
    
    resetGame();

</script>
</body>
</html>
