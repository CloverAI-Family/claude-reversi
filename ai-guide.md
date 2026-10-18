# 黑白棋 AI 操作指南

如何讓你的 Claude Code AI 和你對弈黑白棋（翻轉棋）。

## 設定

1. 將 `artifact.html` 以啟用 `db` 功能的方式發布為 claude.ai Artifact
2. 開啟 Artifact，選擇你的顏色——AI 扮演另一方
3. 將 Artifact URL 分享給你的 Claude Code session

## 操作流程

當玩家說「下好了」時，在 Claude Code 執行以下步驟：

### 步驟 1 — 讀取棋盤

```
ArtifactData get
url: <你的 artifact url>
collection: reversi
doc_id: state
```

### 步驟 2 — 解析棋盤

```javascript
// board 是 8 行以 '|' 分隔的字串，每行 8 個字元
// '0' = 空格，'1' = 黑子，'2' = 白子
const rows = d.board.split('|');
const board = rows.map(r => r.split('').map(Number));
// board[r][c]，0-indexed

const playerColor = d.player_color;  // 玩家顏色
const aiColor = 3 - playerColor;
const turn = d.turn;

// 若 turn !== aiColor，尚未輪到 AI——等待
```

### 步驟 3 — 計算合法落子

```javascript
const DIRS = [[-1,-1],[-1,0],[-1,1],[0,-1],[0,1],[1,-1],[1,0],[1,1]];

function getFlips(board, r, c, color) {
  if (board[r][c] !== 0) return [];
  const opp = 3 - color;
  const result = [];
  for (const [dr, dc] of DIRS) {
    const line = [];
    let nr = r + dr, nc = c + dc;
    while (nr >= 0 && nr < 8 && nc >= 0 && nc < 8 && board[nr][nc] === opp) {
      line.push([nr, nc]);
      nr += dr; nc += dc;
    }
    if (line.length > 0 && nr >= 0 && nr < 8 && nc >= 0 && nc < 8 && board[nr][nc] === color) {
      for (const pos of line) result.push(pos);
    }
  }
  return result;
}

function getLegalMoves(board, color) {
  const moves = [];
  for (let r = 0; r < 8; r++)
    for (let c = 0; c < 8; c++)
      if (getFlips(board, r, c, color).length > 0)
        moves.push([r, c]);
  return moves;
}

const legalMoves = getLegalMoves(board, aiColor);
```

### 步驟 4 — 選擇落子位置

思考棋局：
- 角落 `(0,0) (0,7) (7,0) (7,7)` 最優先——永久穩固，優先搶佔
- 避免落在角落斜鄰格（X 格）——會把角落送給對方
- 盡量最大化翻轉數，同時保護已有的穩固邊線
- 限制對方的可選落點（mobility）

若 `legalMoves.length === 0`：跳過，進入**步驟 6（跳過回合）**。

### 步驟 5 — 寫回棋步

先取得時間戳：
```bash
date +%s%3N
```

計算翻轉並序列化棋盤：
```javascript
const [r, c] = legalMoves[chosen_index];
const flips = getFlips(board, r, c, aiColor);

board[r][c] = aiColor;
for (const [fr, fc] of flips) board[fr][fc] = aiColor;

const newBoard = board.map(row => row.join('')).join('|');
```

寫回：
```
ArtifactData set
url: <你的 artifact url>
collection: reversi
doc_id: state
if_version: <步驟 1 取得的 version>
data: {
  board: <newBoard>,
  turn: <playerColor>,
  player_color: <playerColor>,
  seq: <date +%s%3N 的值>,
  last_r: <r>,
  last_c: <c>
}
```

### 步驟 6 — 跳過回合

若 AI 沒有合法落子：

```
ArtifactData set
...
data: {
  board: <不變的棋盤字串>,
  turn: <playerColor>,
  player_color: <playerColor>,
  seq: <時間戳>,
  last_r: -1,
  last_c: -1
}
```

UI 會自行處理雙方都無法落子的情況（遊戲結束）。

### 步驟 7 — 告知玩家

回覆：「下好了」

棋盤自動更新。

## 資料庫欄位

| 欄位 | 說明 |
|------|------|
| `board` | 8 行以 `\|` 分隔，每行 8 字元（`0`/`1`/`2`） |
| `turn` | 輪到誰：`1` = 黑，`2` = 白 |
| `player_color` | 玩家顏色：`1` = 黑，`2` = 白 |
| `seq` | 單調遞增時間戳——用 `date +%s%3N`，必須 ≥ 前一個值 |
| `last_r` / `last_c` | 最後落子位置；`-1` = 跳過 |

## 注意事項

- collection 是 `reversi`（不是 `game`），doc_id 是 `state`
- 不要寫入 `gameOver` 或 `winner`——UI 自行從棋盤計算
- 角落永不可翻，是最高價值位置
- 每次寫入前用 `date +%s%3N` 取得正確毫秒時間戳

---

<details>
<summary>English</summary>

# Reversi — AI Player Guide

How to play Reversi (Othello) against a human using Claude Code.

## Setup

1. Publish `artifact.html` as a claude.ai Artifact with the `db` capability
2. Open it and choose your color — the AI plays the other color
3. Share the Artifact URL with your Claude Code session

## Playing as the AI

When the human says they're done ("下好了" or similar), run this in your Claude Code session:

### Step 1 — Read the board

```
ArtifactData get
url: <your artifact url>
collection: reversi
doc_id: state
```

### Step 2 — Parse the board

```javascript
// board is 8 rows joined by '|', each row is 8 chars
// '0' = empty, '1' = black, '2' = white
const rows = d.board.split('|');
const board = rows.map(r => r.split('').map(Number));
// board[r][c] — 0-indexed

const playerColor = d.player_color;  // human's color
const aiColor = 3 - playerColor;
const turn = d.turn;

// If turn !== aiColor, it's not your turn yet — wait
```

### Step 3 — Find legal moves

```javascript
const DIRS = [[-1,-1],[-1,0],[-1,1],[0,-1],[0,1],[1,-1],[1,0],[1,1]];

function getFlips(board, r, c, color) {
  if (board[r][c] !== 0) return [];
  const opp = 3 - color;
  const result = [];
  for (const [dr, dc] of DIRS) {
    const line = [];
    let nr = r + dr, nc = c + dc;
    while (nr >= 0 && nr < 8 && nc >= 0 && nc < 8 && board[nr][nc] === opp) {
      line.push([nr, nc]);
      nr += dr; nc += dc;
    }
    if (line.length > 0 && nr >= 0 && nr < 8 && nc >= 0 && nc < 8 && board[nr][nc] === color) {
      for (const pos of line) result.push(pos);
    }
  }
  return result;
}

function getLegalMoves(board, color) {
  const moves = [];
  for (let r = 0; r < 8; r++)
    for (let c = 0; c < 8; c++)
      if (getFlips(board, r, c, color).length > 0)
        moves.push([r, c]);
  return moves;
}

const legalMoves = getLegalMoves(board, aiColor);
```

### Step 4 — Choose your move

Think through the position:
- Corner squares `(0,0) (0,7) (7,0) (7,7)` are extremely strong — grab them when available
- Avoid placing adjacent to empty corners (X-squares / C-squares) — they gift corners to the opponent
- Maximize your stable discs (corners and edges anchored to corners)
- If no good move is available, minimize the opponent's mobility

If `legalMoves.length === 0`: you have no legal move — go to **Step 6 (skip)**.

### Step 5 — Apply the move and write back

```javascript
const [r, c] = legalMoves[chosen_index];
const flips = getFlips(board, r, c, aiColor);

board[r][c] = aiColor;
for (const [fr, fc] of flips) board[fr][fc] = aiColor;

const newBoard = board.map(row => row.join('')).join('|');
```

Get the current timestamp first:
```bash
date +%s%3N
```

Then write back:
```
ArtifactData set
url: <your artifact url>
collection: reversi
doc_id: state
if_version: <version from step 1>
data: {
  board: <newBoard>,
  turn: <playerColor>,
  player_color: <playerColor>,
  seq: <timestamp from date +%s%3N>,
  last_r: <r>,
  last_c: <c>
}
```

### Step 6 — Skipping (no legal move)

If you have no legal move, skip your turn:

```
ArtifactData set
...
data: {
  board: <unchanged board string>,
  turn: <playerColor>,
  player_color: <playerColor>,
  seq: <timestamp>,
  last_r: -1,
  last_c: -1
}
```

The UI handles the case where both sides have no legal moves — it will show game over.

### Step 7 — Tell the human

Reply: "下好了"

The board updates automatically.

## Database Fields

| Field | Description |
|-------|-------------|
| `board` | 8 rows joined by `\|`, each row 8 chars (`0`/`1`/`2`) |
| `turn` | Whose turn: `1` = black, `2` = white |
| `player_color` | Human's color: `1` = black, `2` = white |
| `seq` | Monotone timestamp — use `date +%s%3N`, must be ≥ previous value |
| `last_r` / `last_c` | Last move position; `-1` = skip/none |

## Notes

- Collection is `reversi` (not `game`), doc_id is `state`
- Do NOT write `gameOver` or `winner` — the UI calculates them itself
- Corners are high-value; X-squares (diagonally adjacent to corners) are high-risk
- Use `date +%s%3N` before every write to get the correct millisecond timestamp

</details>
