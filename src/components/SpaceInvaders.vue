<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';

interface GameObject {
  x: number;
  y: number;
  width: number;
  height: number;
}

interface Alien extends GameObject {
  alive: boolean;
  type: 'basic' | 'medium' | 'boss';
  health: number;
  maxHealth: number;
}

interface PowerUp extends GameObject {
  type: 'double' | 'pierce' | 'bounce';
  active: boolean;
  velocity: number;
}

interface Bullet extends GameObject {
  pierce?: boolean;
  bounce?: boolean;
  bounceCount?: number;
}

const GAME_WIDTH = 1800;
const GAME_HEIGHT = 900;
const PLAYER_SPEED = 8;
const BULLET_SPEED = 10;
const ALIEN_SPEED = 2;
const ALIEN_DESCENT_SPEED = 20;
const POWER_UP_SPEED = 3;
const POWER_UP_DURATION = 10000; // 10 segundos
const POWER_UP_DROP_CHANCE = 0.1; // 10% de probabilidad

const player = ref<GameObject>({
  x: GAME_WIDTH / 2 - 25,
  y: GAME_HEIGHT - 60,
  width: 50,
  height: 30,
});

const bullet = ref<Bullet | null>(null);
const aliens = ref<Alien[]>([]);
const powerUps = ref<PowerUp[]>([]);
const score = ref(0);
const gameOver = ref(false);
const alienDirection = ref(1);
const activePowerUp = ref<string | null>(null);
const powerUpTimer = ref<number | null>(null);

function getAlienColorByHealth(type: string, health: number): string {
  if (type === 'boss') {
    switch (health) {
      case 4: return '#f00'; // Rojo
      case 3: return '#f80'; // Naranja
      case 2: return '#ff0'; // Amarillo
      case 1: return '#0f0'; // Verde
      default: return '#f00';
    }
  } else if (type === 'medium') {
    return health === 2 ? '#ff0' : '#0f0';
  }
  return '#0f0';
}

// Initialize aliens
function initAliens() {
  const rows = 5;
  const cols = 8;
  const aliens_array: Alien[] = [];

  for (let row = 0; row < rows; row++) {
    for (let col = 0; col < cols; col++) {
      const type = row === 0 ? 'boss' :
                  row < 2 ? 'medium' : 'basic';
      const maxHealth = type === 'boss' ? 4 :
        type === 'medium' ? 2 : 1;

      aliens_array.push({
        x: col * 80 + 200,
        y: row * 60 + 50,
        width: type === 'boss' ? 60 : 40,
        height: type === 'boss' ? 45 : 30,
        alive: true,
        type,
        health: maxHealth,
        maxHealth
      });
    }
  }
  aliens.value = aliens_array;
}

function handleKeyDown(e: KeyboardEvent) {
  keys.value.add(e.key);
  if (e.key === ' ' && !bullet.value) {
    shoot();
  }
}

function handleKeyUp(e: KeyboardEvent) {
  keys.value.delete(e.key);
}

const keys = ref(new Set<string>());

function movePlayer() {
  if (keys.value.has('ArrowLeft') && player.value.x > 0) {
    player.value.x -= PLAYER_SPEED;
  }
  if (keys.value.has('ArrowRight') && player.value.x < GAME_WIDTH - player.value.width) {
    player.value.x += PLAYER_SPEED;
  }
}

function shoot() {
  bullet.value = {
    x: player.value.x + player.value.width / 2 - 2,
    y: player.value.y,
    width: 4,
    height: 10,
    pierce: activePowerUp.value === 'pierce',
    bounce: activePowerUp.value === 'bounce',
    bounceCount: 0
  };
}

function checkCollision(rect1: GameObject | null, rect2: GameObject): boolean {
  if (!rect1) return false;
  return rect1.x < rect2.x + rect2.width &&
         rect1.x + rect1.width > rect2.x &&
         rect1.y < rect2.y + rect2.height &&
         rect1.y + rect1.height > rect2.y;
}

function spawnPowerUp(x: number, y: number) {
  if (Math.random() < POWER_UP_DROP_CHANCE) {
    const types: Array<'double' | 'pierce' | 'bounce'> = ['double', 'pierce', 'bounce'];
    const type = types[Math.floor(Math.random() * types.length)];
    powerUps.value.push({
      x,
      y,
      width: 20,
      height: 20,
      type,
      active: true,
      velocity: POWER_UP_SPEED
    });
  }
}

function activatePowerUp(type: string) {
  if (powerUpTimer.value) {
    clearTimeout(powerUpTimer.value);
  }

  activePowerUp.value = type;
  powerUpTimer.value = window.setTimeout(() => {
    activePowerUp.value = null;
    powerUpTimer.value = null;
  }, POWER_UP_DURATION);
}

function moveAliens() {
  let needsToChangeDirection = false;
  const livingAliens = aliens.value.filter(alien => alien.alive);

  livingAliens.forEach(alien => {
    const nextX = alien.x + ALIEN_SPEED * alienDirection.value;
    if (nextX <= 0 || nextX + alien.width >= GAME_WIDTH) {
      needsToChangeDirection = true;
    }
  });

  aliens.value.forEach(alien => {
    if (alien.alive) {
      alien.x += ALIEN_SPEED * alienDirection.value;
    }
  });

  if (needsToChangeDirection) {
    alienDirection.value *= -1;
    aliens.value.forEach(alien => {
      if (alien.alive) {
        alien.y += ALIEN_DESCENT_SPEED;
      }
    });
  }
}

function movePowerUps() {
  powerUps.value = powerUps.value.filter(powerUp => {
    if (!powerUp.active) return false;

    powerUp.y += powerUp.velocity;

    // Comprobar colisión con el jugador
    if (checkCollision(player.value, powerUp)) {
      activatePowerUp(powerUp.type);
      return false;
    }

    // Eliminar si sale de la pantalla
    return powerUp.y < GAME_HEIGHT;
  });
}

function handleBulletMovement() {
  if (!bullet.value) return;

  if (bullet.value.bounce && bullet.value.y <= 0 && (bullet.value.bounceCount || 0) < 2) {
    bullet.value.y = 0;
    bullet.value.bounceCount = (bullet.value.bounceCount || 0) + 1;
    return;
  }

  bullet.value.y -= BULLET_SPEED;

  if (bullet.value.y < 0 && (!bullet.value.bounce || (bullet.value.bounceCount || 0) >= 2)) {
    bullet.value = null;
  }
}

function gameLoop() {
  if (gameOver.value) return;

  movePlayer();
  moveAliens();
  movePowerUps();
  handleBulletMovement();

  // Check bullet collisions with aliens
  if (bullet.value) {
    for (const alien of aliens.value) {
      if (alien.alive && checkCollision(bullet.value, alien)) {
        alien.health--;
        if (alien.health <= 0) {
          alien.alive = false;
          score.value += alien.type === 'boss' ? 400 :
                        alien.type === 'medium' ? 200 : 100;
          spawnPowerUp(alien.x, alien.y);
        }
        if (!bullet.value.pierce) {
          bullet.value = null;
          break;
        }
      }
    }
  }

  const livingAliens = aliens.value.filter(alien => alien.alive);

  if (livingAliens.length === 0) {
    gameOver.value = true;
    return;
  }

  for (const alien of livingAliens) {
    if (alien.y + alien.height >= player.value.y) {
      gameOver.value = true;
      return;
    }
  }

  requestAnimationFrame(gameLoop);
}

onMounted(() => {
  initAliens();
  window.addEventListener('keydown', handleKeyDown);
  window.addEventListener('keyup', handleKeyUp);
  requestAnimationFrame(gameLoop);
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyDown);
  window.removeEventListener('keyup', handleKeyUp);
  if (powerUpTimer.value) {
    clearTimeout(powerUpTimer.value);
  }
});
</script>

<template>
  <div class="game-container">
    <div class="game-screen" :style="{ width: GAME_WIDTH + 'px', height: GAME_HEIGHT + 'px' }">
      <!-- Player -->
      <div class="player" :style="{
        left: player.x + 'px',
        top: player.y + 'px',
        width: player.width + 'px',
        height: player.height + 'px'
      }"></div>

      <!-- Bullet -->
      <div v-if="bullet"
           class="bullet"
           :class="{
             'bullet-pierce': bullet.pierce,
             'bullet-bounce': bullet.bounce,
             'bullet-double': activePowerUp === 'double'
           }"
           :style="{
             left: bullet.x + 'px',
             top: bullet.y + 'px',
             width: bullet.width + 'px',
             height: bullet.height + 'px'
           }">
      </div>

      <!-- Aliens -->
      <div v-for="(alien, index) in aliens"
           :key="index"
           v-show="alien.alive"
           class="alien"
           :class="{
             'alien-basic': alien.type === 'basic',
             'alien-medium': alien.type === 'medium',
             'alien-boss': alien.type === 'boss'
           }"
           :style="{
             left: alien.x + 'px',
             top: alien.y + 'px',
             width: alien.width + 'px',
             height: alien.height + 'px',
             backgroundColor: getAlienColorByHealth(alien.type, alien.health)
           }">
      </div>

      <!-- Power-ups -->
      <div v-for="(powerUp, index) in powerUps"
           :key="`powerup-${index}`"
           v-show="powerUp.active"
           class="power-up"
           :class="`power-up-${powerUp.type}`"
           :style="{
             left: powerUp.x + 'px',
             top: powerUp.y + 'px',
             width: powerUp.width + 'px',
             height: powerUp.height + 'px'
           }">
      </div>

      <!-- Score and Power-up indicator -->
      <div class="hud">
        <div class="score">Score: {{ score }}</div>
        <div v-if="activePowerUp" class="power-up-indicator">
          Power-up: {{ activePowerUp }}
        </div>
      </div>

      <!-- Game Over -->
      <div v-if="gameOver" class="game-over">
        <h2>Game Over!</h2>
        <p>Final Score: {{ score }}</p>
      </div>
    </div>
  </div>
</template>

<style scoped>
.game-container {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  background-color: #000;
  font-family: 'Press Start 2P', monospace;
}

.game-screen {
  position: relative;
  background-color: #000;
  border: 4px solid #0f0;
  overflow: hidden;
}

.player {
  position: absolute;
  background-color: #0f0;
  clip-path: polygon(50% 0%, 100% 100%, 0% 100%);
}

.bullet {
  position: absolute;
  background-color: #0f0;
}

.bullet-pierce {
  background-color: #ff0;
  height: 15px !important;
}

.bullet-bounce {
  background-color: #0ff;
  border-radius: 50%;
}

.bullet-double {
  width: 8px !important;
  background-color: #f0f;
}

.alien {
  position: absolute;
  transition: transform 0.1s, background-color 0.3s;
}

.alien-basic {
  clip-path: polygon(
    0% 25%,
    25% 0%,
    75% 0%,
    100% 25%,
    100% 75%,
    75% 100%,
    25% 100%,
    0% 75%
  );
}

.alien-medium {
  clip-path: polygon(
    20% 0%,
    80% 0%,
    100% 20%,
    100% 80%,
    80% 100%,
    20% 100%,
    0% 80%,
    0% 20%
  );
}

.alien-boss {
  clip-path: polygon(
    50% 0%,
    80% 30%,
    100% 30%,
    100% 70%,
    80% 70%,
    50% 100%,
    20% 70%,
    0% 70%,
    0% 30%,
    20% 30%
  );
}

.power-up {
  position: absolute;
  border-radius: 50%;
  animation: pulse 1s infinite;
}

@keyframes pulse {
  0% { transform: scale(1); }
  50% { transform: scale(1.2); }
  100% { transform: scale(1); }
}

.hud {
  position: absolute;
  top: 20px;
  left: 20px;
  color: #0f0;
  font-size: 20px;
}

.power-up-indicator {
  margin-top: 10px;
  color: #ff0;
}

.score {
  color: #0f0;
}

.game-over {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: #0f0;
  text-align: center;
  background-color: rgba(0, 0, 0, 0.8);
  padding: 20px;
  border: 2px solid #0f0;
}

.game-over h2 {
  font-size: 24px;
  margin-bottom: 10px;
}
</style>