<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Гонки с рулём</title>
  <style>
    * { box-sizing: border-box; }
    body {
      margin: 0;
      background: #333;
      color: white;
      font-family: sans-serif;
      display: flex;
      flex-direction: column;
      align-items: center;
    }

    h1 {
      margin-top: 10px;
      font-size: 1.5em;
    }

    #game {
      position: relative;
      width: 320px;
      height: 540px;
      background: linear-gradient(#1e1e1e, #333);
      overflow: hidden;
      margin: 10px;
      border: 4px solid #fff;
    }

    .road-line {
      position: absolute;
      left: 150px;
      width: 4px;
      height: 40px;
      background: white;
      opacity: 0.6;
    }

    .car {
      position: absolute;
      bottom: 10px;
      width: 50px;
      height: 90px;
      left: 135px;
      background: red;
      border-radius: 8px;
      box-shadow: 0 0 10px black inset;
      z-index: 10;
    }

    .car::before {
      content: '';
      position: absolute;
      top: 5px;
      left: 5px;
      width: 40px;
      height: 20px;
      background: lightgray;
      border-radius: 5px;
    }

    .enemy {
      position: absolute;
      width: 50px;
      height: 90px;
      border-radius: 8px;
      box-shadow: 0 0 10px black inset;
    }

    .red { background: #d00; }
    .blue { background: #00d; }
    .green { background: #0b0; }
    .yellow { background: gold; }

    .enemy::before {
      content: '';
      position: absolute;
      top: 5px;
      left: 5px;
      width: 40px;
      height: 20px;
      background: lightgray;
      border-radius: 5px;
    }

    #score {
      font-size: 1.2em;
      margin-bottom: 10px;
    }

    #steeringWheel {
      width: 100px;
      height: 100px;
      border-radius: 50%;
      background: radial-gradient(circle at center, #444, #222);
      border: 6px solid #aaa;
      cursor: grab;
      transform: rotate(0deg);
      transition: transform 0.1s ease-out;
    }

    #restart {
      display: none;
      margin-top: 20px;
      padding: 10px 20px;
      font-size: 1em;
      border: none;
      border-radius: 10px;
      background: green;
      color: white;
      cursor: pointer;
    }
  </style>
</head>
<body>

  <h1>Реалистичные Гонки</h1>
  <div id="score">Счёт: 0</div>
  <div id="game">
    <div class="road-line" style="top: 0;"></div>
    <div class="road-line" style="top: 60px;"></div>
    <div class="road-line" style="top: 120px;"></div>
    <div class="road-line" style="top: 180px;"></div>
    <div class="road-line" style="top: 240px;"></div>
    <div class="road-line" style="top: 300px;"></div>
    <div class="road-line" style="top: 360px;"></div>
    <div class="road-line" style="top: 420px;"></div>
    <div class="road-line" style="top: 480px;"></div>
    <div class="car" id="player"></div>
  </div>
  <div id="steeringWheel"></div>
  <button id="restart" onclick="startGame()">Начать заново</button>

  <!-- Sounds -->
  <audio id="engineSound" src="https://cdn.pixabay.com/audio/2022/03/15/audio_4bfa76d3f4.mp3 " loop></audio>
  <audio id="turnSound" src="https://cdn.pixabay.com/audio/2022/03/15/audio_c8f4d4c59a.mp3 "></audio>
  <audio id="crashSound" src="https://cdn.pixabay.com/audio/2022/03/15/audio_b0ef01c651.mp3 "></audio>
  <audio id="startSound" src="https://cdn.pixabay.com/audio/2022/03/15/audio_28071eb885.mp3 "></audio>

  <script>
    const game = document.getElementById('game');
    const player = document.getElementById('player');
    const scoreEl = document.getElementById('score');
    const restartBtn = document.getElementById('restart');
    const steeringWheel = document.getElementById('steeringWheel');

    const engineSound = document.getElementById("engineSound");
    const turnSound = document.getElementById("turnSound");
    const crashSound = document.getElementById("crashSound");
    const startSound = document.getElementById("startSound");

    let playerX = 135;
    let gameInterval;
    let enemyInterval;
    let score = 0;
    let isGameOver = false;

    let rotation = 0;
    let isDragging = false;
    let startX = 0;

    const lanes = [50, 135, 220];

    function movePlayerTo(x) {
      if (x < 0) x = 0;
      if (x > 270) x = 270;
      playerX = x;
      player.style.left = playerX + 'px';
      updateSteeringWheel();
    }

    function updateSteeringWheel() {
      const normalized = playerX / 270;
      rotation = (normalized - 0.5) * 60;
      steeringWheel.style.transform = `rotate(${rotation}deg)`;
    }

    // Управление мышью
    steeringWheel.addEventListener('mousedown', e => {
      isDragging = true;
      startX = e.clientX;
      playTurnSound();
    });

    window.addEventListener('mousemove', e => {
      if (!isDragging || isGameOver) return;
      const dx = e.clientX - startX;
      movePlayerTo(playerX + dx * 0.5);
      startX = e.clientX;
    });

    window.addEventListener('mouseup', () => isDragging = false);

    // Управление тачскрином
    steeringWheel.addEventListener('touchstart', e => {
      isDragging = true;
      startX = e.touches[0].clientX;
      playTurnSound();
    });

    steeringWheel.addEventListener('touchmove', e => {
      if (!isDragging || isGameOver) return;
      const dx = e.touches[0].clientX - startX;
      movePlayerTo(playerX + dx * 0.5);
      startX = e.touches[0].clientX;
    });

    steeringWheel.addEventListener('touchend', () => isDragging = false);

    function createEnemyCar() {
      const usedLanes = [...document.querySelectorAll('.enemy')].map(car => parseInt(car.style.left));
      const availableLanes = lanes.filter(lane => !usedLanes.includes(lane));

      if (availableLanes.length === 0) return;

      const randomLane = availableLanes[Math.floor(Math.random() * availableLanes.length)];
      const colors = ['red', 'blue', 'green', 'yellow'];
      const randomColor = colors[Math.floor(Math.random() * colors.length)];

      const enemy = document.createElement('div');
      enemy.classList.add('enemy', randomColor);
      enemy.style.left = randomLane + 'px';
      enemy.style.top = '-90px';
      game.appendChild(enemy);

      let speed = 2 + Math.random() * 2;

      const interval = setInterval(() => {
        if (isGameOver) {
          clearInterval(interval);
          enemy.remove();
          return;
        }

        const top = parseFloat(enemy.style.top);
        enemy.style.top = (top + speed) + 'px';

        if (checkCollision(player, enemy)) {
          gameOver();
          clearInterval(interval);
          return;
        }

        if (top > 600) {
          enemy.remove();
          clearInterval(interval);
          updateScore();
        }
      }, 16);
    }

    function checkCollision(a, b) {
      const aRect = a.getBoundingClientRect();
      const bRect = b.getBoundingClientRect();

      return !(
        aRect.top > bRect.bottom ||
        aRect.bottom < bRect.top ||
        aRect.right < bRect.left ||
        aRect.left > bRect.right
      );
    }

    function updateScore() {
      score++;
      scoreEl.textContent = `Счёт: ${score}`;
    }

    function gameOver() {
      isGameOver = true;
      engineSound.pause();
      crashSound.play();
      clearInterval(gameInterval);
      clearInterval(enemyInterval);
      restartBtn.style.display = 'block';
      alert("💥 Столкновение! Игра окончена.");
    }

    function playTurnSound() {
      turnSound.currentTime = 0;
      turnSound.play();
    }

    function startGame() {
      engineSound.volume = 0.2;
      engineSound.loop = true;
      engineSound.play();
      startSound.play();

      isGameOver = false;
      score = 0;
      playerX = 135;
      player.style.left = playerX + 'px';
      scoreEl.textContent = `Счёт: ${score}`;
      restartBtn.style.display = 'none';

      document.querySelectorAll('.enemy').forEach(car => car.remove());

      // Анимация дороги
      const lines = document.querySelectorAll('.road-line');
      gameInterval = setInterval(() => {
        lines.forEach(line => {
          let top = parseInt(line.style.top);
          line.style.top = ((top + 4) % 600) + 'px';
        });
      }, 16);

      enemyInterval = setInterval(createEnemyCar, 1500);
    }

    window.onload = () => {
      startGame();
    };
  </script>
</body>
</html>
