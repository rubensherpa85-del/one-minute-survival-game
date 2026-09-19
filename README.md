const DEBUG = false;

const STORAGE_KEYS = {
  highScores: 'oneMinuteSurvivalHighScores',
  settings: 'oneMinuteSurvivalSettings',
  muted: 'oneMinuteSurvivalMuted'
};

const CONFIG = {
  roundDuration: 60,
  player: {
    radius: 18,
    color: '#61f0ff',
    startX: 480,
    startY: 270,
    maxHealth: 100,
    health: 100,
    moveSpeed: 220,
    attackDamage: 10,
    attackCooldown: 0.6,
    projectileSpeed: 470,
    projectileRadius: 6,
    projectileLifetime: 1.5,
    collectionRadius: 84,
    invincibilityDuration: 0.8,
    experienceRequirement: 100,
    minAttackCooldown: 0.12
  },
  enemy: {
    chaser: {
      radius: 16,
      speed: 84,
      health: 48,
      damage: 12,
      color: '#d38bff',
      score: 90,
      xp: 20,
      type: 'chaser'
    },
    runner: {
      radius: 12,
      speed: 120,
      health: 24,
      damage: 8,
      color: '#ffe66d',
      score: 45,
      xp: 15,
      type: 'runner'
    },
    brute: {
      radius: 22,
      speed: 58,
      health: 90,
      damage: 18,
      color: '#ff9d4d',
      score: 130,
      xp: 30,
      type: 'brute'
    }
  },
  spawn: {
    initialDelay: 1.3,
    minimumDelay: 0.42,
    maxActiveEnemies: 10,
    maxActiveEnemiesLate: 16,
    runnerThreshold: 12,
    bruteThreshold: 28,
    lateBoostThreshold: 45
  },
  particles: {
    max: 300,
    maxFloating: 100
  },
  screenShake: {
    base: 12,
    falloff: 0.9
  },
  gems: {
    max: 100,
    magnetSpeed: 230,
    radius: 6
  },
  debug: {
    enabled: DEBUG
  }
};

const UPGRADE_DEFS = [
  { id: 'swift-feet', name: 'Swift Feet', description: 'Increase movement speed by 15%.', icon: '⚡', apply: (player) => { player.moveSpeed *= 1.15; } },
  { id: 'sharper-shots', name: 'Sharper Shots', description: 'Increase attack damage by 20%.', icon: '🎯', apply: (player) => { player.attackDamage *= 1.2; } },
  { id: 'rapid-fire', name: 'Rapid Fire', description: 'Reduce attack cooldown by 15%.', icon: '💥', apply: (player) => { player.attackCooldown = Math.max(CONFIG.player.minAttackCooldown, player.attackCooldown * 0.85); } },
  { id: 'heavy-projectiles', name: 'Heavy Projectiles', description: 'Increase projectile radius by 30%.', icon: '🛡', apply: (player) => { player.projectileRadius *= 1.3; } },
  { id: 'vitality', name: 'Vitality', description: 'Increase maximum health by 25 and restore 25 health.', icon: '❤', apply: (player) => { player.maxHealth += 25; player.health = Math.min(player.maxHealth, player.health + 25); } },
  { id: 'magnetism', name: 'Magnetism', description: 'Increase experience collection radius by 50%.', icon: '🧲', apply: (player) => { player.collectionRadius *= 1.5; } },
  { id: 'multishot', name: 'Multishot', description: 'Add one additional projectile.', icon: '✦', apply: (player) => { player.multishot += 1; } },
  { id: 'pierce', name: 'Pierce', description: 'Projectiles can pass through one additional enemy.', icon: '🌀', apply: (player) => { player.pierce += 1; } },
  { id: 'regeneration', name: 'Regeneration', description: 'Restore 1 health every 5 seconds.', icon: '🌿', apply: (player) => { player.regenerationTimer = 5; } },
  { id: 'force-field', name: 'Force Field', description: 'Reduce collision damage by 15%.', icon: '🔒', apply: (player) => { player.damageReduction *= 0.85; } }
];

function clamp(value, min, max) {
  return Math.min(Math.max(value, min), max);
}

function lerp(start, end, amount) {
  return start + (end - start) * amount;
}

function randomRange(min, max) {
  return min + Math.random() * (max - min);
}

function distanceBetweenPoints(a, b) {
  const dx = b.x - a.x;
  const dy = b.y - a.y;
  return Math.hypot(dx, dy);
}

function circlesOverlap(a, b) {
  return distanceBetweenPoints(a, b) < a.radius + b.radius;
}

function normalizeVector(x, y) {
  const length = Math.hypot(x, y) || 1;
  return { x: x / length, y: y / length };
}

function formatTime(seconds) {
  return Math.max(0, seconds).toFixed(1);
}

function safeNumber(value, fallback = 0) {
  if (!Number.isFinite(value)) {
    return fallback;
  }
  return value;
}

class InputManager {
  constructor(game) {
    this.game = game;
    this.keys = {};
    this.joystickPointerId = null;
    this.joystickActive = false;
    this.joystickVector = { x: 0, y: 0 };
    this.bindKeyboard();
    this.bindJoystick();
  }

  bindKeyboard() {
    window.addEventListener('keydown', (event) => {
      const key = event.key.toLowerCase();
      if (['arrowup', 'arrowdown', 'arrowleft', 'arrowright', ' ', 'w', 'a', 's', 'd', 'escape', 'enter'].includes(key)) {
        event.preventDefault();
      }

      this.keys[key] = true;

      if (key === 'escape') {
        if (this.game.state === 'PLAYING') this.game.pauseGame();
        else if (this.game.state === 'PAUSED') this.game.resumeGame();
      }

      if ((key === 'enter' || key === ' ') && document.activeElement && document.activeElement.tagName === 'BUTTON') {
        document.activeElement.click();
      }
    });

    window.addEventListener('keyup', (event) => {
      this.keys[event.key.toLowerCase()] = false;
    });
  }

  bindJoystick() {
    const joystick = document.getElementById('joystick');
    const stick = document.getElementById('joystick-stick');
    if (!joystick || !stick) return;

    const resetStick = () => {
      this.joystickActive = false;
      this.joystickPointerId = null;
      this.joystickVector.x = 0;
      this.joystickVector.y = 0;
      stick.style.transform = 'translate(-50%, -50%)';
    };

    const updateFromPointer = (clientX, clientY) => {
      const rect = joystick.getBoundingClientRect();
      const cx = rect.left + rect.width / 2;
      const cy = rect.top + rect.height / 2;
      const dx = clientX - cx;
      const dy = clientY - cy;
      const maxRadius = rect.width * 0.32;
      const magnitude = Math.hypot(dx, dy) || 1;
      const clampedX = magnitude > maxRadius ? (dx / magnitude) * maxRadius : dx;
      const clampedY = magnitude > maxRadius ? (dy / magnitude) * maxRadius : dy;
      stick.style.transform = `translate(calc(-50% + ${clampedX}px), calc(-50% + ${clampedY}px))`;
      this.joystickVector.x = clamp(clampedX / maxRadius, -1, 1);
      this.joystickVector.y = clamp(clampedY / maxRadius, -1, 1);
    };

    joystick.addEventListener('pointerdown', (event) => {
      if (this.joystickPointerId !== null) return;
      this.joystickPointerId = event.pointerId;
      this.joystickActive = true;
      joystick.setPointerCapture(event.pointerId);
      updateFromPointer(event.clientX, event.clientY);
      event.preventDefault();
    });

    joystick.addEventListener('pointermove', (event) => {
      if (!this.joystickActive || this.joystickPointerId !== event.pointerId) return;
      updateFromPointer(event.clientX, event.clientY);
      event.preventDefault();
    });

    ['pointerup', 'pointercancel', 'pointerleave'].forEach((type) => {
      joystick.addEventListener(type, (event) => {
        if (this.joystickPointerId !== null && event.pointerId === this.joystickPointerId) {
          resetStick();
          if (joystick.hasPointerCapture && joystick.hasPointerCapture(event.pointerId)) {
            joystick.releasePointerCapture(event.pointerId);
          }
        }
      });
    });
  }

  getMovementVector() {
    let x = 0;
    let y = 0;

    if (this.keys['w'] || this.keys['arrowup']) y -= 1;
    if (this.keys['s'] || this.keys['arrowdown']) y += 1;
    if (this.keys['a'] || this.keys['arrowleft']) x -= 1;
    if (this.keys['d'] || this.keys['arrowright']) x += 1;

    if (this.joystickActive) {
      x += this.joystickVector.x;
      y += this.joystickVector.y;
    }

    if (x !== 0 || y !== 0) {
      const normalized = normalizeVector(x, y);
      x = normalized.x;
      y = normalized.y;
    }

    return { x, y };
  }

  setJoystickVisible(visible) {
    const joystick = document.getElementById('joystick');
    if (!joystick) return;
    joystick.classList.toggle('visible', visible);
  }
}

class AudioManager {
  constructor() {
    this.ctx = null;
    this.master = null;
    this.muted = this.loadMuteState();
  }

  loadMuteState() {
    try {
      const raw = localStorage.getItem(STORAGE_KEYS.muted);
      return raw === 'true';
    } catch {
      return false;
    }
  }

  saveMuteState() {
    try {
      localStorage.setItem(STORAGE_KEYS.muted, String(this.muted));
    } catch {
      // ignore storage failures
    }
  }

  ensureContext() {
    if (!this.ctx) {
      const AudioCtor = window.AudioContext || window.webkitAudioContext;
      if (!AudioCtor) return null;
      this.ctx = new AudioCtor();
      this.master = this.ctx.createGain();
      this.master.gain.value = this.muted ? 0 : 0.07;
      this.master.connect(this.ctx.destination);
    }

    if (this.ctx.state === 'suspended') {
      this.ctx.resume();
    }

    return this.ctx;
  }

  toggleMute() {
    this.muted = !this.muted;
    if (this.master) {
      this.master.gain.value = this.muted ? 0 : 0.07;
    }
    this.saveMuteState();
    return this.muted;
  }

  playTone({ frequency = 220, duration = 0.08, type = 'square', volume = 0.04, sweep = 0 }) {
    if (this.muted) return;
    const ctx = this.ensureContext();
    if (!ctx || !this.master) return;

    const oscillator = ctx.createOscillator();
    const gain = ctx.createGain();
    oscillator.type = type;
    oscillator.frequency.setValueAtTime(frequency, ctx.currentTime);
    if (sweep !== 0) {
      oscillator.frequency.linearRampToValueAtTime(frequency + sweep, ctx.currentTime + duration);
    }
    gain.gain.setValueAtTime(0.0001, ctx.currentTime);
    gain.gain.exponentialRampToValueAtTime(volume, ctx.currentTime + 0.01);
    gain.gain.exponentialRampToValueAtTime(0.0001, ctx.currentTime + duration);
    oscillator.connect(gain);
    gain.connect(this.master);
    oscillator.start();
    oscillator.stop(ctx.currentTime + duration);
  }

  shoot() { this.playTone({ frequency: 310, duration: 0.06, type: 'square', volume: 0.04, sweep: 30 }); }
  enemyHit() { this.playTone({ frequency: 170, duration: 0.08, type: 'triangle', volume: 0.04, sweep: -18 }); }
  enemyDefeated() { this.playTone({ frequency: 520, duration: 0.12, type: 'sawtooth', volume: 0.05, sweep: 70 }); }
  collectGem() { this.playTone({ frequency: 780, duration: 0.08, type: 'triangle', volume: 0.05, sweep: 50 }); }
  playerDamaged() { this.playTone({ frequency: 140, duration: 0.13, type: 'sawtooth', volume: 0.05, sweep: -35 }); }
  levelUp() { this.playTone({ frequency: 620, duration: 0.12, type: 'triangle', volume: 0.05, sweep: 80 }); }
  buttonClick() { this.playTone({ frequency: 480, duration: 0.05, type: 'square', volume: 0.03, sweep: 20 }); }
  victory() { this.playTone({ frequency: 660, duration: 0.12, type: 'triangle', volume: 0.05, sweep: 90 }); this.playTone({ frequency: 830, duration: 0.2, type: 'triangle', volume: 0.05, sweep: 100 }); }
  gameOver() { this.playTone({ frequency: 190, duration: 0.28, type: 'sawtooth', volume: 0.05, sweep: -100 }); }
}

class UIManager {
  constructor(game) {
    this.game = game;
    this.hud = document.getElementById('hud');
    this.timerValue = document.getElementById('timer-value');
    this.scoreValue = document.getElementById('score-value');
    this.enemyCountValue = document.getElementById('enemy-count-value');
    this.healthText = document.getElementById('health-text');
    this.healthFill = document.getElementById('health-fill');
    this.experienceText = document.getElementById('experience-text');
    this.experienceFill = document.getElementById('experience-fill');
    this.levelValue = document.getElementById('level-value');
    this.damageValue = document.getElementById('damage-value');
    this.speedValue = document.getElementById('speed-value');
    this.highScoreList = document.getElementById('high-score-list');
    this.gameOverSummary = document.getElementById('game-over-summary');
    this.victorySummary = document.getElementById('victory-summary');
    this.upgradeCards = document.getElementById('upgrade-cards');
    this.overlays = {
      menu: document.getElementById('menu-overlay'),
      instructions: document.getElementById('instructions-overlay'),
      highScores: document.getElementById('high-scores-overlay'),
      pause: document.getElementById('pause-overlay'),
      upgrade: document.getElementById('upgrade-overlay'),
      gameOver: document.getElementById('game-over-overlay'),
      victory: document.getElementById('victory-overlay')
    };
    this.lastHudState = {};
    this.bindEvents();
    this.syncMuteButtons();
    this.syncSettingsButtons();
  }

  bindEvents() {
    document.getElementById('start-button').addEventListener('click', () => this.game.startGame());
    document.getElementById('instructions-button').addEventListener('click', () => this.game.showInstructions());
    document.getElementById('high-scores-button').addEventListener('click', () => this.game.showHighScores());
    document.getElementById('instructions-back').addEventListener('click', () => this.game.showMenu());
    document.getElementById('scores-back').addEventListener('click', () => this.game.showMenu());
    document.getElementById('resume-button').addEventListener('click', () => this.game.resumeGame());
    document.getElementById('pause-menu-button').addEventListener('click', () => this.game.returnToMenu());
    document.getElementById('restart-button').addEventListener('click', () => this.game.restartGame());
    document.getElementById('game-over-menu-button').addEventListener('click', () => this.game.returnToMenu());
    document.getElementById('victory-restart-button').addEventListener('click', () => this.game.restartGame());
    document.getElementById('victory-menu-button').addEventListener('click', () => this.game.returnToMenu());
    document.getElementById('scores-clear-button').addEventListener('click', () => this.game.clearHighScores());
    document.getElementById('instructions-clear-scores').addEventListener('click', () => this.game.clearHighScores());
    document.getElementById('instructions-mute-toggle').addEventListener('click', () => {
      this.game.audioManager.toggleMute();
      this.syncMuteButtons();
    });
    document.getElementById('reduced-motion-toggle').addEventListener('click', () => {
      this.game.settings.reducedMotion = !this.game.settings.reducedMotion;
      this.game.saveSettings();
      this.syncSettingsButtons();
    });
    document.getElementById('mute-button').addEventListener('click', () => {
      this.game.audioManager.toggleMute();
      this.syncMuteButtons();
    });
    document.getElementById('pause-button').addEventListener('click', () => {
      if (this.game.state === 'PLAYING') this.game.pauseGame();
      else if (this.game.state === 'PAUSED') this.game.resumeGame();
    });
  }

  syncMuteButtons() {
    const muted = this.game.audioManager.muted;
    const muteButton = document.getElementById('mute-button');
    const instructionsMuteToggle = document.getElementById('instructions-mute-toggle');
    if (muteButton) {
      muteButton.textContent = muted ? '🔇' : '🔊';
      muteButton.setAttribute('aria-label', muted ? 'Unmute sound' : 'Mute sound');
    }
    if (instructionsMuteToggle) {
      instructionsMuteToggle.textContent = `Mute: ${muted ? 'On' : 'Off'}`;
    }
  }

  syncSettingsButtons() {
    const reduced = !!this.game.settings.reducedMotion;
    const toggle = document.getElementById('reduced-motion-toggle');
    if (toggle) toggle.textContent = `Reduced Motion: ${reduced ? 'On' : 'Off'}`;
  }

  setOverlay(name, visible) {
    Object.keys(this.overlays).forEach((key) => {
      if (key === name) {
        this.overlays[key].classList.toggle('visible', visible);
      } else {
        this.overlays[key].classList.remove('visible');
      }
    });
  }

  showHUD(show) {
    this.hud.classList.toggle('hidden', !show);
  }

  updateHUD() {
    const player = this.game.player;
    if (!player) return;

    const healthPercent = (player.health / player.maxHealth) * 100;
    const xpPercent = (player.experience / player.experienceRequired) * 100;
    const timerText = formatTime(this.game.remainingTime);

    const nextState = {
      timer: timerText,
      score: Math.floor(this.game.score),
      kills: this.game.defeatedEnemies,
      healthText: `${Math.max(0, Math.ceil(player.health))}/${player.maxHealth}`,
      xpText: `${Math.floor(player.experience)}/${player.experienceRequired}`,
      level: player.level,
      damage: Math.round(player.attackDamage),
      speed: `${(player.moveSpeed / CONFIG.player.moveSpeed).toFixed(2)}x`,
      healthPercent,
      xpPercent
    };

    if (this.lastHudState.timer !== nextState.timer) this.timerValue.textContent = timerText;
    if (this.lastHudState.score !== nextState.score) this.scoreValue.textContent = String(nextState.score);
    if (this.lastHudState.kills !== nextState.kills) this.enemyCountValue.textContent = String(nextState.kills);
    if (this.lastHudState.healthText !== nextState.healthText) this.healthText.textContent = nextState.healthText;
    if (this.lastHudState.xpText !== nextState.xpText) this.experienceText.textContent = nextState.xpText;
    if (this.lastHudState.level !== nextState.level) this.levelValue.textContent = String(nextState.level);
    if (this.lastHudState.damage !== nextState.damage) this.damageValue.textContent = String(nextState.damage);
    if (this.lastHudState.speed !== nextState.speed) this.speedValue.textContent = nextState.speed;

    this.healthFill.style.width = `${clamp(healthPercent, 0, 100)}%`;
    this.experienceFill.style.width = `${clamp(xpPercent, 0, 100)}%`;

    if (this.game.remainingTime < 15) this.timerValue.style.color = '#ffb347';
    else this.timerValue.style.color = '#edf7ff';

    if (healthPercent < 30) this.healthFill.style.background = 'linear-gradient(90deg, #ff5c7d, #ffb347)';
    else this.healthFill.style.background = 'linear-gradient(90deg, #ff5c7d, #ff8a52)';

    this.lastHudState = nextState;
  }

  renderHighScores(scores) {
    if (!scores || !scores.length) {
      this.highScoreList.innerHTML = '<li class="empty-score-state">No scores yet. Survive for a minute and set the pace.</li>';
      return;
    }

    const safeScores = scores.map((entry) => ({
      score: safeNumber(entry.score, 0),
      survivalTime: safeNumber(entry.survivalTime, 0),
      level: safeNumber(entry.level, 1),
      enemyCount: safeNumber(entry.enemyCount, 0),
      date: typeof entry.date === 'string' ? entry.date : new Date().toISOString()
    }));

    safeScores.sort((a, b) => b.score - a.score || b.survivalTime - a.survivalTime);

    this.highScoreList.innerHTML = safeScores.map((entry, index) => {
      const safeDate = new Date(entry.date);
      const dateText = Number.isNaN(safeDate.getTime()) ? 'Unknown' : safeDate.toLocaleDateString();
      return `
        <li class="score-item">
          <span class="score-rank">#${index + 1}</span>
          <span class="score-meta">${Number(entry.score).toLocaleString()} pts · Lv ${entry.level} · ${entry.enemyCount} kills · ${entry.survivalTime.toFixed(1)}s</span>
          <span class="score-score">${dateText}</span>
        </li>
      `;
    }).join('');
  }

  renderUpgradeCards(upgrades) {
    this.upgradeCards.innerHTML = '';
    upgrades.forEach((upgrade) => {
      const button = document.createElement('button');
      button.type = 'button';
      button.className = 'upgrade-card';
      button.tabIndex = 0;
      button.innerHTML = `
        <div class="upgrade-icon">${upgrade.icon}</div>
        <div><h3>${upgrade.name}</h3></div>
        <p>${upgrade.description}</p>
      `;
      button.addEventListener('click', () => this.game.applyUpgrade(upgrade.id));
      button.addEventListener('keydown', (event) => {
        if (event.key === 'Enter' || event.key === ' ') {
          event.preventDefault();
          this.game.applyUpgrade(upgrade.id);
        }
      });
      this.upgradeCards.appendChild(button);
    });
  }
}

class Particle {
  constructor(x, y, color, options = {}) {
    this.x = x;
    this.y = y;
    this.vx = randomRange(-1, 1) * (options.speed || 120);
    this.vy = randomRange(-1, 1) * (options.speed || 120);
    this.size = options.size || randomRange(1.5, 4.5);
    this.color = color;
    this.alpha = options.alpha || 1;
    this.life = options.life || 0.6;
    this.gravity = options.gravity || 0;
    this.friction = options.friction || 0.96;
  }

  update(dt) {
    this.x += this.vx * dt;
    this.y += this.vy * dt;
    this.vy += this.gravity * dt;
    this.vx *= this.friction;
    this.vy *= this.friction;
    this.alpha = Math.max(0, this.alpha - dt * 1.4);
    this.life -= dt;
  }

  draw(ctx) {
    ctx.save();
    ctx.globalAlpha = clamp(this.alpha, 0, 1);
    ctx.fillStyle = this.color;
    ctx.beginPath();
    ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2);
    ctx.fill();
    ctx.restore();
  }
}

class FloatingDamageNumber {
  constructor(x, y, text, color) {
    this.x = x;
    this.y = y;
    this.text = String(text);
    this.color = color;
    this.life = 1.1;
    this.velocityY = -24;
    this.alpha = 1;
  }

  update(dt) {
    this.y += this.velocityY * dt;
    this.velocityY *= 0.94;
    this.alpha -= dt * 1.2;
    this.life -= dt;
  }

  draw(ctx) {
    ctx.save();
    ctx.globalAlpha = clamp(this.alpha, 0, 1);
    ctx.fillStyle = this.color;
    ctx.font = 'bold 16px Arial';
    ctx.textAlign = 'center';
    ctx.fillText(this.text, this.x, this.y);
    ctx.restore();
  }
}

class ExperienceGem {
  constructor(x, y, value) {
    this.x = x;
    this.y = y;
    this.radius = CONFIG.gems.radius;
    this.value = value;
    this.pulse = Math.random() * Math.PI * 2;
    this.color = '#7cff7d';
    this.magnetSpeed = CONFIG.gems.magnetSpeed;
  }

  update(dt, game) {
    const dx = game.player.x - this.x;
    const dy = game.player.y - this.y;
    const distance = Math.hypot(dx, dy) || 1;

    if (distance < game.player.collectionRadius) {
      const strength = clamp(1 - distance / game.player.collectionRadius, 0, 1);
      this.x += (dx / distance) * this.magnetSpeed * strength * dt;
      this.y += (dy / distance) * this.magnetSpeed * strength * dt;
    }

    this.pulse += dt * 5;
  }

  draw(ctx) {
    const glow = 5 + Math.sin(this.pulse) * 3;
    ctx.save();
    ctx.translate(this.x, this.y);
    ctx.shadowBlur = glow * 3;
    ctx.shadowColor = '#7cff7d';
    ctx.fillStyle = '#7cff7d';
    ctx.beginPath();
    ctx.moveTo(0, -this.radius * 1.4);
    ctx.lineTo(this.radius * 0.95, 0);
    ctx.lineTo(0, this.radius * 1.4);
    ctx.lineTo(-this.radius * 0.95, 0);
    ctx.closePath();
    ctx.fill();
    ctx.restore();
  }
}

class Projectile {
  constructor(x, y, targetX, targetY, damage, color, speed, radius, lifetime, pierce) {
    const direction = normalizeVector(targetX - x, targetY - y);
    this.x = x;
    this.y = y;
    this.vx = direction.x * speed;
    this.vy = direction.y * speed;
    this.damage = damage;
    this.color = color;
    this.radius = radius;
    this.lifetime = lifetime;
    this.pierce = pierce;
    this.hitEnemies = new Set();
    this.angle = Math.atan2(this.vy, this.vx);
  }

  update(dt) {
    this.x += this.vx * dt;
    this.y += this.vy * dt;
    this.lifetime -= dt;
    this.angle = Math.atan2(this.vy, this.vx);
  }

  draw(ctx) {
    ctx.save();
    ctx.translate(this.x, this.y);
    ctx.rotate(this.angle);
    ctx.fillStyle = this.color;
    ctx.shadowBlur = 14;
    ctx.shadowColor = this.color;
    ctx.beginPath();
    ctx.moveTo(this.radius + 8, 0);
    ctx.lineTo(-this.radius - 2, this.radius * 0.75);
    ctx.lineTo(-this.radius - 2, -this.radius * 0.75);
    ctx.closePath();
    ctx.fill();
    ctx.restore();
  }
}

class Enemy {
  constructor(type, x, y) {
    const template = CONFIG.enemy[type];
    this.type = type;
    this.x = x;
    this.y = y;
    this.radius = template.radius;
    this.speed = template.speed;
    this.health = template.health;
    this.maxHealth = template.health;
    this.damage = template.damage;
    this.color = template.color;
    this.scoreValue = template.score;
    this.experienceValue = template.xp;
    this.hitFlashTimer = 0;
    this.id = `${type}-${Math.random().toString(16).slice(2)}`;
    this.alive = true;
  }

  update(dt, game) {
    const dx = game.player.x - this.x;
    const dy = game.player.y - this.y;
    const dist = Math.hypot(dx, dy) || 1;
    const nx = dx / dist;
    const ny = dy / dist;
    this.x += nx * this.speed * dt;
    this.y += ny * this.speed * dt;
    this.hitFlashTimer = Math.max(0, this.hitFlashTimer - dt);
  }

  draw(ctx) {
    if (!this.alive) return;
    const pulse = 1 + Math.sin(Date.now() * 0.006 + this.x) * 0.08;
    ctx.save();
    ctx.translate(this.x, this.y);
    ctx.scale(pulse, pulse);
    ctx.fillStyle = this.hitFlashTimer > 0 ? '#ffffff' : this.color;
    ctx.shadowBlur = 18;
    ctx.shadowColor = this.color;
    ctx.beginPath();
    ctx.arc(0, 0, this.radius, 0, Math.PI * 2);
    ctx.fill();
    ctx.restore();

    if (this.health < this.maxHealth || this.hitFlashTimer > 0) {
      const width = this.radius * 2.3;
      const ratio = clamp(this.health / this.maxHealth, 0, 1);
      const barX = this.x - width / 2;
      const barY = this.y - this.radius - 14;
      ctx.fillStyle = 'rgba(10,15,22,0.8)';
      ctx.fillRect(barX, barY, width, 5);
      ctx.fillStyle = '#ff5c7d';
      ctx.fillRect(barX, barY, width * ratio, 5);
    }
  }
}

class Player {
  constructor(game) {
    this.game = game;
    this.x = CONFIG.player.startX;
    this.y = CONFIG.player.startY;
    this.radius = CONFIG.player.radius;
    this.color = CONFIG.player.color;
    this.maxHealth = CONFIG.player.maxHealth;
    this.health = CONFIG.player.health;
    this.moveSpeed = CONFIG.player.moveSpeed;
    this.attackDamage = CONFIG.player.attackDamage;
    this.attackCooldown = CONFIG.player.attackCooldown;
    this.attackTimer = 0;
    this.projectileSpeed = CONFIG.player.projectileSpeed;
    this.projectileRadius = CONFIG.player.projectileRadius;
    this.projectileLifetime = CONFIG.player.projectileLifetime;
    this.collectionRadius = CONFIG.player.collectionRadius;
    this.experience = 0;
    this.experienceRequired = CONFIG.player.experienceRequirement;
    this.level = 1;
    this.invincibilityTimer = 0;
    this.latestDirection = { x: 1, y: 0 };
    this.multishot = 0;
    this.pierce = 0;
    this.damageReduction = 1;
    this.regenerationTimer = 0;
    this.attackRange = 500;
  }

  reset() {
    this.x = CONFIG.player.startX;
    this.y = CONFIG.player.startY;
    this.radius = CONFIG.player.radius;
    this.color = CONFIG.player.color;
    this.maxHealth = CONFIG.player.maxHealth;
    this.health = CONFIG.player.health;
    this.moveSpeed = CONFIG.player.moveSpeed;
    this.attackDamage = CONFIG.player.attackDamage;
    this.attackCooldown = CONFIG.player.attackCooldown;
    this.attackTimer = 0;
    this.projectileSpeed = CONFIG.player.projectileSpeed;
    this.projectileRadius = CONFIG.player.projectileRadius;
    this.projectileLifetime = CONFIG.player.projectileLifetime;
    this.collectionRadius = CONFIG.player.collectionRadius;
    this.experience = 0;
    this.experienceRequired = CONFIG.player.experienceRequirement;
    this.level = 1;
    this.invincibilityTimer = 0;
    this.latestDirection = { x: 1, y: 0 };
    this.multishot = 0;
    this.pierce = 0;
    this.damageReduction = 1;
    this.regenerationTimer = 0;
    this.attackRange = 500;
  }

  update(dt, game) {
    const move = game.input.getMovementVector();
    const magnitude = Math.hypot(move.x, move.y) || 1;
    if (move.x !== 0 || move.y !== 0) {
      this.latestDirection = { x: move.x / magnitude, y: move.y / magnitude };
      this.x += move.x * this.moveSpeed * dt;
      this.y += move.y * this.moveSpeed * dt;
    }

    this.x = clamp(this.x, this.radius, game.width - this.radius);
    this.y = clamp(this.y, this.radius, game.height - this.radius);

    this.attackTimer = Math.max(0, this.attackTimer - dt);
    this.invincibilityTimer = Math.max(0, this.invincibilityTimer - dt);

    if (this.regenerationTimer > 0) {
      this.regenerationTimer -= dt;
      if (this.regenerationTimer <= 0) {
        this.health = Math.min(this.maxHealth, this.health + 1);
        this.regenerationTimer = 5;
      }
    }

    if (this.attackTimer <= 0) {
      const nearest = this.findNearestEnemy(game.enemies);
      if (nearest) {
        this.fireAtTarget(nearest, game);
        this.attackTimer = this.attackCooldown;
      }
    }
  }

  findNearestEnemy(enemies) {
    let nearest = null;
    let bestDistance = Infinity;
    for (const enemy of enemies) {
      if (!enemy.alive) continue;
      const dist = distanceBetweenPoints(this, enemy);
      if (dist < bestDistance) {
        nearest = enemy;
        bestDistance = dist;
      }
    }
    return nearest;
  }

  fireAtTarget(target, game) {
    if (!target || !target.alive) return;
    const count = 1 + this.multishot;
    const baseAngles = [];
    for (let i = 0; i < count; i++) {
      if (count === 1) {
        baseAngles.push(0);
      } else {
        const spread = (i - (count - 1) / 2) * 0.22;
        baseAngles.push(spread);
      }
    }

    const baseDx = target.x - this.x;
    const baseDy = target.y - this.y;
    const baseMagnitude = Math.hypot(baseDx, baseDy) || 1;
    const forwardX = baseDx / baseMagnitude;
    const forwardY = baseDy / baseMagnitude;

    for (const angleOffset of baseAngles) {
      const cos = Math.cos(angleOffset);
      const sin = Math.sin(angleOffset);
      const dirX = forwardX * cos - forwardY * sin;
      const dirY = forwardX * sin + forwardY * cos;

      const projectile = new Projectile(
        this.x + dirX * 18,
        this.y + dirY * 18,
        this.x + dirX * 100,
        this.y + dirY * 100,
        this.attackDamage,
        '#61f0ff',
        this.projectileSpeed,
        this.projectileRadius,
        this.projectileLifetime,
        this.pierce
      );
      projectile.vx = dirX * this.projectileSpeed;
      projectile.vy = dirY * this.projectileSpeed;
      game.projectiles.push(projectile);
    }

    game.audioManager.shoot();
  }

  takeDamage(amount, game) {
    if (this.invincibilityTimer > 0) return;
    const damage = amount * this.damageReduction;
    this.health -= damage;
    this.invincibilityTimer = CONFIG.player.invincibilityDuration;
    game.createParticles(this.x, this.y, '#ff5c7d', 12, { speed: 170, gravity: 0 });
    game.floatingDamageNumbers.push(new FloatingDamageNumber(this.x, this.y - 10, `-${Math.ceil(damage)}`, '#ff5c7d'));
    game.audioManager.playerDamaged();
    game.screenShake = CONFIG.screenShake.base;
    if (this.health <= 0) {
      this.health = 0;
      game.endGame();
    }
  }

  draw(ctx) {
    if (this.invincibilityTimer > 0 && Math.floor(this.invincibilityTimer * 10) % 2 === 0) return;
    ctx.save();
    ctx.translate(this.x, this.y);
    const angle = Math.atan2(this.latestDirection.y, this.latestDirection.x);
    ctx.rotate(angle);

    ctx.fillStyle = 'rgba(0,0,0,0.35)';
    ctx.beginPath();
    ctx.ellipse(0, this.radius + 3, this.radius * 0.9, this.radius * 0.52, 0, 0, Math.PI * 2);
    ctx.fill();

    ctx.fillStyle = '#61f0ff';
    ctx.shadowBlur = 18;
    ctx.shadowColor = '#61f0ff';
    ctx.beginPath();
    ctx.moveTo(this.radius + 8, 0);
    ctx.lineTo(-this.radius * 0.6, this.radius * 0.95);
    ctx.lineTo(-this.radius * 0.8, 0);
    ctx.lineTo(-this.radius * 0.6, -this.radius * 0.95);
    ctx.closePath();
    ctx.fill();

    ctx.fillStyle = '#d7fbff';
    ctx.beginPath();
    ctx.arc(this.radius * 0.28, 0, 3.5, 0, Math.PI * 2);
    ctx.fill();
    ctx.restore();
  }
}

class Game {
  constructor() {
    this.canvas = document.getElementById('game-canvas');
    this.ctx = this.canvas.getContext('2d');
    this.width = 960;
    this.height = 540;
    this.state = 'MENU';
    this.lastTime = 0;
    this.elapsed = 0;
    this.remainingTime = CONFIG.roundDuration;
    this.score = 0;
    this.defeatedEnemies = 0;
    this.spawnTimer = CONFIG.spawn.initialDelay;
    this.maxActiveEnemies = CONFIG.spawn.maxActiveEnemies;
    this.screenShake = 0;
    this.player = new Player(this);
    this.enemies = [];
    this.projectiles = [];
    this.gems = [];
    this.particles = [];
    this.floatingDamageNumbers = [];
    this.pendingUpgrades = [];
    this.backgroundStars = Array.from({ length: 50 }, () => ({
      x: Math.random() * this.width,
      y: Math.random() * this.height,
      size: randomRange(1, 3),
      alpha: randomRange(0.3, 1),
      drift: randomRange(10, 28)
    }));

    this.input = new InputManager(this);
    this.audioManager = new AudioManager();
    this.ui = new UIManager(this);
    this.settings = { reducedMotion: false };
    this.loadSettings();
    this.resizeCanvas();
    this.bindEvents();
    this.ui.setOverlay('menu', true);
    this.ui.showHUD(false);
    this.render();
    requestAnimationFrame((time) => this.gameLoop(time));
  }

  bindEvents() {
    window.addEventListener('resize', () => this.resizeCanvas());
    document.addEventListener('visibilitychange', () => {
      if (document.hidden && this.state === 'PLAYING') {
        this.pauseGame();
      }
    });
  }

  loadSettings() {
    try {
      const raw = localStorage.getItem(STORAGE_KEYS.settings);
      if (!raw) {
        this.settings = { reducedMotion: false };
        return;
      }
      const parsed = JSON.parse(raw);
      this.settings = {
        reducedMotion: !!parsed.reducedMotion
      };
    } catch {
      this.settings = { reducedMotion: false };
    }
  }

  saveSettings() {
    try {
      localStorage.setItem(STORAGE_KEYS.settings, JSON.stringify(this.settings));
    } catch {
      // ignore storage issues
    }
  }

  resizeCanvas() {
    const rect = this.canvas.parentElement.getBoundingClientRect();
    const cssWidth = Math.max(320, rect.width);
    const cssHeight = Math.max(240, rect.height);
    const dpr = window.devicePixelRatio || 1;
    this.canvas.width = Math.round(cssWidth * dpr);
    this.canvas.height = Math.round(cssHeight * dpr);
    this.width = cssWidth;
    this.height = cssHeight;
    this.canvas.style.width = `${cssWidth}px`;
    this.canvas.style.height = `${cssHeight}px`;
    this.ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
    this.backgroundStars = this.backgroundStars.map((star) => ({ ...star, x: randomRange(0, this.width), y: randomRange(0, this.height) }));
    this.ui.updateHUD();
  }

  gameLoop(timestamp) {
    const deltaRaw = (timestamp - this.lastTime) / 1000 || 0.016;
    const dt = clamp(deltaRaw, 0, 0.05);
    this.lastTime = timestamp;

    if (this.state === 'PLAYING') {
      this.update(dt);
    } else {
      this.updateBackground(dt);
    }

    this.render();
    requestAnimationFrame((next) => this.gameLoop(next));
  }

  updateBackground(dt) {
    for (const star of this.backgroundStars) {
      star.y += star.drift * dt;
      if (star.y > this.height + 10) {
        star.y = -10;
        star.x = Math.random() * this.width;
      }
    }
  }

  startGame() {
    this.resetGame();
    this.state = 'PLAYING';
    this.ui.setOverlay('menu', false);
    this.ui.setOverlay('pause', false);
    this.ui.setOverlay('gameOver', false);
    this.ui.setOverlay('victory', false);
    this.ui.showHUD(true);
    this.audioManager.ensureContext();
    this.audioManager.buttonClick();
  }

  resetGame() {
    this.elapsed = 0;
    this.remainingTime = CONFIG.roundDuration;
    this.score = 0;
    this.defeatedEnemies = 0;
    this.spawnTimer = CONFIG.spawn.initialDelay;
    this.maxActiveEnemies = CONFIG.spawn.maxActiveEnemies;
    this.screenShake = 0;
    this.pendingUpgrades = [];
    this.enemies = [];
    this.projectiles = [];
    this.gems = [];
    this.particles = [];
    this.floatingDamageNumbers = [];
    this.player = new Player(this);
    this.player.reset();
    this.ui.updateHUD();
  }

  update(dt) {
    this.elapsed += dt;
    this.remainingTime = Math.max(0, CONFIG.roundDuration - this.elapsed);
    this.updateDifficulty();

    if (this.remainingTime <= 0) {
      this.winGame();
      return;
    }

    this.spawnTimer -= dt;
    const currentDelay = clamp(CONFIG.spawn.initialDelay - this.elapsed * 0.02, CONFIG.spawn.minimumDelay, CONFIG.spawn.initialDelay + 0.5);
    if (this.spawnTimer <= 0 && this.enemies.length < this.maxActiveEnemies) {
      this.spawnEnemy();
      this.spawnTimer = currentDelay;
    }

    this.player.update(dt, this);

    for (let i = this.projectiles.length - 1; i >= 0; i--) {
      const projectile = this.projectiles[i];
      projectile.update(dt);
      if (projectile.lifetime <= 0 || projectile.x < -20 || projectile.x > this.width + 20 || projectile.y < -20 || projectile.y > this.height + 20) {
        this.projectiles.splice(i, 1);
        continue;
      }

      for (let j = this.enemies.length - 1; j >= 0; j--) {
        const enemy = this.enemies[j];
        if (!enemy.alive || projectile.hitEnemies.has(enemy.id)) continue;
        if (distanceBetweenPoints(projectile, enemy) <= projectile.radius + enemy.radius) {
          enemy.health -= projectile.damage;
          enemy.hitFlashTimer = 0.18;
          this.audioManager.enemyHit();
          this.createParticles(projectile.x, projectile.y, '#61f0ff', 8, { speed: 120, gravity: 0, life: 0.5 });
          this.floatingDamageNumbers.push(new FloatingDamageNumber(projectile.x, projectile.y, `-${Math.ceil(projectile.damage)}`, '#61f0ff'));
          projectile.hitEnemies.add(enemy.id);

          if (projectile.pierce > 0) {
            projectile.pierce -= 1;
            if (projectile.pierce <= 0) {
              this.projectiles.splice(i, 1);
              break;
            }
          } else {
            this.projectiles.splice(i, 1);
            break;
          }

          if (enemy.health <= 0) {
            this.defeatEnemy(enemy);
          }
        }
      }
    }

    for (let i = this.enemies.length - 1; i >= 0; i--) {
      const enemy = this.enemies[i];
      if (!enemy.alive) {
        this.enemies.splice(i, 1);
        continue;
      }
      enemy.update(dt, this);

      if (circlesOverlap(enemy, this.player)) {
        const push = normalizeVector(this.player.x - enemy.x, this.player.y - enemy.y);
        enemy.x += push.x * 8;
        enemy.y += push.y * 8;
        this.player.takeDamage(enemy.damage, this);
        if (this.player.health <= 0) return;
      }
    }

    for (let i = this.gems.length - 1; i >= 0; i--) {
      const gem = this.gems[i];
      gem.update(dt, this);
      if (distanceBetweenPoints(gem, this.player) <= gem.radius + this.player.radius + this.player.collectionRadius * 0.15) {
        this.player.experience += gem.value;
        this.audioManager.collectGem();
        this.createParticles(gem.x, gem.y, '#7cff7d', 12, { speed: 90, gravity: 0, life: 0.7 });
        this.gems.splice(i, 1);
        this.checkLevelUp();
      }
    }

    for (let i = this.particles.length - 1; i >= 0; i--) {
      const particle = this.particles[i];
      particle.update(dt);
      if (particle.life <= 0 || this.particles.length > CONFIG.particles.max) {
        this.particles.splice(i, 1);
      }
    }

    for (let i = this.floatingDamageNumbers.length - 1; i >= 0; i--) {
      const number = this.floatingDamageNumbers[i];
      number.update(dt);
      if (number.life <= 0 || number.alpha <= 0) {
        this.floatingDamageNumbers.splice(i, 1);
      }
    }

    if (this.screenShake > 0) {
      this.screenShake *= CONFIG.screenShake.falloff;
    }

    this.ui.updateHUD();
  }

  updateDifficulty() {
    if (this.elapsed < CONFIG.spawn.runnerThreshold) {
      this.maxActiveEnemies = CONFIG.spawn.maxActiveEnemies;
    } else if (this.elapsed < CONFIG.spawn.bruteThreshold) {
      this.maxActiveEnemies = CONFIG.spawn.maxActiveEnemies + 2;
    } else if (this.elapsed < CONFIG.spawn.lateBoostThreshold) {
      this.maxActiveEnemies = CONFIG.spawn.maxActiveEnemies + 4;
    } else {
      this.maxActiveEnemies = CONFIG.spawn.maxActiveEnemiesLate;
    }
  }

  spawnEnemy() {
    const edge = Math.floor(Math.random() * 4);
    let x = 0;
    let y = 0;

    if (edge === 0) {
      x = randomRange(-30, this.width + 30);
      y = -30;
    } else if (edge === 1) {
      x = this.width + 30;
      y = randomRange(-30, this.height + 30);
    } else if (edge === 2) {
      x = randomRange(-30, this.width + 30);
      y = this.height + 30;
    } else {
      x = -30;
      y = randomRange(-30, this.height + 30);
    }

    let type = 'chaser';
    const elapsed = this.elapsed;
    if (elapsed >= CONFIG.spawn.bruteThreshold && Math.random() < 0.28) type = 'brute';
    else if (elapsed >= CONFIG.spawn.runnerThreshold && Math.random() < 0.5) type = 'runner';

    const enemy = new Enemy(type, x, y);
    const minDistanceToPlayer = enemy.radius + this.player.radius + 90;
    if (distanceBetweenPoints(enemy, this.player) < minDistanceToPlayer) {
      enemy.x = this.width / 2 + randomRange(-80, 80);
      enemy.y = this.height / 2 + randomRange(-80, 80);
    }

    this.enemies.push(enemy);
    if (type === 'brute') {
      this.createParticles(enemy.x, enemy.y, '#ff9d4d', 16, { speed: 150, gravity: 0, life: 0.7 });
      this.screenShake = 10;
    }
  }

  defeatEnemy(enemy) {
    if (!enemy || !enemy.alive) return;
    enemy.alive = false;
    this.score += enemy.scoreValue;
    this.defeatedEnemies += 1;
    this.audioManager.enemyDefeated();
    this.createParticles(enemy.x, enemy.y, enemy.color, 16, { speed: 150, gravity: 0, life: 0.8 });
    this.floatingDamageNumbers.push(new FloatingDamageNumber(enemy.x, enemy.y - 16, `+${enemy.scoreValue}`, '#ffe66d'));

    const gemCount = Math.max(1, Math.round(enemy.experienceValue / 10));
    for (let i = 0; i < gemCount; i++) {
      this.gems.push(new ExperienceGem(enemy.x + randomRange(-18, 18), enemy.y + randomRange(-18, 18), enemy.experienceValue));
    }

    const index = this.enemies.indexOf(enemy);
    if (index >= 0) this.enemies.splice(index, 1);
  }

  checkLevelUp() {
    while (this.player.experience >= this.player.experienceRequired) {
      this.player.experience -= this.player.experienceRequired;
      this.player.level += 1;
      this.player.experienceRequired = Math.floor(100 * Math.pow(1.25, this.player.level - 1));
      this.showLevelUp();
      return;
    }
  }

  showLevelUp() {
    this.state = 'LEVEL_UP';
    const choices = this.getRandomUpgradeChoices(3);
    this.pendingUpgrades = choices;
    this.ui.renderUpgradeCards(choices);
    this.ui.setOverlay('upgrade', true);
    this.audioManager.levelUp();
  }

  getRandomUpgradeChoices(count) {
    const selectedIds = new Set(this.pendingUpgrades.map((entry) => entry.id));
    const pool = UPGRADE_DEFS.filter((upgrade) => !selectedIds.has(upgrade.id));
    const shuffled = [...pool].sort(() => Math.random() - 0.5);
    return shuffled.slice(0, count).map((upgrade) => ({ ...upgrade }));
  }

  applyUpgrade(id) {
    const upgrade = this.pendingUpgrades.find((entry) => entry.id === id);
    if (!upgrade) return;
    const player = this.player;
    const def = UPGRADE_DEFS.find((entry) => entry.id === id);
    if (def && typeof def.apply === 'function') {
      def.apply(player);
    }
    this.pendingUpgrades = [];
    this.ui.setOverlay('upgrade', false);
    this.state = 'PLAYING';
    this.audioManager.buttonClick();
  }

  showInstructions() {
    this.state = 'INSTRUCTIONS';
    this.ui.setOverlay('menu', false);
    this.ui.setOverlay('instructions', true);
    this.ui.showHUD(false);
  }

  showHighScores() {
    this.state = 'HIGH_SCORES';
    this.ui.renderHighScores(this.loadHighScores());
    this.ui.setOverlay('menu', false);
    this.ui.setOverlay('highScores', true);
    this.ui.showHUD(false);
  }

  showMenu() {
    this.state = 'MENU';
    this.ui.setOverlay('instructions', false);
    this.ui.setOverlay('highScores', false);
    this.ui.setOverlay('pause', false);
    this.ui.setOverlay('upgrade', false);
    this.ui.setOverlay('gameOver', false);
    this.ui.setOverlay('victory', false);
    this.ui.setOverlay('menu', true);
    this.ui.showHUD(false);
  }

  pauseGame() {
    if (this.state !== 'PLAYING') return;
    this.state = 'PAUSED';
    this.ui.setOverlay('pause', true);
    this.ui.setOverlay('upgrade', false);
    this.ui.showHUD(true);
  }

  resumeGame() {
    if (this.state !== 'PAUSED') return;
    this.state = 'PLAYING';
    this.ui.setOverlay('pause', false);
    this.ui.showHUD(true);
  }

  restartGame() {
    this.audioManager.buttonClick();
    this.startGame();
  }

  returnToMenu() {
    this.state = 'MENU';
    this.audioManager.buttonClick();
    this.ui.setOverlay('pause', false);
    this.ui.setOverlay('upgrade', false);
    this.ui.setOverlay('gameOver', false);
    this.ui.setOverlay('victory', false);
    this.ui.setOverlay('menu', true);
    this.ui.showHUD(false);
    this.resetGame();
  }

  endGame() {
    if (this.state === 'GAME_OVER' || this.state === 'VICTORY') return;
    this.state = 'GAME_OVER';
    this.ui.setOverlay('gameOver', true);
    this.ui.showHUD(true);
    this.ui.gameOverSummary.innerHTML = `
      <strong>Score:</strong> ${Math.floor(this.score)}<br>
      <strong>Time survived:</strong> ${this.elapsed.toFixed(1)}s<br>
      <strong>Level reached:</strong> ${this.player.level}<br>
      <strong>Enemies defeated:</strong> ${this.defeatedEnemies}<br>
      <strong>Result:</strong> The arena got the better of you.
    `;
    this.saveHighScore();
    this.ui.renderHighScores(this.loadHighScores());
    this.audioManager.gameOver();
    this.createParticles(this.player.x, this.player.y, '#ff5c7d', 22, { speed: 200, gravity: 0, life: 0.9 });
    this.screenShake = CONFIG.screenShake.base * 1.3;
  }

  winGame() {
    if (this.state === 'VICTORY' || this.state === 'GAME_OVER') return;
    this.state = 'VICTORY';
    this.ui.setOverlay('victory', true);
    this.ui.showHUD(true);
    this.ui.victorySummary.innerHTML = `
      <strong>Score:</strong> ${Math.floor(this.score)}<br>
      <strong>Time survived:</strong> ${this.elapsed.toFixed(1)}s<br>
      <strong>Final level:</strong> ${this.player.level}<br>
      <strong>Enemies defeated:</strong> ${this.defeatedEnemies}<br>
      <strong>Result:</strong> You survived the full minute.
    `;
    this.saveHighScore();
    this.ui.renderHighScores(this.loadHighScores());
    this.audioManager.victory();
    this.createParticles(this.player.x, this.player.y, '#7cff7d', 26, { speed: 200, gravity: 0, life: 1.1 });
    this.screenShake = CONFIG.screenShake.base * 1.5;
  }

  saveHighScore() {
    try {
      const existing = JSON.parse(localStorage.getItem(STORAGE_KEYS.highScores) || '[]');
      const entry = {
        score: Math.floor(this.score),
        survivalTime: Number(this.elapsed.toFixed(1)),
        level: this.player.level,
        enemyCount: this.defeatedEnemies,
        date: new Date().toISOString()
      };
      const next = Array.isArray(existing) ? existing : [];
      next.push(entry);
      next.sort((a, b) => b.score - a.score || b.survivalTime - a.survivalTime);
      const trimmed = next.slice(0, 10);
      localStorage.setItem(STORAGE_KEYS.highScores, JSON.stringify(trimmed));
    } catch {
      // ignore storage issues
    }
  }

  loadHighScores() {
    try {
      const raw = localStorage.getItem(STORAGE_KEYS.highScores);
      if (!raw) return [];
      const parsed = JSON.parse(raw);
      if (!Array.isArray(parsed)) return [];
      return parsed.filter((entry) => entry && Number.isFinite(entry.score));
    } catch {
      return [];
    }
  }

  clearHighScores() {
    const confirmed = window.confirm('Clear all high scores?');
    if (!confirmed) return;
    try {
      localStorage.removeItem(STORAGE_KEYS.highScores);
    } catch {
      // ignore storage issues
    }
    this.ui.renderHighScores([]);
  }

  createParticles(x, y, color, count, options = {}) {
    if (this.particles.length >= CONFIG.particles.max) return;
    for (let i = 0; i < count; i++) {
      const particle = new Particle(x, y, color, {
        speed: options.speed || 120,
        gravity: options.gravity || 0,
        life: options.life || 0.6,
        size: options.size || randomRange(2, 5),
        friction: 0.95,
        alpha: 1
      });
      particle.vx = randomRange(-1, 1) * (options.speed || 120);
      particle.vy = randomRange(-1, 1) * (options.speed || 120);
      this.particles.push(particle);
    }
  }

  render() {
    const ctx = this.ctx;
    ctx.clearRect(0, 0, this.width, this.height);
    this.drawBackground(ctx);

    if (this.state !== 'MENU' && this.state !== 'INSTRUCTIONS' && this.state !== 'HIGH_SCORES') {
      ctx.save();
      const shakeX = this.settings.reducedMotion ? 0 : (Math.random() - 0.5) * this.screenShake;
      const shakeY = this.settings.reducedMotion ? 0 : (Math.random() - 0.5) * this.screenShake;
      ctx.translate(shakeX, shakeY);

      ctx.strokeStyle = 'rgba(97, 240, 255, 0.14)';
      ctx.lineWidth = 2;
      ctx.strokeRect(12, 12, this.width - 24, this.height - 24);

      for (const gem of this.gems) gem.draw(ctx);
      for (const projectile of this.projectiles) projectile.draw(ctx);
      for (const enemy of this.enemies) enemy.draw(ctx);
      if (this.player) this.player.draw(ctx);
      for (const particle of this.particles) particle.draw(ctx);
      for (const damage of this.floatingDamageNumbers) damage.draw(ctx);

      ctx.restore();
    }

    if (DEBUG) this.renderDebugInfo();
  }

  drawBackground(ctx) {
    const gradient = ctx.createLinearGradient(0, 0, this.width, this.height);
    gradient.addColorStop(0, '#0b1e2d');
    gradient.addColorStop(1, '#02060b');
    ctx.fillStyle = gradient;
    ctx.fillRect(0, 0, this.width, this.height);

    ctx.save();
    ctx.strokeStyle = 'rgba(97, 240, 255, 0.07)';
    ctx.lineWidth = 1;
    for (let x = 0; x <= this.width; x += 32) {
      ctx.beginPath();
      ctx.moveTo(x, 0);
      ctx.lineTo(x, this.height);
      ctx.stroke();
    }
    for (let y = 0; y <= this.height; y += 32) {
      ctx.beginPath();
      ctx.moveTo(0, y);
      ctx.lineTo(this.width, y);
      ctx.stroke();
    }

    for (const star of this.backgroundStars) {
      ctx.fillStyle = `rgba(255,255,255,${star.alpha})`;
      ctx.beginPath();
      ctx.arc(star.x, star.y, star.size, 0, Math.PI * 2);
      ctx.fill();
    }
    ctx.restore();
  }

  renderDebugInfo() {
    const ctx = this.ctx;
    ctx.save();
    ctx.fillStyle = 'rgba(0,0,0,0.45)';
    ctx.fillRect(12, 12, 200, 120);
    ctx.font = '12px monospace';
    ctx.fillStyle = '#fff';
    const spawnInterval = Math.max(this.spawnTimer, 0).toFixed(2);
    ctx.fillText(`FPS: ${Math.round(1 / Math.max(0.016, this.lastFrameDt || 0.016))}`, 20, 30);
    ctx.fillText(`State: ${this.state}`, 20, 46);
    ctx.fillText(`Player: ${this.player.x.toFixed(1)}, ${this.player.y.toFixed(1)}`, 20, 62);
    ctx.fillText(`Enemies: ${this.enemies.length}`, 20, 78);
    ctx.fillText(`Projectiles: ${this.projectiles.length}`, 20, 94);
    ctx.fillText(`Particles: ${this.particles.length}`, 20, 110);
    ctx.fillText(`Spawn: ${spawnInterval}s`, 20, 126);
    ctx.restore();
  }
}

window.addEventListener('load', () => {
  const game = new Game();
  window.game = game;
  const width = window.innerWidth;
  const useJoystick = width <= 820 || 'ontouchstart' in window;
  game.input.setJoystickVisible(useJoystick);
  const initialScores = game.loadHighScores();
  game.ui.renderHighScores(initialScores);
  game.ui.setOverlay('menu', true);
  game.ui.showHUD(false);
  if (!game.audioManager.muted) game.audioManager.ensureContext();
});

window.addEventListener('resize', () => {
  const game = window.game;
  if (!game) return;
  const useJoystick = window.innerWidth <= 820 || 'ontouchstart' in window;
  game.input.setJoystickVisible(useJoystick && game.state === 'PLAYING');
});

if (DEBUG) {
  window.__debugGame = true;
}

