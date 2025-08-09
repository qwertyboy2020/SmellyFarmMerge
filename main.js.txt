/* main.js
   Idle Merge Farm — Phaser 3
   Features:
   - grid of tiles, plant crops (different types & levels)
   - growth stages with timers
   - drag-to-merge (same type & level => merged into next level)
   - harvest for coins (auto-harvest upgrade)
   - upgrades shop (faster growth, auto-harvester)
   - save/load (localStorage), export/import JSON
   - fallback graphics when asset files are missing
*/

// -------- CONFIG --------
const CONFIG = {
  GRID_SIZE: 4,
  TILE_SIZE: 96,
  TILE_SPACING: 16,
  AUTO_SAVE_INTERVAL: 10000, // ms
  SAVE_KEY: 'idleMergeFarm_v1',
  CROP_TYPES: [
    { id: 'carrot', baseValue: 1, color: 0xff8c42 },
    { id: 'corn', baseValue: 2, color: 0xffdf5a },
    { id: 'tomato', baseValue: 3, color: 0xff5a6b }
  ],
  INITIAL_GROWTH_SECONDS: 12
};

// -------- BOILERPLATE / PHASER SETUP --------
const config = {
  type: Phaser.AUTO,
  width: Math.max(window.innerWidth, 800),
  height: Math.max(window.innerHeight, 600),
  parent: 'game-container',
  backgroundColor: 0x88c070,
  scene: {
    preload: preload,
    create: create,
    update: update
  }
};

let game = new Phaser.Game(config);

// -------- GLOBAL STATE --------
let state = {
  coins: 0,
  grid: [], // array of tile objects {x,y,tileSprite,crop?}
  upgrades: {
    growthSpeedMultiplier: 1.0,
    autoHarvester: false
  }
};

// UI handles
let coinText, infoText, shopContainer, exportArea;

// preload: try to load assets; if not available, shapes will be drawn later
function preload() {
  // Background and tiles
  this.load.image('farm_bg', 'assets/farm_bg.png');
  this.load.image('tile', 'assets/tile.png');

  // Crops: multiple stages per type+level (we'll load at least one per type)
  for (let t of CONFIG.CROP_TYPES) {
    this.load.image(t.id + '_1', `assets/${t.id}_1.png`);
    this.load.image(t.id + '_2', `assets/${t.id}_2.png`);
    this.load.image(t.id + '_3', `assets/${t.id}_3.png`);
  }

  this.load.image('coin', 'assets/coin.png');
  // small ui icons (optional)
  this.load.image('btn_plus', 'assets/btn_plus.png');
}

// create: layout, grid, UI, events
function create() {
  const scene = this;

  // background: prefer image, else a tiled color rectangle
  if (this.textures.exists('farm_bg')) {
    let bg = this.add.image(config.width/2, config.height/2, 'farm_bg');
    bg.setDisplaySize(config.width, config.height);
  } else {
    // fallback: gradient rectangle
    let g = this.add.graphics();
    g.fillStyle(0x9dd977, 1);
    g.fillRect(0, 0, config.width, config.height);
  }

  // top coin text & controls container
  coinText = this.add.text(16, 70, 'Coins: 0', { fontSize: '26px', fill:'#ffffff', stroke:'#000', strokeThickness:3 });

  infoText = this.add.text(16, 100, 'Click a tile to plant. Drag crops onto each other to merge same level.', { fontSize: '14px', fill:'#fff' });

  // build grid
  const gridSize = CONFIG.GRID_SIZE;
  const tsize = CONFIG.TILE_SIZE;
  const spacing = CONFIG.TILE_SPACING;
  const totalWidth = gridSize * tsize + (gridSize-1) * spacing;
  const offsetX = (config.width - totalWidth) / 2 + tsize/2;
  const offsetY = 160;

  // container to hold tiles for input ordering
  state.grid = [];
  for (let row = 0; row < gridSize; row++) {
    for (let col = 0; col < gridSize; col++) {
      const x = offsetX + col * (tsize + spacing);
      const y = offsetY + row * (tsize + spacing);

      let tileSprite;
      if (this.textures.exists('tile')) {
        tileSprite = this.add.image(x, y, 'tile').setInteractive({ useHandCursor: true });
        tileSprite.setDisplaySize(tsize, tsize);
      } else {
        // fallback: draw tile
        tileSprite = this.add.rectangle(x,y, tsize, tsize, 0xcddeab).setStrokeStyle(2, 0x8aa85a).setInteractive();
      }

      tileSprite.setData('col', col);
      tileSprite.setData('row', row);
      tileSprite.setData('crop', null);
      tileSprite.on('pointerdown', () => onTileClicked(scene, tileSprite));
      state.grid.push({ sprite: tileSprite, x, y, crop: null });
    }
  }

  // drag group: crops will be sprites that are interactive & draggable
  this.input.setDraggable([]);

  // timer: each second tick progress growth of crops
  this.time.addEvent({
    delay: 1000,
    loop: true,
    callback: onTick,
    callbackScope: this
  });

  // coin sprite for fly-to-counter animation
  this.coinPool = this.add.group();

  // shop / side-panel UI (DOM overlay)
  createShopUI();

  // load saved game if present
  loadGame();

  // autosave interval
  setInterval(() => {
    saveGame();
  }, CONFIG.AUTO_SAVE_INTERVAL);

  // save on unload
  window.addEventListener('beforeunload', saveGame);
}

// update loop
function update(time, dt) {
  // nothing heavy in update; animations handled separately
}

// ---------------- GAME MECHANICS ----------------

function onTileClicked(scene, tileSprite) {
  // if tile is empty: open quick-plant menu (for simplicity pick a default type or cycle through types)
  const tile = state.grid.find(g => g.sprite === tileSprite);
  if (!tile) return;

  if (!tile.crop) {
    // show small menu: for simplicity, plant default carrot by clicking
    plantCropAt(scene, tile, CONFIG.CROP_TYPES[0].id);
  } else {
    // if mature and not auto-harvested: harvest
    if (tile.crop.stage >= tile.crop.maxStage) {
      collectCrop(scene, tile);
    } else {
      // maybe speed up growth using coins? For now show hint
      flashInfo(`Crop is growing (${Math.ceil(tile.crop.timeLeft)}s left)`);
    }
  }
}

function plantCropAt(scene, tile, typeId, level = 1) {
  const tcfg = CONFIG.CROP_TYPES.find(t => t.id === typeId);
  if (!tcfg) return;

  const baseGrow = CONFIG.INITIAL_GROWTH_SECONDS;
  const growthTime = baseGrow / state.upgrades.growthSpeedMultiplier;

  const crop = {
    id: typeId,
    level: level,
    stage: 1,
    maxStage: 3, // 3 stages: young, mid, mature
    timeLeft: growthTime, // seconds to next stage/mature
    growthTime: growthTime
  };
  // create sprite
  const cropSprite = makeCropSprite(scene, tile.x, tile.y, crop);
  // set interactive and draggable
  cropSprite.setInteractive({ useHandCursor: true });
  scene.input.setDraggable(cropSprite);
  cropSprite.on('dragstart', function(pointer) {
    this.setDepth(1000);
    this.scene.tweens.add({
      targets: this,
      scale: 1.05,
      duration: 80,
      ease: 'Power1'
    });
  });
  cropSprite.on('drag', function(pointer, dragX, dragY) {
    this.x = dragX; this.y = dragY;
  });
  cropSprite.on('dragend', function(pointer) {
    // drop logic: snap to tile under pointer, else snap back
    const dropTile = findTileAt(pointer.worldX, pointer.worldY);
    if (dropTile) {
      // if dropTile has crop -> attempt merge
      if (dropTile.crop) {
        attemptMerge(scene, cropSprite, tile, dropTile);
      } else {
        // move crop to that tile
        moveCropToTile(scene, cropSprite, tile, dropTile);
      }
    } else {
      // no tile under drop -> snap back
      snapCropToTile(cropSprite, tile);
    }
    this.setDepth(0);
    this.scene.tweens.add({
      targets: this,
      scale: 0.95,
      duration: 120,
      ease: 'Power1'
    });
  });

  // store in tile
  tile.crop = {
    data: crop,
    sprite: cropSprite
  };

  // small planting animation
  scene.tweens.add({
    targets: cropSprite,
    scale: 0.95,
    alpha: { from: 0, to: 1 },
    duration: 250
  });
}

function makeCropSprite(scene, x, y, crop) {
  // prefer stage-specific texture if available
  const texKey = crop.id + '_' + crop.stage;
  let sprite;
  if (scene.textures.exists(texKey)) {
    sprite = scene.add.image(x, y, texKey);
    sprite.setDisplaySize(CONFIG.TILE_SIZE * 0.85, CONFIG.TILE_SIZE * 0.85);
  } else {
    // draw fallback: colored circle with text
    const g = scene.add.graphics();
    const tcfg = CONFIG.CROP_TYPES.find(t => t.id === crop.id);
    g.fillStyle(tcfg.color, 1);
    g.fillRoundedRect(-CONFIG.TILE_SIZE*0.35, -CONFIG.TILE_SIZE*0.35, CONFIG.TILE_SIZE*0.7, CONFIG.TILE_SIZE*0.7, 8);
    // render to texture
    const key = `tex_${crop.id}_${crop.stage}`;
    const rt = scene.add.renderTexture(0,0, CONFIG.TILE_SIZE, CONFIG.TILE_SIZE).setOrigin(0.5);
    rt.draw(g, CONFIG.TILE_SIZE/2, CONFIG.TILE_SIZE/2);
    g.destroy();
    sprite = rt;
    sprite.x = x; sprite.y = y;
    sprite.setDisplaySize(CONFIG.TILE_SIZE * 0.85, CONFIG.TILE_SIZE * 0.85);
  }
  sprite.setAlpha(1);
  sprite.setScale(0.95);
  return sprite;
}

function onTick() {
  // iterate crops and reduce timeLeft
  for (let tile of state.grid) {
    if (tile.crop) {
      const crop = tile.crop.data;
      // if already at max stage no further time decrease unless you want re-growth after harvest
      if (crop.stage < crop.maxStage) {
        crop.timeLeft -= 1;
        if (crop.timeLeft <= 0) {
          // progress stage
          crop.stage += 1;
          // reset next stage timer or clamp
          if (crop.stage < crop.maxStage) {
            crop.growthTime = CONFIG.INITIAL_GROWTH_SECONDS / state.upgrades.growthSpeedMultiplier;
            crop.timeLeft = crop.growthTime;
          } else {
            crop.timeLeft = 0;
            // matured
            flashInfo(`A ${crop.id} is ready to harvest!`);
            // if auto-harvester enabled, auto collect
            if (state.upgrades.autoHarvester) {
              // harvest after a small delay
              setTimeout(() => {
                collectCrop(this, tile);
              }, 300);
            }
          }
          // update sprite to new stage texture (replace)
          updateCropSprite(tile);
        }
      }
    }
  }
  updateUI();
}

function updateCropSprite(tile) {
  const scene = game.scene.scenes[0];
  if (!tile.crop) return;
  const crop = tile.crop.data;
  const old = tile.crop.sprite;
  // create replacement sprite at same position
  const newSprite = makeCropSprite(scene, tile.x, tile.y, crop);
  newSprite.setInteractive({ useHandCursor: true });
  // copy drag handlers as earlier
  scene.input.setDraggable(newSprite);
  newSprite.on('dragstart', function(pointer) { this.setDepth(1000); this.scene.tweens.add({targets:this, scale:1.05, duration:80}); });
  newSprite.on('drag', function(pointer, dragX, dragY) { this.x = dragX; this.y = dragY; });
  newSprite.on('dragend', function(pointer) {
    const dropTile = findTileAt(pointer.worldX, pointer.worldY);
    if (dropTile) {
      if (dropTile.crop) {
        attemptMerge(scene, newSprite, tile, dropTile);
      } else {
        moveCropToTile(scene, newSprite, tile, dropTile);
      }
    } else {
      snapCropToTile(newSprite, tile);
    }
    this.setDepth(0);
    this.scene.tweens.add({targets: this, scale:0.95, duration:120});
  });

  // destroy old sprite
  try { old.destroy(); } catch(e){}
  tile.crop.sprite = newSprite;
}

function collectCrop(scene, tile) {
  if (!tile.crop) return;
  const crop = tile.crop.data;
  // compute value: base * 2^(level-1) * stage multiplier
  const tcfg = CONFIG.CROP_TYPES.find(t => t.id === crop.id);
  const value = Math.round(tcfg.baseValue * Math.pow(2, crop.level - 1) * crop.stage);
  state.coins += value;
  animateCoinToUI(scene, tile.x, tile.y, value);
  // remove sprite
  try { tile.crop.sprite.destroy(); } catch(e) {}
  tile.crop = null;
  updateUI();
}

function animateCoinToUI(scene, fromX, fromY, amount) {
  // coin floating then move to top-left
  const coinSprite = scene.add.image(fromX, fromY - 10, scene.textures.exists('coin') ? 'coin' : null);
  if (!scene.textures.exists('coin')) {
    // fallback: small yellow circle
    const g = scene.add.graphics();
    g.fillStyle(0xffe066, 1);
    g.fillCircle(12, 12, 12);
    const rt = scene.add.renderTexture(0,0,24,24);
    rt.draw(g);
    g.destroy();
    coinSprite.destroy();
    rt.x = fromX; rt.y = fromY - 10;
    scene.tweens.add({
      targets: rt,
      x: 16 + 40, y: 70, alpha:0.8,
      duration: 800,
      ease: 'Cubic.easeOut',
      onComplete: () => { rt.destroy(); updateUI(); }
    });
  } else {
    coinSprite.setDisplaySize(28,28);
    scene.tweens.add({
      targets: coinSprite,
      x: 16 + 60, y: 70,
      duration: 800,
      ease: 'Cubic.easeOut',
      onComplete: () => { coinSprite.destroy(); }
    });
  }
  // update coin text smoothly
  updateUI();
}

function findTileAt(worldX, worldY) {
  for (let tile of state.grid) {
    // simple bounding box
    const half = CONFIG.TILE_SIZE / 2;
    if (worldX >= tile.x - half && worldX <= tile.x + half && worldY >= tile.y - half && worldY <= tile.y + half) {
      return tile;
    }
  }
  return null;
}

function moveCropToTile(scene, sprite, fromTile, toTile) {
  // move sprite visually, update tiles
  fromTile.crop = null;
  toTile.crop = {
    data: sprite.cropData || (sprite._cropData ? sprite._cropData : null),
    sprite: sprite
  };
  // ensure sprite snaps to new tile coords
  scene.tweens.add({
    targets: sprite,
    x: toTile.x, y: toTile.y,
    duration: 180,
    ease: 'Power2'
  });
}

function snapCropToTile(sprite, tile) {
  sprite.scene.tweens.add({
    targets: sprite,
    x: tile.x, y: tile.y,
    duration: 180,
    ease: 'Power2'
  });
}

function attemptMerge(scene, dragSprite, fromTile, targetTile) {
  if (!fromTile.crop || !targetTile.crop) {
    snapCropToTile(dragSprite, fromTile);
    return;
  }
  const a = fromTile.crop.data;
  const b = targetTile.crop.data;
  // merge only if same id and same level
  if (a.id === b.id && a.level === b.level) {
    // compute merged level
    const newLevel = a.level + 1;
    // remove both sprites
    try { fromTile.crop.sprite.destroy(); } catch(e){}
    try { targetTile.crop.sprite.destroy(); } catch(e){}
    // create new crop at targetTile with higher level, reset stages
    const mergedCropData = {
      id: a.id,
      level: newLevel,
      stage: 1,
      maxStage: 3,
      growthTime: CONFIG.INITIAL_GROWTH_SECONDS / state.upgrades.growthSpeedMultiplier,
      timeLeft: CONFIG.INITIAL_GROWTH_SECONDS / state.upgrades.growthSpeedMultiplier
    };
    const sprite = makeCropSprite(scene, targetTile.x, targetTile.y, mergedCropData);
    sprite.setInteractive({ useHandCursor: true });
    scene.input.setDraggable(sprite);
    // assign drag handlers
    sprite.on('dragstart', function(){ this.setDepth(1000); });
    sprite.on('drag', function(_, dx, dy){ this.x = dx; this.y = dy; });
    sprite.on('dragend', function(pointer) {
      const dropTile = findTileAt(pointer.worldX, pointer.worldY);
      if (dropTile) {
        if (dropTile.crop) attemptMerge(scene, sprite, targetTile, dropTile);
        else moveCropToTile(scene, sprite, targetTile, dropTile);
      } else snapCropToTile(sprite, targetTile);
      this.setDepth(0);
    });

    // put into targetTile
    targetTile.crop = { data: mergedCropData, sprite: sprite };
    // clear from fromTile
    fromTile.crop = null;

    // small merge pop + coins reward
    state.coins += Math.round( (CONFIG.CROP_TYPES.find(t=>t.id===a.id).baseValue) * Math.pow(2, newLevel-1) * 0.5 );
    flashInfo('Merged into level ' + newLevel + '! Bonus coins awarded.');
    updateUI();
  } else {
    // cannot merge: snap back
    snapCropToTile(dragSprite, fromTile);
  }
}

// ---------- UPGRADES / SHOP UI ----------
function createShopUI() {
  // create DOM side-panel in-game as a simple overlay (HTML nodes)
  const panel = document.createElement('div');
  panel.id = 'side-panel';
  panel.innerHTML = `
    <h3>Shop & Save</h3>
    <div>Growth Speed: <span id="growth-mul">1.0x</span></div>
    <div style="margin-top:8px;">
      <button id="buy-speed" class="btn">Buy +25% speed (cost: 50)</button>
    </div>
    <div style="margin-top:8px;">
      <button id="buy-auto" class="btn">Buy Auto Harvester (cost: 150)</button>
    </div>
    <hr />
    <div><strong>Save / Load</strong></div>
    <div style="margin-top:6px;">
      <button id="manual-save" class="btn">Save Now</button>
      <button id="manual-load" class="btn">Load</button>
    </div>
    <div id="save-import" style="margin-top:8px;">
      <div>Export:</div>
      <textarea id="export-data" readonly style="width:100%;height:60px;"></textarea>
      <button id="do-export" class="btn" style="margin-top:6px;">Generate Export</button>
      <div style="margin-top:6px;">Import (paste JSON):</div>
      <textarea id="import-data" style="width:100%;height:60px;"></textarea>
      <button id="do-import" class="btn" style="margin-top:6px;">Import</button>
    </div>
  `;
  document.body.appendChild(panel);

  // attach handlers
  document.getElementById('buy-speed').onclick = () => {
    if (state.coins >= 50) {
      state.coins -= 50;
      state.upgrades.growthSpeedMultiplier *= 1.25;
      flashInfo('Growth speed increased!');
      updateUI();
      saveGame();
    } else {
      flashInfo('Not enough coins.');
    }
  };
  document.getElementById('buy-auto').onclick = () => {
    if (state.coins >= 150) {
      state.coins -= 150;
      state.upgrades.autoHarvester = true;
      flashInfo('Auto Harvester unlocked!');
      updateUI();
      saveGame();
    } else {
      flashInfo('Not enough coins.');
    }
  };

  document.getElementById('manual-save').onclick = () => { saveGame(); flashInfo('Saved.'); };
  document.getElementById('manual-load').onclick = () => { loadGame(true); flashInfo('Loaded.'); };

  document.getElementById('do-export').onclick = () => {
    const j = exportSave();
    document.getElementById('export-data').value = j;
    flashInfo('Export generated — copy the text to save externally.');
  };
  document.getElementById('do-import').onclick = () => {
    const raw = document.getElementById('import-data').value;
    if (!raw) { flashInfo('Paste JSON into import box.'); return; }
    try {
      importSave(raw);
      flashInfo('Import successful.');
    } catch (e) {
      flashInfo('Import failed: invalid JSON.');
    }
  };
}

function updateUI() {
  coinText.setText('Coins: ' + Math.floor(state.coins));
  // update shop display values
  const mulEl = document.getElementById('growth-mul');
  if (mulEl) mulEl.innerText = state.upgrades.growthSpeedMultiplier.toFixed(2) + 'x';
}

// small temporary notification text
let infoTimer = null;
function flashInfo(msg) {
  const scene = game.scene.scenes[0];
  if (!scene) return;
  if (infoTimer) { clearTimeout(infoTimer); infoTimer = null; }
  infoText.setText(msg);
  infoTimer = setTimeout(() => {
    infoText.setText('Click a tile to plant. Drag crops onto each other to merge same level.');
    infoTimer = null;
  }, 2500);
}

// ---------------- SAVE / LOAD / EXPORT / IMPORT ----------------

function saveGame() {
  try {
    const data = {
      coins: state.coins,
      upgrades: state.upgrades,
      grid: state.grid.map(t => {
        if (!t.crop) return null;
        return {
          id: t.crop.data.id,
          level: t.crop.data.level,
          stage: t.crop.data.stage,
          timeLeft: t.crop.data.timeLeft
        };
      })
    };
    localStorage.setItem(CONFIG.SAVE_KEY, JSON.stringify(data));
    //console.log('Game saved.');
  } catch (e) {
    console.warn('Save failed', e);
  }
}

function loadGame(showFlash=false) {
  try {
    const raw = localStorage.getItem(CONFIG.SAVE_KEY);
    if (!raw) {
      if (showFlash) flashInfo('No save found.');
      return;
    }
    const data = JSON.parse(raw);
    // apply coins & upgrades
    state.coins = data.coins || 0;
    state.upgrades = data.upgrades || state.upgrades;

    // clear current crops
    for (let tile of state.grid) {
      if (tile.crop) { try { tile.crop.sprite.destroy(); } catch(e){} }
      tile.crop = null;
    }
    // restore grid crops
    for (let i = 0; i < state.grid.length; i++) {
      const tile = state.grid[i];
      const c = (data.grid && data.grid[i]) ? data.grid[i] : null;
      if (c) {
        const cropData = {
          id: c.id,
          level: c.level,
          stage: c.stage,
          maxStage: 3,
          growthTime: CONFIG.INITIAL_GROWTH_SECONDS / (state.upgrades.growthSpeedMultiplier || 1),
          timeLeft: c.timeLeft
        };
        const sprite = makeCropSprite(game.scene.scenes[0], tile.x, tile.y, cropData);
        sprite.setInteractive({ useHandCursor: true });
        game.scene.scenes[0].input.setDraggable(sprite);
        // attach drag handlers (same as earlier)
        sprite.on('dragstart', function(){ this.setDepth(1000); });
        sprite.on('drag', function(_,dx,dy){ this.x = dx; this.y = dy; });
        sprite.on('dragend', function(pointer) {
          const dropTile = findTileAt(pointer.worldX, pointer.worldY);
          if (dropTile) {
            if (dropTile.crop) attemptMerge(game.scene.scenes[0], sprite, tile, dropTile);
            else moveCropToTile(game.scene.scenes[0], sprite, tile, dropTile);
          } else snapCropToTile(sprite, tile);
          this.setDepth(0);
        });
        tile.crop = { data: cropData, sprite: sprite };
      }
    }
    updateUI();
    if (showFlash) flashInfo('Save loaded.');
  } catch (e) {
    console.warn('Load failed', e);
    if (showFlash) flashInfo('Load failed.');
  }
}

function exportSave() {
  try {
    return localStorage.getItem(CONFIG.SAVE_KEY) || '{}';
  } catch (e) { return '{}'; }
}

function importSave(raw) {
  // set localStorage and reload
  localStorage.setItem(CONFIG.SAVE_KEY, raw);
  loadGame(true);
}

// ------------- END -------------
