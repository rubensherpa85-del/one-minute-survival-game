const DEBUG = false;

const STORAGE_KEYS = {
  highScores: 'one-minute-survival.high-scores',
  settings: 'one-minute-survival.settings',
  muted: 'one-minute-survival.muted'
};

const CONFIG = {
  roundDuration: 60,
  player: {
    startX: 480,
    startY: 270,
    radius: 18,
    color: '#61f0ff',
    maxHealth: 100,
    health: 100,
    moveSpeed: 220,
    attackDamage: 20,
    attackCooldown: 0.5,
    projectileSpeed: 470,
    projectileSize: 7,
    collectionRadius: 80,
    invincibility: 0.8,
    minAttackCooldown: 0.12,
    baseExperienceRequired: 100,
    baseLevel: 1
  },
  enemy: {
    brute: { radius: 20, speed: 55, health: 90, damage: 18, color: '#ff9d4d', score: 120, xp: 30, type: 'Brute' },
    runner: { radius: 12, speed: 115, health: 28, damage: 8, color: '#ffe66d', score: 45, xp: 15, type: 'Runner' },
    chaser: { radius: 16, speed: 85, health: 55, damage: 12, color: '#d38bff', score: 90, xp: 20, type: 'Chaser' }
  },
  spawn: {
    startDelay: 1.2,
    minDelay: 0.35,
    maxEnemies: 12,
    maxEnemiesLate: 18,
    difficultEnemyThresholds: {
      runner: 15,
      brute: 30,
      maxEnemiesBoost: 45
    }
  },
  projectile: {
    lifetime: 1.5,
    maxProjectiles: 100,
    impactParticleCount: 10,
    radius: 6
  },
  gem: {
    baseValue: 12,
    radius: 6,
    magnetSpeed: 210,
    maxGems: 90
  },
  particles: {
    maxParticles: 260,
    maxFloatingTexts: 80
  },
  screenShake: {
    intensity: 8,
    falloff: 0.9
  },
  upgrades: {
    swiftFeet: 1.15,
    sharperShots: 1.2,
    rapidFire: 0.85,
    heavyProjectiles: 1.3,
    vitality: 1.25,
    magnetism: 1.5,
    multishot: 1,
    pierce: 1,
    regeneration: 1,
    forceField: 0.85
  }
};

const upgradeCatalog = [
  { id: 'swift-feet', name: 'Swift Feet', description: 'Increase movement speed by 15%.', icon: '⚡', sort: 'speed', effect: (player) => { player.moveSpeed *= CONFIG.upgrades.swiftFeet; } },
  { id: 'sharper-shots', name: 'Sharper Shots', description: 'Increase attack damage by 20%.', icon: '🎯', sort: 'damage', effect: (player) => { player.attackDamage *= CONFIG.upgrades.sharperShots; } },
  { id: 'rapid-fire', name: 'Rapid Fire', description: 'Reduce attack cooldown by 15%.', icon: '💥', sort: 'rate', effect: (player) => { player.attackCooldown = Math.max(CONFIG.player.minAttackCooldown, player.attackCooldown * CONFIG.upgrades.rapidFire); } },
  { id: 'heavy-projectiles', name: 'Heavy Projectiles', description: 'Increase projectile size by 30%.', icon: '🛡', sort: 'projectile', effect: (player) => { player.projectileSize *= CONFIG.upgrades.heavyProjectiles; } },
  { id: 'vitality', name: 'Vitality', description: 'Increase max health by 25 and restore 25 health.', icon: '❤', sort: 'vitality', effect: (player) => { player.maxHealth += 25; player.health = Math.min(player.maxHealth, player.health + 25); } },
  { id: 'magnetism', name: 'Magnetism', description: 'Increase experience collection radius by 50%.', icon: '🧲', sort: 'magnet', effect: (player) => { player.collectionRadius *= CONFIG.upgrades.magnetism; } },
  { id: 'multishot', name: 'Multishot', description: 'Fire one additional projectile toward a nearby enemy.', icon: '✦', sort: 'multi', effect: (player) => { player.multishotLevel += 1; } },
  { id: 'pierce', name: 'Pierce', description: 'Projectiles can pass through one additional enemy.', icon: '🌀', sort: 'pierce', effect: (player) => { player.pierceLevel += 1; } },
  { id: 'regeneration', name: 'Regeneration', description: 'Restore 1 health every 5 seconds.', icon: '🌿', sort: 'regen', effect: (player) => { player.regenerationLevel += 1; } },
  { id: 'force-field', name: 'Force Field', description: 'Reduce collision damage by 15%.', icon: '🔒', sort: 'shield', effect: (player) => { player.forceFieldLevel += 1; } }
];

function clamp(value, min, max) {
  return Math.min(Math.max(value, min), max);
}

function randomRange(min, max) {
  if (!Number.isFinite(min) || !Number.isFinite(max)) {
    return 0;
  }
  return min + Math.random() * (max - min);
}

function distanceBetweenPoints(x1, y1, x2, y2) {
  const dx = x2 - x1;
  const dy = y2 - y1;
  return Math.hypot(dx, dy);
}

function circlesOverlap(a, b) {
  return distanceBetweenPoints(a.x, a.y, b.x, b.y) < a.radius + b.radius;
}

function lerp(a, b, t) {
  return a + (b - a) * t;
}

function normalizeVector(x, y) {
  const length = Math.hypot(x, y) || 1;
  return { x: x / length, y: y / length };
}

class Upgrade {
  constructor(definition) {
    this.id = definition.id;
    this.name = definition.name;
    this.description = definition.description;
    this.icon = definition.icon;
    this.effect = definition.effect;
  }

  apply(player) {
    if (typeof this.effect === 'function') {
      this.effect(player);
    }
  }
}

class InputManager {
  constructor(game) {
    this.game = game;
    this.keys = {};
    this.joystickVector = { x: 0, y: 0 };
    this.joystickActive = false;
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
        if (this.game.state === 'PLAYING') {
          this.game.pauseGame();
        } else if (this.game.state === 'PAUSED') {
          this.game.resumeGame();
        }
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
    if (!joystick || !stick) {
      return;
    }

    const updateStick = (clientX, clientY) => {
      const rect = joystick.getBoundingClientRect();
      const centerX = rect.left + rect.width / 2;
      const centerY = rect.top + rect.height / 2;
      const dx = clientX - centerX;
      const dy = clientY - centerY;
      const maxDistance = rect.width * 0.33;
      const length = Math.hypot(dx, dy);
      const clampedX = length > maxDistance ? (dx / length) * maxDistance : dx;
      const clampedY = length > maxDistance ? (dy / length) * maxDistance : dy;
      stick.style.transform = `translate(${clampedX}px, ${clampedY}px)`;
      this.joystickVector.x = clamp(clampedX / maxDistance, -1, 1);
      this.joystickVector.y = clamp(clampedY / maxDistance, -1, 1);
      if (length === 0) {
        this.joystickVector.x = 0;
        this.joystickVector.y = 0;
      }
    };

    const resetStick = () => {
      this.joystickActive = false;
      this.joystickVector.x = 0;
      this.joystickVector.y = 0;
      stick.style.transform = 'translate(-50%, -50%)';
    };

    joystick.addEventListener('pointerdown', (event) => {
      event.preventDefault();
      this.joystickActive = true;
      joystick.setPointerCapture(event.pointerId);
      updateStick(event.clientX, event.clientY);
    });

    joystick.addEventListener('pointermove', (event) => {
      if (!this.joystickActive) {
        return;
      }
      event.preventDefault();
      updateStick(event.clientX, event.clientY);
    });

    joystick.addEventListener('pointerup', (event) => {
      event.preventDefault();
      resetStick();
      joystick.releasePointerCapture?.(event.pointerId);
    });

    joystick.addEventListener('pointercancel', () => {
      resetStick();
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
}

class AudioManager {
  constructor() {
    this.ctx = null;
    this.gainNode = null;
    this.muted = this.readMutePreference();
  }

  readMutePreference() {
    try {
      const value = localStorage.getItem(STORAGE_KEYS.muted);
      return value === 'true';
    } catch (error) {
      return false;
    }
  }

  writeMutePreference() {
    try {
      localStorage.setItem(STORAGE_KEYS.muted, String(this.muted));
    } catch (error) {
      // Failure to save settings should not block gameplay.
    }
  }

  ensureContext() {
    if (!this.ctx) {
      const AudioCtx = window.AudioContext || window.webkitAudioContext;
      if (!AudioCtx) {
        return null;
      }
      this.ctx = new AudioCtx();
      this.gainNode = this.ctx.createGain();
      this.gainNode.gain.value = this.muted ? 0 : 0.09;
      this.gainNode.connect(this.ctx.destination);
    }
    if (this.ctx.state === 'suspended') {
      this.ctx.resume();
    }
    return this.ctx;
  }

  toggleMute() {
    this.muted = !this.muted;
    if (this.gainNode) {
      this.gainNode.gain.value = this.muted ? 0 : 0.09;
    }
    this.writeMutePreference();
    return this.muted;
  }

  playTone({ frequency = 220, duration = 0.08, type = 'square', volume = 0.08, slide = 0 }) {
    if (this.muted) {
      return;
    }
    const ctx = this.ensureContext();
    if (!ctx || !this.gainNode) {
      return;
    }

    const oscillator = ctx.createOscillator();
    const gain = ctx.createGain();
    oscillator.type = type;
    oscillator.frequency.setValueAtTime(frequency, ctx.currentTime);
    if (slide) {
      oscillator.frequency.linearRampToValueAtTime(frequency + slide, ctx.currentTime + duration);
    }
    gain.gain.setValueAtTime(0.0001, ctx.currentTime);
    gain.gain.exponentialRampToValueAtTime(volume, ctx.currentTime + 0.01);
    gain.gain.exponentialRampToValueAtTime(0.0001, ctx.currentTime + duration);
    oscillator.connect(gain);
    gain.connect(this.gainNode);
    oscillator.start();
    oscillator.stop(ctx.currentTime + duration);
  }

  shoot() { this.playTone({ frequency: 340, duration: 0.06, type: 'square', volume: 0.05, slide: 20 }); }
  enemyHit() { this.playTone({ frequency: 180, duration: 0.08, type: 'triangle', volume: 0.045, slide: -24 }); }
  enemyDefeated() { this.playTone({ frequency: 540, duration: 0.12, type: 'sawtooth', volume: 0.055, slide: 60 }); }
  collectGem() { this.playTone({ frequency: 820, duration: 0.08, type: 'triangle', volume: 0.05, slide: 50 }); }
  playerDamaged() { this.playTone({ frequency: 120, duration: 0.12, type: 'sawtooth', volume: 0.05, slide: -30 }); }
  levelUp() { this.playTone({ frequency: 620, duration: 0.16, type: 'triangle', volume: 0.06, slide: 80 }); this.playTone({ frequency: 820, duration: 0.18, type: 'triangle', volume: 0.05, slide: 110 }); }
  buttonClick() { this.playTone({ frequency: 440, duration: 0.05, type: 'square', volume: 0.04, slide: 15 }); }
  victory() { this.playTone({ frequency: 620, duration: 0.14, type: 'triangle', volume: 0.06, slide: 90 }); this.playTone({ frequency: 740, duration: 0.22, type: 'triangle', volume: 0.06, slide: 110 }); }
  gameOver() { this.playTone({ frequency: 180, duration: 0.3, type: 'sawtooth', volume: 0.06, slide: -80 }); }
}

class UIManager {
  constructor(game) {
    this.game = game;
    this.hud = document.getElementById('hud');
    this.timerValue = document.getElementById('timer-value');
    this.scoreValue = document.getElementById('score-value');
    this.enemyCountValue = document.getElementById('enemy-count-value');
    this.levelValue = document.getElementById('level-value');
    this.damageValue = document.getElementById('damage-value');
    this.speedValue = document.getElementById('speed-value');
    this.healthFill = document.getElementById('health-fill');
    this.experienceFill = document.getElementById('experience-fill');
    this.healthText = document.getElementById('health-text');
    this.experienceText = document.getElementById('experience-text');
    this.highScoreList = document.getElementById('high-score-list');
    this.muteButton = document.getElementById('mute-button');
    this.pauseButton = document.getElementById('pause-button');
    this.overlays = {
      menu: document.getElementById('menu-overlay'),
      instructions: document.getElementById('instructions-overlay'),
      highScores: document.getElementById('high-scores-overlay'),
      pause: document.getElementById('pause-overlay'),
      upgrade: document.getElementById('upgrade-overlay'),
      gameOver: document.getElementById('game-over-overlay'),
      victory: document.getElementById('victory-overlay')
    };
    this.upgradeCardsContainer = document.getElementById('upgrade-cards');
    this.gameOverSummary = document.getElementById('game-over-summary');
    this.victorySummary = document.getElementById('victory-summary');
    this.lastHudState = {};
    this.bindCommonUI();
  }

  bindCommonUI() {
    const startButton = document.getElementById('start-button');
    const instructionsButton = document.getElementById('instructions-button');
    const highScoresButton = document.getElementById('high-scores-button');
    const instructionsBack = document.getElementById('instructions-back');
    const scoresBack = document.getElementById('scores-back');
    const resumeButton = document.getElementById('resume-button');
    const pauseMenuButton = document.getElementById('pause-menu-button');
    const restartButton = document.getElementById('restart-button');
    const gameOverMenuButton = document.getElementById('game-over-menu-button');
    const victoryRestartButton = document.getElementById('victory-restart-button');
    const victoryMenuButton = document.getElementById('victory-menu-button');
    const instructionsMuteToggle = document.getElementById('instructions-mute-toggle');
    const reducedMotionToggle = document.getElementById('reduced-motion-toggle');
    const instructionsClearScores = document.getElementById('instructions-clear-scores');
    const scoresClearButton = document.getElementById('scores-clear-button');

    startButton.addEventListener('click', () => this.game.startGame());
    instructionsButton.addEventListener('click', () => this.game.showInstructions());
    highScoresButton.addEventListener('click', () => this.game.showHighScores());
    instructionsBack.addEventListener('click', () => this.game.showMenu());
    scoresBack.addEventListener('click', () => this.game.showMenu());
    resumeButton.addEventListener('click', () => this.game.resumeGame());
    pauseMenuButton.addEventListener('click', () => this.game.returnToMenu());
    restartButton.addEventListener('click', () => this.game.restartGame());
    gameOverMenuButton.addEventListener('click', () => this.game.returnToMenu());
    victoryRestartButton.addEventListener('click', () => this.game.restartGame());
    victoryMenuButton.addEventListener('click', () => this.game.returnToMenu());

    this.muteButton.addEventListener('click', () => {
      this.game.audioManager.toggleMute();
      this.game.ui.syncMuteButtons();
    });

    instructionsMuteToggle.addEventListener('click', () => {
      this.game.audioManager.toggleMute();
      this.game.ui.syncMuteButtons();
    });

    reducedMotionToggle.addEventListener('click', () => {
      this.game.settings.reducedMotion = !this.game.settings.reducedMotion;
      this.storeSettings();
      this.syncSettingsButtons();
    });

    instructionsClearScores.addEventListener('click', () => this.game.clearHighScores());
    scoresClearButton.addEventListener('click', () => this.game.clearHighScores());
    this.pauseButton.addEventListener('click', () => {
      if (this.game.state === 'PLAYING') {
        this.game.pauseGame();
      } else if (this.game.state === 'PAUSED') {
        this.game.resumeGame();
      }
    });
  }

  storeSettings() {
    try {
      localStorage.setItem(STORAGE_KEYS.settings, JSON.stringify(this.game.settings));
    } catch (error) {
      // ignore storage failures
    }
  }

  loadSettings() {
    try {
      const raw = localStorage.getItem(STORAGE_KEYS.settings);
      if (!raw) {
        this.game.settings = { reducedMotion: false };
        return;
      }
      const parsed = JSON.parse(raw);
      this.game.settings = {
        reducedMotion: Boolean(parsed.reducedMotion)
      };
    } catch (error) {
      this.game.settings = { reducedMotion: false };
    }
    this.syncSettingsButtons();
  }

  syncMuteButtons() {
    const muted = this.game.audioManager.muted;
    const label = muted ? '🔇 Muted' : '🔊 Sound';
    this.muteButton.textContent = muted ? '🔇' : '🔊';
    this.muteButton.setAttribute('aria-label', `Toggle sound. Currently ${muted ? 'muted' : 'on'}`);
    const instructionsToggle = document.getElementById('instructions-mute-toggle');
    if (instructionsToggle) {
      instructionsToggle.textContent = `Mute: ${muted ? 'On' : 'Off'}`;
    }
  }

  syncSettingsButtons() {
    const reduced = this.game.settings.reducedMotion;
    const toggle = document.getElementById('reduced-motion-toggle');
    if (toggle) {
      toggle.textContent = `Reduced Motion: ${reduced ? 'On' : 'Off'}`;
    }
  }

  setOverlayState(name, visible) {
    Object.keys(this.overlays).forEach((key) => {
      if (key === name) {
        this.overlays[key].classList.toggle('visible', visible);
      } else {
        this.overlays[key].classList.remove('visible');
      }
    });
  }

  updateHUD(game) {
    const player = game.player;
    const healthPercent = (player.health / player.maxHealth) * 100;
    const xpPercent = ((player.experience / player.experienceRequired) * 100);
    const timerText = (Math.max(game.remainingTime, 0)).toFixed(1);

    const hudState = {
      timer: timerText,
      score: game.score,
      kills: game.defeatedEnemies,
      health: `${Math.max(0, Math.ceil(player.health))}/${player.maxHealth}`,
      exp: `${Math.floor(player.experience)}/${player.experienceRequired}`,
      level: player.level,
      damage: Math.round(player.attackDamage),
      speed: `${(player.moveSpeed / CONFIG.player.moveSpeed).toFixed(2)}x`,
      healthPercent,
      xpPercent
    };

    if (hudState.timer !== this.lastHudState.timer) this.timerValue.textContent = timerText;
    if (hudState.score !== this.lastHudState.score) this.scoreValue.textContent = String(Math.floor(game.score));
    if (hudState.kills !== this.lastHudState.kills) this.enemyCountValue.textContent = String(game.defeatedEnemies);
    if (hudState.health !== this.lastHudState.health) this.healthText.textContent = hudState.health;
    if (hudState.exp !== this.lastHudState.exp) this.experienceText.textContent = hudState.exp;
    if (hudState.level !== this.lastHudState.level) this.levelValue.textContent = String(player.level);
    if (hudState.damage !== this.lastHudState.damage) this.damageValue.textContent = String(Math.round(player.attackDamage));
    if (hudState.speed !== this.lastHudState.speed) this.speedValue.textContent = hudState.speed;

    if (Math.abs(hudState.healthPercent - (this.lastHudState.healthPercent || 0)) > 1) this.healthFill.style.width = `${clamp(healthPercent, 0, 100)}%`;
    if (Math.abs(hudState.xpPercent - (this.lastHudState.xpPercent || 0)) > 1) this.experienceFill.style.width = `${clamp(xpPercent, 0, 100)}%`;

    if (game.remainingTime < 15) {
      this.timerValue.style.color = '#ff9d4d';
    } else {
      this.timerValue.style.color = '#edf7ff';
    }

    this.lastHudState = hudState;
  }

  showHUD(show) {
    if (show) {
      this.hud.classList.remove('hidden');
    } else {
      this.hud.classList.add('hidden');
    }
  }

  renderHighScores(scores) {
    if (!scores || scores.length === 0) {
      this.highScoreList.innerHTML = '<li class="empty-score-state">No scores yet. Survive long enough to set the pace.</li>';
      return;
    }

    this.highScoreList.innerHTML = scores
      .map((entry, index) => {
        const dateText = entry.date ? new Date(entry.date).toLocaleDateString() : 'Unknown';
        return `
          <li class="score-item">
            <span class="score-rank">#${index + 1}</span>
            <span class="score-meta">${entry.score} pts · Lv ${entry.level} · ${entry.enemyCount} kills · ${entry.survivalTime.toFixed(1)}s</span>
            <span class="score-score">${dateText}</span>
          </li>
        `;
      })
      .join('');
  }

  renderUpgradeChoices(upgrades) {
    this.upgradeCardsContainer.innerHTML = '';
    upgrades.forEach((upgrade) => {
      const button = document.createElement('button');
      button.type = 'button';
      button.className = 'upgrade-card';
      button.innerHTML = `
        <div class="upgrade-icon">${upgrade.icon}</div>
        <div>
          <h3>${upgrade.name}</h3>
        </div>
        <p>${upgrade.description}</p>
      `;
      button.addEventListener('click', () => {
        this.game.applyUpgrade(upgrade.id);
      });
      this.upgradeCardsContainer.appendChild(button);
    });
  }

  setSummaryText(content, element) {
    element.innerHTML = content;
  }
}

class Particle {
  constructor(x, y, color, options = {}) {
    this.x = x;
    this.y = y;
    this.vx = randomRange(-60, 60) * (options.vxFactor || 1);
    this.vy = randomRange(-60, 60) * (options.vyFactor || 1);
    this.size = options.size || randomRange(2, 5);
    this.color = color;
    this.alpha = options.alpha || 1;
    this.lifetime = options.lifetime || 0.7;
    this.gravity = options.gravity || 80;
    this.friction = options.friction || 0.96;
    this.fade = options.fade || 1.4;
  }

  update(dt) {
    this.x += this.vx * dt;
    this.y += this.vy * dt;
    this.vy += this.gravity * dt;
    this.vx *= this.friction;
    this.vy *= this.friction;
    this.alpha = Math.max(0, this.alpha - this.fade * dt);
    this.lifetime -= dt;
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

class Projectile {
  constructor(x, y, targetX, targetY, damage, color, speed, radius, pierceLevel = 0) {
    const direction = normalizeVector(targetX - x, targetY - y);
    this.x = x;
    this.y = y;
    this.vx = direction.x * speed;
    this.vy = direction.y * speed;
    this.radius = radius;
    this.damage = damage;
    this.color = color;
    this.lifetime = CONFIG.projectile.lifetime;
    this.pierceLevel = pierceLevel;
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
    ctx.beginPath();
    ctx.moveTo(this.radius + 7, 0);
    ctx.lineTo(-this.radius - 3, this.radius * 0.7);
    ctx.lineTo(-this.radius - 3, -this.radius * 0.7);
    ctx.closePath();
    ctx.fill();
    ctx.restore();
  }
}

class ExperienceGem {
  constructor(x, y, value) {
    this.x = x;
    this.y = y;
    this.radius = CONFIG.gem.radius;
    this.value = value;
    this.magnetSpeed = CONFIG.gem.magnetSpeed;
    this.pulse = Math.random() * Math.PI * 2;
    this.color = '#7cff7d';
  }

  update(dt, game) {
    const player = game.player;
    const dx = player.x - this.x;
    const dy = player.y - this.y;
    const distance = Math.hypot(dx, dy) || 1;

    if (distance < player.collectionRadius) {
      const pullPower = clamp(1 - distance / player.collectionRadius, 0, 1);
      this.x += (dx / distance) * this.magnetSpeed * pullPower * dt;
      this.y += (dy / distance) * this.magnetSpeed * pullPower * dt;
    }

    this.pulse += dt * 4.5;
  }

  draw(ctx) {
    const glow = 6 + Math.sin(this.pulse) * 3;
    ctx.save();
    ctx.translate(this.x, this.y);
    ctx.shadowBlur = glow * 3;
    ctx.shadowColor = '#7cff7d';
    ctx.fillStyle = '#7cff7d';
    ctx.beginPath();
    ctx.moveTo(0, -this.radius * 1.3);
    ctx.lineTo(this.radius * 0.8, 0);
    ctx.lineTo(0, this.radius * 1.3);
    ctx.lineTo(-this.radius * 0.8, 0);
    ctx.closePath();
    ctx.fill();
    ctx.restore();
  }
}

class Enemy {
  constructor(type, x, y) {
    const template = CONFIG.enemy[type];
    this.x = x;
    this.y = y;
    this.radius = template.radius;
    this.speed = template.speed;
    this.health = template.health;
    this.maxHealth = template.health;
    this.damage = template.damage;
    this.color = template.color;
    this.enemyType = template.type;
    this.scoreValue = template.score;
    this.experienceValue = template.xp;
    this.hitFlashTimer = 0;
    this.attackCooldown = 0.6;
    this.id = Math.random().toString(16).slice(2);
    this.killed = false;
    this.alive = true;
  }

  update(dt, game) {
    if (!this.alive) {
      return;
    }
    const dx = game.player.x - this.x;
    const dy = game.player.y - this.y;
    const distance = Math.hypot(dx, dy) || 1;
    const nx = dx / distance;
    const ny = dy / distance;
    this.x += nx * this.speed * dt;
    this.y += ny * this.speed * dt;
    this.hitFlashTimer = Math.max(0, this.hitFlashTimer - dt);
  }

  draw(ctx) {
    if (!this.alive) {
      return;
    }

    ctx.save();
    ctx.translate(this.x, this.y);
    const pulse = 1 + Math.sin(Date.now() * 0.005 + this.x) * 0.08;
    ctx.scale(pulse, pulse);
    ctx.fillStyle = this.hitFlashTimer > 0 ? '#ffffff' : this.color;
    ctx.shadowBlur = 16;
    ctx.shadowColor = this.color;
    ctx.beginPath();
    ctx.arc(0, 0, this.radius, 0, Math.PI * 2);
    ctx.fill();
    ctx.restore();

    if (this.health < this.maxHealth || this.hitFlashTimer > 0) {
      const barWidth = this.radius * 2.2;
      const healthRatio = clamp(this.health / this.maxHealth, 0, 1);
      const barX = this.x - barWidth / 2;
      const barY = this.y - this.radius - 14;
      ctx.fillStyle = 'rgba(10, 15, 22, 0.8)';
      ctx.fillRect(barX, barY, barWidth, 5);
      ctx.fillStyle = '#ff5c7d';
      ctx.fillRect(barX, barY, barWidth * healthRatio, 5);
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
    this.projectileSize = CONFIG.player.projectileSize;
    this.collectionRadius = CONFIG.player.collectionRadius;
    this.experience = 0;
    this.level = CONFIG.player.baseLevel;
    this.experienceRequired = Math.floor(CONFIG.player.baseExperienceRequired * Math.pow(1.25, this.level - 1));
    this.invincibilityTimer = 0;
    this.lastMoveX = 1;
    this.lastMoveY = 0;
    this.multishotLevel = 0;
    this.pierceLevel = 0;
    this.regenerationLevel = 0;
    this.forceFieldLevel = 0;
    this.regenerationTimer = 0;
  }

  reset(game) {
    this.game = game;
    this.x = CONFIG.player.startX;
    this.y = CONFIG.player.startY;
    this.health = CONFIG.player.health;
    this.maxHealth = CONFIG.player.maxHealth;
    this.moveSpeed = CONFIG.player.moveSpeed;
    this.attackDamage = CONFIG.player.attackDamage;
    this.attackCooldown = CONFIG.player.attackCooldown;
    this.attackTimer = 0;
    this.projectileSpeed = CONFIG.player.projectileSpeed;
    this.projectileSize = CONFIG.player.projectileSize;
    this.collectionRadius = CONFIG.player.collectionRadius;
    this.experience = 0;
    this.level = 1;
    this.experienceRequired = Math.floor(CONFIG.player.baseExperienceRequired * Math.pow(1.25, this.level - 1));
    this.invincibilityTimer = 0;
    this.lastMoveX = 1;
    this.lastMoveY = 0;
    this.multishotLevel = 0;
    this.pierceLevel = 0;
    this.regenerationLevel = 0;
    this.forceFieldLevel = 0;
    this.regenerationTimer = 0;
  }

  update(dt, game) {
    const moveInput = game.input.getMovementVector();
    const dx = moveInput.x;
    const dy = moveInput.y;

    if (dx !== 0 || dy !== 0) {
      this.lastMoveX = dx;
      this.lastMoveY = dy;
      this.x += dx * this.moveSpeed * dt;
      this.y += dy * this.moveSpeed * dt;
    }

    this.x = clamp(this.x, this.radius, game.width - this.radius);
    this.y = clamp(this.y, this.radius, game.height - this.radius);

    this.attackTimer = Math.max(0, this.attackTimer - dt);
    this.invincibilityTimer = Math.max(0, this.invincibilityTimer - dt);
    this.regenerationTimer += dt;

    if (this.regenerationLevel > 0 && this.regenerationTimer >= 5) {
      this.regenerationTimer = 0;
      this.health = Math.min(this.maxHealth, this.health + this.regenerationLevel);
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
    let nearestDistance = Infinity;
    for (const enemy of enemies) {
      if (!enemy.alive) {
        continue;
      }
      const dist = distanceBetweenPoints(this.x, this.y, enemy.x, enemy.y);
      if (dist < nearestDistance) {
        nearest = enemy;
        nearestDistance = dist;
      }
    }
    return nearest;
  }

  fireAtTarget(target, game) {
    if (!target || !target.alive) {
      return;
    }

    const directionX = target.x - this.x;
    const directionY = target.y - this.y;
    const length = Math.hypot(directionX, directionY) || 1;
    const dx = directionX / length;
    const dy = directionY / length;

    const baseProjectiles = [
      { x: this.x, y: this.y, dx, dy }
    ];

    if (this.multishotLevel > 0) {
      const extraCount = Math.min(this.multishotLevel, 2);
      for (let i = 0; i < extraCount; i++) {
        const angleOffset = (i + 1) * 0.2;
        const offsetX = dx * Math.cos(angleOffset) - dy * Math.sin(angleOffset);
        const offsetY = dx * Math.sin(angleOffset) + dy * Math.cos(angleOffset);
        baseProjectiles.push({ x: this.x, y: this.y, dx: offsetX, dy: offsetY });
      }
    }

    for (let i = 0; i < baseProjectiles.length; i++) {
      const projectile = new Projectile(
        baseProjectiles[i].x,
        baseProjectiles[i].y,
        this.x + baseProjectiles[i].dx * 10,
        this.y + baseProjectiles[i].dy * 10,
        this.attackDamage,
        '#61f0ff',
        this.projectileSpeed,
        this.projectileSize,
        this.pierceLevel
      );
      projectile.vx = baseProjectiles[i].dx * this.projectileSpeed;
      projectile.vy = baseProjectiles[i].dy * this.projectileSpeed;
      game.projectiles.push(projectile);
    }

    game.audioManager.shoot();
  }

  takeDamage(amount, game) {
    if (this.invincibilityTimer > 0) {
      return;
    }

    const damageReduction = this.forceFieldLevel > 0 ? 0.85 : 1;
    const finalDamage = amount * damageReduction;
    this.health -= finalDamage;
    this.invincibilityTimer = CONFIG.player.invincibility;
    game.createParticles(this.x, this.y, '#ff5c7d', 12, { speed: 160, gravity: 0 });
    game.addFloatingText(this.x, this.y, `-${Math.ceil(finalDamage)}`, '#ff5c7d');
    game.audioManager.playerDamaged();
    game.screenShake = 10;

    if (this.health <= 0) {
      this.health = 0;
      game.endGame();
    }
  }

  draw(ctx) {
    const blink = this.invincibilityTimer > 0 && Math.floor(this.invincibilityTimer * 12) % 2 === 0;
    if (blink) {
      return;
    }

    ctx.save();
    ctx.translate(this.x, this.y);

    const angle = Math.atan2(this.lastMoveY, this.lastMoveX);
    ctx.rotate(angle);

    ctx.fillStyle = '#61f0ff';
    ctx.shadowBlur = 18;
    ctx.shadowColor = '#61f0ff';
    ctx.beginPath();
    ctx.moveTo(this.radius + 10, 0);
    ctx.lineTo(-this.radius * 0.6, this.radius * 0.9);
    ctx.lineTo(-this.radius * 0.9, 0);
    ctx.lineTo(-this.radius * 0.6, -this.radius * 0.9);
    ctx.closePath();
    ctx.fill();

    ctx.fillStyle = '#d8faff';
    ctx.beginPath();
    ctx.arc(this.radius * 0.3, 0, 3.2, 0, Math.PI * 2);
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
    this.player = null;
    this.enemies = [];
    this.projectiles = [];
    this.gems = [];
    this.particles = [];
    this.floatingTexts = [];
    this.score = 0;
    this.defeatedEnemies = 0;
    this.elapsed = 0;
    this.remainingTime = CONFIG.roundDuration;
    this.spawnTimer = CONFIG.spawn.startDelay;
    this.difficultyLevel = 1;
    this.maxActiveEnemies = CONFIG.spawn.maxEnemies;
    this.input = new InputManager(this);
    this.audioManager = new AudioManager();
    this.ui = new UIManager(this);
    this.settings = { reducedMotion: false };
    this.pendingUpgrades = [];
    this.screenShake = 0;
    this.lastTime = 0;
    this.backgroundStars = Array.from({ length: 50 }, () => ({
      x: Math.random() * this.width,
      y: Math.random() * this.height,
      size: randomRange(1, 3),
      alpha: randomRange(0.3, 1),
      drift: randomRange(8, 28)
    }));
    this.resizeCanvas();
    this.ui.loadSettings();
    this.ui.syncMuteButtons();
    this.ui.renderHighScores(this.loadHighScores());
    this.ui.setOverlayState('menu', true);
    this.ui.showHUD(false);
    this.bindEvents();
    this.initialize();
  }

  bindEvents() {
    window.addEventListener('resize', () => this.resizeCanvas());
    document.addEventListener('visibilitychange', () => {
      if (document.hidden && this.state === 'PLAYING') {
        this.pauseGame();
      }
    });
  }

  resizeCanvas() {
    const rect = this.canvas.parentElement.getBoundingClientRect();
    const width = Math.max(320, rect.width);
    const height = Math.max(240, rect.height);
    const ratio = window.devicePixelRatio || 1;
    this.canvas.width = width * ratio;
    this.canvas.height = height * ratio;
    this.ctx.setTransform(ratio, 0, 0, ratio, 0, 0);
    this.width = width;
    this.height = height;
    this.canvas.style.width = `${width}px`;
    this.canvas.style.height = `${height}px`;
  }

  initialize() {
    this.player = new Player(this);
    this.player.reset(this);
    this.ui.updateHUD(this);
    this.render();
    requestAnimationFrame((time) => this.gameLoop(time));
  }

  gameLoop(timestamp) {
    const deltaSeconds = Math.min((timestamp - this.lastTime) / 1000 || 0.016, 0.05);
    this.lastTime = timestamp;

    if (this.state === 'PLAYING') {
      this.update(deltaSeconds);
    } else {
      this.updateBackground(deltaSeconds);
    }

    this.render();
    requestAnimationFrame((nextTs) => this.gameLoop(nextTs));
  }

  updateBackground(dt) {
    this.backgroundStars.forEach((star) => {
      star.y += star.drift * dt;
      if (star.y > this.height + 10) {
        star.y = -10;
        star.x = Math.random() * this.width;
      }
    });
  }

  startGame() {
    this.resetGame();
    this.state = 'PLAYING';
    this.ui.setOverlayState('menu', false);
    this.ui.setOverlayState('pause', false);
    this.ui.setOverlayState('gameOver', false);
    this.ui.setOverlayState('victory', false);
    this.ui.showHUD(true);
    this.ui.syncMuteButtons();
    this.audioManager.ensureContext();
    this.audioManager.buttonClick();
  }

  resetGame() {
    this.enemies = [];
    this.projectiles = [];
    this.gems = [];
    this.particles = [];
    this.floatingTexts = [];
    this.score = 0;
    this.defeatedEnemies = 0;
    this.elapsed = 0;
    this.remainingTime = CONFIG.roundDuration;
    this.spawnTimer = CONFIG.spawn.startDelay;
    this.difficultyLevel = 1;
    this.maxActiveEnemies = CONFIG.spawn.maxEnemies;
    this.screenShake = 0;
    this.player = new Player(this);
    this.player.reset(this);
    this.pendingUpgrades = [];
    this.ui.updateHUD(this);
  }

  update(dt) {
    this.elapsed += dt;
    this.remainingTime = Math.max(0, CONFIG.roundDuration - this.elapsed);
    this.ui.updateHUD(this);

    if (this.remainingTime <= 0) {
      this.winGame();
      return;
    }

    this.updateDifficulty();
    this.updateSpawning(dt);
    this.player.update(dt, this);

    this.updateProjectiles(dt);
    this.updateEnemies(dt);
    this.updateGems(dt);
    this.updateParticles(dt);
    this.updateFloatingTexts(dt);

    if (this.player.health <= 0) {
      this.endGame();
    }
  }

  updateDifficulty() {
    if (this.elapsed < 15) {
      this.difficultyLevel = 1;
      this.maxActiveEnemies = CONFIG.spawn.maxEnemies;
    } else if (this.elapsed < 30) {
      this.difficultyLevel = 2;
      this.maxActiveEnemies = CONFIG.spawn.maxEnemies + 2;
    } else if (this.elapsed < 45) {
      this.difficultyLevel = 3;
      this.maxActiveEnemies = CONFIG.spawn.maxEnemies + 4;
    } else {
      this.difficultyLevel = 4;
      this.maxActiveEnemies = CONFIG.spawn.maxEnemiesLate;
    }
  }

  updateSpawning(dt) {
    if (this.state !== 'PLAYING') {
      return;
    }

    this.spawnTimer -= dt;
    const spawnIntensity = clamp(1.35 - this.elapsed / 120, 0.45, 1.2);
    const spawnDelay = clamp(CONFIG.spawn.startDelay * spawnIntensity, CONFIG.spawn.minDelay, 2);
    const difficultyRamp = 1 + this.elapsed / 80;
    const targetDelay = Math.max(CONFIG.spawn.minDelay, spawnDelay / difficultyRamp);

    if (this.spawnTimer <= 0 && this.enemies.length < this.maxActiveEnemies) {
      this.spawnEnemy();
      this.spawnTimer = targetDelay;
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
    if (this.elapsed >= CONFIG.spawn.difficultEnemyThresholds.brute) {
      const roll = Math.random();
      if (roll < 0.25) type = 'brute'; else if (this.elapsed >= CONFIG.spawn.difficultEnemyThresholds.runner && roll < 0.65) type = 'runner';
    } else if (this.elapsed >= CONFIG.spawn.difficultEnemyThresholds.runner) {
      const roll = Math.random();
      if (roll < 0.45) type = 'runner';
    }

    const enemy = new Enemy(type, x, y);
    const minDistance = this.player.radius + enemy.radius + 120;
    if (distanceBetweenPoints(enemy.x, enemy.y, this.player.x, this.player.y) < minDistance) {
      enemy.x = this.width / 2 + randomRange(-120, 120);
      enemy.y = this.height / 2 + randomRange(-120, 120);
    }

    this.enemies.push(enemy);
    if (enemy.enemyType === 'Brute') {
      this.createParticles(enemy.x, enemy.y, '#ff9d4d', 16, { speed: 130, gravity: 0 });
      this.screenShake = 12;
    }
  }

  updateEnemies(dt) {
    for (let i = this.enemies.length - 1; i >= 0; i--) {
      const enemy = this.enemies[i];
      if (!enemy.alive) {
        this.enemies.splice(i, 1);
        continue;
      }

      enemy.update(dt, this);

      if (circlesOverlap(enemy, this.player)) {
        const damage = enemy.damage * (this.player.forceFieldLevel > 0 ? 0.85 : 1);
        this.player.takeDamage(damage, this);
        const push = normalizeVector(this.player.x - enemy.x, this.player.y - enemy.y);
        enemy.x += push.x * 8;
        enemy.y += push.y * 8;
        if (this.player.health <= 0) {
          return;
        }
      }
    }
  }

  updateProjectiles(dt) {
    for (let i = this.projectiles.length - 1; i >= 0; i--) {
      const projectile = this.projectiles[i];
      projectile.update(dt);

      if (projectile.lifetime <= 0 || projectile.x < -20 || projectile.x > this.width + 20 || projectile.y < -20 || projectile.y > this.height + 20) {
        this.projectiles.splice(i, 1);
        continue;
      }

      for (let j = this.enemies.length - 1; j >= 0; j--) {
        const enemy = this.enemies[j];
        if (!enemy.alive) {
          continue;
        }
        if (projectile.hitEnemies.has(enemy.id)) {
          continue;
        }
        if (distanceBetweenPoints(projectile.x, projectile.y, enemy.x, enemy.y) <= projectile.radius + enemy.radius) {
          enemy.health -= projectile.damage;
          enemy.hitFlashTimer = 0.15;
          this.audioManager.enemyHit();
          this.createParticles(projectile.x, projectile.y, '#61f0ff', 8, { speed: 90, gravity: 0 });
          this.addFloatingText(projectile.x, projectile.y, `-${Math.ceil(projectile.damage)}`, '#61f0ff');

          projectile.hitEnemies.add(enemy.id);
          if (this.player.pierceLevel > 0 && projectile.pierceLevel > 0) {
            projectile.pierceLevel--;
            if (projectile.pierceLevel <= 0) {
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
  }

  updateGems(dt) {
    for (let i = this.gems.length - 1; i >= 0; i--) {
      const gem = this.gems[i];
      gem.update(dt, this);

      if (distanceBetweenPoints(gem.x, gem.y, this.player.x, this.player.y) <= this.player.collectionRadius * 0.5 + gem.radius) {
        this.player.experience += gem.value;
        this.audioManager.collectGem();
        this.createParticles(gem.x, gem.y, '#7cff7d', 12, { speed: 75, gravity: 0 });
        this.gems.splice(i, 1);
        this.checkForLevelUp();
      }
    }
  }

  updateParticles(dt) {
    for (let i = this.particles.length - 1; i >= 0; i--) {
      const particle = this.particles[i];
      particle.update(dt);
      if (particle.lifetime <= 0 || this.particles.length > CONFIG.particles.maxParticles) {
        this.particles.splice(i, 1);
      }
    }
  }

  updateFloatingTexts(dt) {
    for (let i = this.floatingTexts.length - 1; i >= 0; i--) {
      const text = this.floatingTexts[i];
      text.y -= (text.speed || 26) * dt;
      text.alpha -= dt * 1.2;
      text.life -= dt;
      if (text.life <= 0 || text.alpha <= 0) {
        this.floatingTexts.splice(i, 1);
      }
    }
  }

  createParticles(x, y, color, count = 10, options = {}) {
    if (this.particles.length > CONFIG.particles.maxParticles) {
      return;
    }

    for (let i = 0; i < count; i++) {
      this.particles.push(new Particle(x, y, color, {
        size: options.size || randomRange(2, 5),
        lifetime: options.lifetime || randomRange(0.3, 0.9),
        gravity: options.gravity !== undefined ? options.gravity : 25,
        friction: options.friction || 0.95,
        fade: options.fade || 1.2,
        vxFactor: randomRange(0.6, 1.4) * (options.speed || 60) / 60,
        vyFactor: randomRange(0.6, 1.4) * (options.speed || 60) / 60
      }));
    }
  }

  addFloatingText(x, y, text, color, speed = 26) {
    if (this.floatingTexts.length > CONFIG.particles.maxFloatingTexts) {
      return;
    }
    this.floatingTexts.push({ x, y, text, color, alpha: 1, life: 1.1, speed });
  }

  defeatEnemy(enemy) {
    if (!enemy || !enemy.alive) {
      return;
    }
    enemy.alive = false;
    this.score += enemy.scoreValue;
    this.defeatedEnemies += 1;
    this.audioManager.enemyDefeated();
    this.createParticles(enemy.x, enemy.y, '#ffb347', 18, { speed: 150, gravity: 0 });
    this.addFloatingText(enemy.x, enemy.y - 16, `+${enemy.scoreValue}`, '#ffe66d', 28);

    const gemCount = Math.max(1, Math.floor(enemy.experienceValue / 10));
    for (let i = 0; i < gemCount; i++) {
      this.gems.push(new ExperienceGem(enemy.x + randomRange(-20, 20), enemy.y + randomRange(-20, 20), enemy.experienceValue));
    }

    const enemyIndex = this.enemies.indexOf(enemy);
    if (enemyIndex >= 0) {
      this.enemies.splice(enemyIndex, 1);
    }
  }

  checkForLevelUp() {
    while (this.player.experience >= this.player.experienceRequired) {
      this.player.experience -= this.player.experienceRequired;
      this.player.level += 1;
      this.player.experienceRequired = Math.floor(CONFIG.player.baseExperienceRequired * Math.pow(1.25, this.player.level - 1));
      this.showLevelUp();
      return;
    }
  }

  showLevelUp() {
    this.state = 'LEVEL_UP';
    const choices = this.getRandomUpgradeChoices(3);
    this.pendingUpgrades = choices;
    this.ui.renderUpgradeChoices(choices);
    this.ui.setOverlayState('upgrade', true);
    this.ui.showHUD(true);
    this.audioManager.levelUp();
  }

  getRandomUpgradeChoices(count) {
    const available = upgradeCatalog.filter((upgrade) => !this.pendingUpgrades.some((selected) => selected.id === upgrade.id));
    const shuffled = [...available].sort(() => Math.random() - 0.5);
    return shuffled.slice(0, count).map((upgrade) => new Upgrade(upgrade));
  }

  applyUpgrade(id) {
    const upgrade = this.pendingUpgrades.find((choice) => choice.id === id);
    if (!upgrade) {
      return;
    }
    upgrade.apply(this.player);
    this.pendingUpgrades = [];
    this.ui.setOverlayState('upgrade', false);
    this.state = 'PLAYING';
    this.audioManager.buttonClick();
  }

  pauseGame() {
    if (this.state === 'PLAYING') {
      this.state = 'PAUSED';
      this.ui.setOverlayState('pause', true);
      this.ui.setOverlayState('upgrade', false);
      this.ui.showHUD(true);
    }
  }

  resumeGame() {
    if (this.state === 'PAUSED') {
      this.state = 'PLAYING';
      this.ui.setOverlayState('pause', false);
      this.ui.showHUD(true);
    }
  }

  endGame() {
    if (this.state === 'GAME_OVER' || this.state === 'VICTORY') {
      return;
    }
    this.state = 'GAME_OVER';
    this.ui.setOverlayState('gameOver', true);
    this.ui.setOverlayState('pause', false);
    this.ui.showHUD(true);
    this.ui.setSummaryText(`
      <strong>Score:</strong> ${Math.floor(this.score)}<br>
      <strong>Time survived:</strong> ${(this.elapsed).toFixed(1)}s<br>
      <strong>Level reached:</strong> ${this.player.level}<br>
      <strong>Enemies defeated:</strong> ${this.defeatedEnemies}
    `, this.ui.gameOverSummary);
    this.saveHighScore();
    this.ui.renderHighScores(this.loadHighScores());
    this.audioManager.gameOver();
    this.createParticles(this.player.x, this.player.y, '#ff5c7d', 30, { speed: 180, gravity: 0 });
    this.screenShake = 20;
  }

  winGame() {
    if (this.state === 'VICTORY' || this.state === 'GAME_OVER') {
      return;
    }
    this.state = 'VICTORY';
    this.ui.setOverlayState('victory', true);
    this.ui.setOverlayState('pause', false);
    this.ui.showHUD(true);
    this.ui.setSummaryText(`
      <strong>Score:</strong> ${Math.floor(this.score)}<br>
      <strong>Time survived:</strong> ${(this.elapsed).toFixed(1)}s<br>
      <strong>Final level:</strong> ${this.player.level}<br>
      <strong>Enemies defeated:</strong> ${this.defeatedEnemies}
    `, this.ui.victorySummary);
    this.saveHighScore();
    this.ui.renderHighScores(this.loadHighScores());
    this.audioManager.victory();
    this.createParticles(this.player.x, this.player.y, '#7cff7d', 35, { speed: 200, gravity: 0 });
    this.screenShake = 25;
  }

  saveHighScore() {
    try {
      const raw = localStorage.getItem(STORAGE_KEYS.highScores);
      const existing = raw ? JSON.parse(raw) : [];
      const nextEntry = {
        score: Math.floor(this.score),
        survivalTime: Number(this.elapsed.toFixed(1)),
        level: this.player.level,
        enemyCount: this.defeatedEnemies,
        date: new Date().toISOString()
      };
      const data = Array.isArray(existing) ? existing : [];
      data.push(nextEntry);
      data.sort((a, b) => b.score - a.score || b.survivalTime - a.survivalTime);
      const topTen = data.slice(0, 10);
      localStorage.setItem(STORAGE_KEYS.highScores, JSON.stringify(topTen));
    } catch (error) {
      // ignore storage issues
    }
  }

  loadHighScores() {
    try {
      const raw = localStorage.getItem(STORAGE_KEYS.highScores);
      if (!raw) {
        return [];
      }
      const parsed = JSON.parse(raw);
      if (!Array.isArray(parsed)) {
        return [];
      }
      return parsed.filter((entry) => entry && Number.isFinite(entry.score));
    } catch (error) {
      return [];
    }
  }

  clearHighScores() {
    const confirmClear = window.confirm('Clear all high scores?');
    if (!confirmClear) {
      return;
    }
    try {
      localStorage.removeItem(STORAGE_KEYS.highScores);
    } catch (error) {
      // ignore storage issues
    }
    this.ui.renderHighScores([]);
  }

  showInstructions() {
    this.state = 'INSTRUCTIONS';
    this.ui.setOverlayState('menu', false);
    this.ui.setOverlayState('instructions', true);
    this.ui.showHUD(false);
  }

  showHighScores() {
    this.state = 'HIGH_SCORES';
    this.ui.renderHighScores(this.loadHighScores());
    this.ui.setOverlayState('menu', false);
    this.ui.setOverlayState('highScores', true);
    this.ui.showHUD(false);
  }

  showMenu() {
    this.state = 'MENU';
    this.ui.setOverlayState('instructions', false);
    this.ui.setOverlayState('highScores', false);
    this.ui.setOverlayState('pause', false);
    this.ui.setOverlayState('upgrade', false);
    this.ui.setOverlayState('gameOver', false);
    this.ui.setOverlayState('victory', false);
    this.ui.setOverlayState('menu', true);
    this.ui.showHUD(false);
  }

  restartGame() {
    this.audioManager.buttonClick();
    this.startGame();
  }

  returnToMenu() {
    this.audioManager.buttonClick();
    this.state = 'MENU';
    this.ui.setOverlayState('pause', false);
    this.ui.setOverlayState('upgrade', false);
    this.ui.setOverlayState('gameOver', false);
    this.ui.setOverlayState('victory', false);
    this.ui.setOverlayState('menu', true);
    this.ui.showHUD(false);
    this.resetGame();
  }

  render() {
    const ctx = this.ctx;
    ctx.clearRect(0, 0, this.width, this.height);

    this.drawBackground(ctx);

    if (this.state !== 'MENU' && this.state !== 'INSTRUCTIONS' && this.state !== 'HIGH_SCORES') {
      this.drawArena(ctx);
      for (const gem of this.gems) {
        gem.draw(ctx);
      }
      for (const projectile of this.projectiles) {
        projectile.draw(ctx);
      }
      for (const enemy of this.enemies) {
        enemy.draw(ctx);
      }
      if (this.player) {
        this.player.draw(ctx);
      }
      for (const particle of this.particles) {
        particle.draw(ctx);
      }
      for (const text of this.floatingTexts) {
        ctx.save();
        ctx.globalAlpha = text.alpha || 1;
        ctx.fillStyle = text.color;
        ctx.font = 'bold 18px Arial';
        ctx.textAlign = 'center';
        ctx.fillText(text.text, text.x, text.y);
        ctx.restore();
      }
    }

    if (DEBUG) {
      this.renderDebugOverlay();
    }
  }

  drawBackground(ctx) {
    const gradient = ctx.createLinearGradient(0, 0, this.width, this.height);
    gradient.addColorStop(0, '#08161f');
    gradient.addColorStop(1, '#01070d');
    ctx.fillStyle = gradient;
    ctx.fillRect(0, 0, this.width, this.height);

    ctx.save();
    ctx.strokeStyle = 'rgba(97, 240, 255, 0.08)';
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

  drawArena(ctx) {
    ctx.save();
    ctx.strokeStyle = 'rgba(97, 240, 255, 0.18)';
    ctx.lineWidth = 2;
    ctx.strokeRect(12, 12, this.width - 24, this.height - 24);
    ctx.restore();
  }

  renderDebugOverlay() {
    const ctx = this.ctx;
    ctx.save();
    ctx.fillStyle = 'rgba(0,0,0,0.45)';
    ctx.fillRect(12, 12, 260, 140);
    ctx.fillStyle = '#fff';
    ctx.font = '12px monospace';
    ctx.fillText(`FPS: ${Math.round(1 / Math.max(0.016, this.lastFrameMs || 0.016))}`, 20, 32);
    ctx.fillText(`Enemies: ${this.enemies.length}`, 20, 48);
    ctx.fillText(`Projectiles: ${this.projectiles.length}`, 20, 64);
    ctx.fillText(`Particles: ${this.particles.length}`, 20, 80);
    ctx.fillText(`Player: ${this.player.x.toFixed(1)}, ${this.player.y.toFixed(1)}`, 20, 96);
    ctx.fillText(`State: ${this.state}`, 20, 112);
    ctx.fillText(`Spawn: ${this.spawnTimer.toFixed(2)}s`, 20, 128);
    ctx.restore();
  }
}

window.addEventListener('load', () => {
  const game = new Game();
  window.game = game;
  const ui = game.ui;
  ui.renderHighScores(game.loadHighScores());
  ui.setOverlayState('menu', true);
  ui.showHUD(false);
  if (!game.audioManager.muted) {
    game.audioManager.ensureContext();
  }
});

if (DEBUG) {
  window.__debugGame = () => true;
}

