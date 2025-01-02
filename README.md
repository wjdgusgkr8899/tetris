const canvas = document.getElementById('tetris');
const context = canvas.getContext('2d');
const storedCanvas = document.getElementById('stored');
const storedContext = storedCanvas.getContext('2d');
let blocktime = null;
const blockdelay = 500;

storedCanvas.width = 100;
storedCanvas.height = 100;

storedContext.scale(20, 20);
context.scale(20, 20);

function drawStoredPiece() {
  storedContext.clearRect(-2, -2, storedCanvas.width, storedCanvas.height);
  if (storedPiece) {
    const offsetX = Math.floor((storedCanvas.width / 20 - storedPiece[0].length) / 2);
    const offsetY = Math.floor((storedCanvas.height / 20 - storedPiece.length) / 2);
    drawMatrix(storedPiece, { x: offsetX, y: offsetY }, storedContext);
  }
}

const colors = [
  null,
  '#FF0D72',
  '#0DC2FF',
  '#0DFF72',
  '#F538FF',
  '#FF8E0D',
  '#FFE138',
  '#3877FF',
];

const arena = createMatrix(15, 30);

const player = {
  pos: { x: 0, y: 0 },
  matrix: null,
  score: 0,
};

const ghost = {
  pos: { x: 0, y: 0 },
  matrix: null,
};

let storedPiece = null;
let canStore = true;
const gameState = {
  moveInterval: null,
  softDropInterval: null,
  keyHeld: {},
  keyPressTimeout: {},
};



function clearLines(linesToClear) {
  const duration = 300; // 애니메이션 지속 시간
  const startTime = performance.now();

  function animateClear(currentTime) {
      const elapsed = currentTime - startTime;
      const progress = Math.min(1, elapsed / duration);
      const clearWidth = Math.floor(arena[0].length * progress);

      // 지워지는 줄의 해당 부분만 다시 그리기
      linesToClear.forEach(y => {
          context.fillStyle = '#000'; // 배경색으로 채우기
          context.fillRect(0, y, clearWidth, 1); // 지워지는 부분만 다시 그리기
      });

      drawMatrix(arena, { x: 0, y: 0 }); // 변경된 arena를 다시 그리기

      if (progress < 1) {
          requestAnimationFrame(animateClear);
      } else {
          linesToClear.sort((a, b) => b - a);
          linesToClear.forEach(y => {
              arena.splice(y, 1);
              arena.unshift(new Array(arena[0].length).fill(0));
          });

          const scoreMultiplier = [0, 40, 100, 300, 1200];
          player.score += scoreMultiplier[linesToClear.length];
          updateScore();
      }
  }

  requestAnimationFrame(animateClear);
}

function arenaSweep() {
  let linesCleared=0;
  let rowCount = 1;
  outer: for (let y = arena.length - 1; y > 0; --y) {
    for (let x = 0; x < arena[y].length; ++x) {
      if (arena[y][x] === 0) {
        continue outer;
      }
    }
    
    
    const row = arena.splice(y,1)[0].fill(0);
    arena.unshift(row);
    linesCleared++;
    y++;
  }
  if(linesCleared>0){
    player.score += linesCleared ===1 ? 10:linesCleared*10;
  }
}


function collide(arena, player) {
  const [m, o] = [player.matrix, player.pos];
  for (let y = 0; y < m.length; ++y) {
    for (let x = 0; x < m[y].length; ++x) {
      if (
        m[y][x] !== 0 &&
        (arena[y + o.y] === undefined ||
          arena[y + o.y][x + o.x] === undefined ||
          arena[y + o.y][x + o.x] !== 0)
      ) {
        return true;
      }
    }
  }
  return false;
}

function createMatrix(w, h) {
  const matrix = [];
  while (h--) {
    matrix.push(Array(w).fill(0));
  }
  return matrix;
}

function createPiece(type) {
  const pieces = {
    'T': [
      [0, 0, 0],
      [1, 1, 1],
      [0, 1, 0],
    ],
    'O': [
      [2, 2],
      [2, 2],
    ],
    'L': [
      [0, 3, 0],
      [0, 3, 0],
      [0, 3, 3],
    ],
    'J': [
      [0, 4, 0],
      [0, 4, 0],
      [4, 4, 0],
    ],
    'I': [
      [0, 5, 0, 0],
      [0, 5, 0, 0],
      [0, 5, 0, 0],
      [0, 5, 0, 0],
    ],
    'S': [
      [0, 6, 6],
      [6, 6, 0],
      [0, 0, 0],
    ],
    'Z': [
      [7, 7, 0],
      [0, 7, 7],
      [0, 0, 0],
    ],
  };
  return pieces[type];
}

function updateGhostPosition() {
  ghost.pos.x = player.pos.x;
  ghost.pos.y = player.pos.y;
  while (!collide(arena, ghost)) {
    ghost.pos.y++;
  }
  ghost.pos.y--;
}

function drawGhost() {
  context.globalAlpha = 0.4;
  drawMatrix(ghost.matrix, ghost.pos);
  context.globalAlpha = 1;
}

function drawMatrix(matrix, offset, ctx = context) {
  matrix.forEach((row, y) => {
    row.forEach((value, x) => {
      if (value !== 0) {
        ctx.fillStyle = colors[value];
        ctx.fillRect(x + offset.x, y + offset.y, 1, 1);
        ctx.strokeStyle = '#000';
        ctx.lineWidth = 0.05;
        ctx.strokeRect(x + offset.x, y + offset.y, 1, 1);
      }
    });
  });
}

function draw() {
  context.fillStyle = '#000';
  context.fillRect(0, 0, canvas.width, canvas.height);

  drawMatrix(arena, { x: 0, y: 0 });
  updateGhostPosition();
  drawGhost();
  drawMatrix(player.matrix, player.pos);

  drawStoredPiece();
}

function merge(arena, player) {
  player.matrix.forEach((row, y) => {
    row.forEach((value, x) => {
      if (value !== 0) {
        if (y + player.pos.y >= 0 && y + player.pos.y < arena.length &&
            x + player.pos.x >= 0 && x + player.pos.x < arena[0].length) {
          arena[y + player.pos.y][x + player.pos.x] = Number(value);
        }
      }
    });
  });
  arenaSweep();
  playerReset();
}

function rotate(matrix, dir) {
  for (let y = 0; y < matrix.length; ++y) {
    for (let x = 0; x < y; ++x) {
      [matrix[x][y], matrix[y][x]] = [matrix[y][x], matrix[x][y]];
    }
  }
  if (dir > 0) {
    matrix.forEach(row => row.reverse());
  } else {
    matrix.reverse();
  }
}

function resetBlockTime() {
  if (blocktime) {
    clearTimeout(blocktime);
    blocktime = null;
  }
}

function playerSoftDrop() {
  player.pos.y++;
  if (collide(arena, player)) {
    player.pos.y--;
    if (!blocktime) {
      blocktime = setTimeout(() => {
        merge(arena, player);
        blocktime = null;
      }, 1000);
    }
  } else {
    resetBlockTime();
  }
  dropCounter = 0;
}

function freeDrop() {
  while (!collide(arena, player)) {
    player.pos.y++;
  }
  player.pos.y--;
  merge(arena, player);
  dropCounter = 0;
}

function playerMove(offset) {
  player.pos.x += offset;
  if (collide(arena, player)) {
    player.pos.x -= offset;
  }
  updateGhostPosition();
  resetBlockTime();
}

function playerReset() {
  const pieces = 'ILJOTSZ';
  player.matrix = createPiece(pieces[pieces.length * Math.random() | 0]);
  player.pos.y = 0;
  player.pos.x = (arena[0].length / 2 | 0) - (player.matrix[0].length / 2 | 0);
  ghost.matrix = player.matrix;
  updateGhostPosition();
  canStore = true;

  if (collide(arena, player)) {
    arena.forEach(row => row.fill(0));
    player.score = 0;
    storedPiece = null;
    drawStoredPiece();
    updateScore();
  }
}

function storePiece() {
  if (!canStore) return;

  if (storedPiece) {
    let temp = player.matrix;
    player.matrix = storedPiece;
    storedPiece = temp;
    player.pos.y = 0;
    player.pos.x = (arena[0].length / 2 | 0) - (player.matrix[0].length / 2 | 0);
  } else {
    storedPiece = player.matrix;
    playerReset();
  }
  ghost.matrix = player.matrix;
  updateGhostPosition();
  canStore = false;
}

function playerRotate(dir) {
  const pos = player.pos.x;
  let offset = 1;
  rotate(player.matrix, dir);
  while (collide(arena, player)) {
    player.pos.x += offset;
    offset = -(offset + (offset > 0 ? 1 : -1));
    if (offset > player.matrix[0].length) {
      rotate(player.matrix, -dir);
      player.pos.x = pos;
      return;
    }
  }
  updateGhostPosition();
  resetBlockTime();
}

let dropCounter = 0;
let dropInterval = 1000;

function fast() {
  dropInterval = Math.max(100, 1000 - Math.min(player.score, 90) * 10);
}

let lastTime = 0;

function update(time = 0) {
  const deltaTime = time - lastTime;
  lastTime = time;

  dropCounter += deltaTime;
  if (dropCounter > dropInterval) {
    playerSoftDrop();
    dropCounter = 0;
  }

  draw();
  updateScore();
  requestAnimationFrame(update);
}

function updateScore() {
  document.getElementById('score').innerText = player.score;
  fast(); // 속도 조절 함수
}


document.addEventListener('keydown', event => {
  const { keyHeld, keyPressTimeout, moveInterval, softDropInterval } = gameState;

  if (!keyHeld[event.key]) {
    keyHeld[event.key] = true;

    if (event.key === 'ArrowLeft') { // 왼쪽
      playerMove(-1);
      keyPressTimeout['ArrowLeft'] = setTimeout(() => {
        gameState.moveInterval = setInterval(() => playerMove(-1), 50); // 반복 이동
      }, 200);
    } else if (event.key === 'ArrowRight') { // 오른쪽
      playerMove(1);
      keyPressTimeout['ArrowRight'] = setTimeout(() => {
        gameState.moveInterval = setInterval(() => playerMove(1), 50);
      }, 200);
    } else if (event.key === 'ArrowDown') { // 아래
      playerSoftDrop();
      keyPressTimeout['ArrowDown'] = setTimeout(() => {
        gameState.softDropInterval = setInterval(playerSoftDrop, 50); // 빠르게 낙하
      }, 300);
    } else if (event.key === 'z') { // 회전 왼쪽
      playerRotate(-1);
    } else if (event.key === 'x') { // 회전 오른쪽
      playerRotate(1);
    } else if (event.key === ' ') { // 즉시 낙하
      freeDrop();
    } else if (event.key === 'c') { // 블록 저장
      storePiece();
    }
  }
});

document.addEventListener('keyup', event => {
  const { keyHeld, keyPressTimeout, moveInterval, softDropInterval } = gameState;

  keyHeld[event.key] = false;
  if (event.key === 'ArrowLeft' || event.key === 'ArrowRight') {
    clearInterval(moveInterval);
    clearTimeout(keyPressTimeout[event.key]);
    gameState.moveInterval = null;
  } else if (event.key === 'ArrowDown') {
    clearInterval(softDropInterval);
    clearTimeout(keyPressTimeout[event.key]);
    gameState.softDropInterval = null;
  }
});


playerReset();
updateScore();
update();

  
