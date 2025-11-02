<template>
  <div class="crossword-wrapper">
    <div class="crossword-container" ref="containerRef">
      <!-- Background image (fills container) -->
      <img src="/cross-word.jpg" class="background-image" alt="crossword bg" />

      <!-- Table overlay (fills container exactly) -->
      <table class="crossword-table" @mousedown.prevent>
        <tbody>
          <tr v-for="(row, r) in grid" :key="r">
            <td
              v-for="(cell, c) in row"
              :key="c"
              :class="cellClass(cell)"
              @click="onCellClick(cell)"
            >
              <template v-if="cell && !cell.blocked">
                <span v-if="cell.number" class="cell-number">{{
                  cell.number
                }}</span>
                <input
                  class="cross-input"
                  maxlength="1"
                  v-model="cell.value"
                  :data-r="cell.r"
                  :data-c="cell.c"
                  @focus="onFocus(cell)"
                  @input="onInput(cell)"
                  @keydown="onKeyDown($event, cell)"
                  autocomplete="off"
                />
              </template>
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <!--    <aside class="clue-panel">
      <h2>Clues</h2>

      <div class="clue-section">
        <h3>Across</h3>
        <ul>
          <li
            v-for="clue in clues.across"
            :key="clue.number"
            :class="{ selected: selectedClue === clue }"
            @click="selectClue(clue, 'across')"
          >
            {{ clue.number }}. {{ clue.text }}
          </li>
        </ul>
      </div>

      <div class="clue-section">
        <h3>Down</h3>
        <ul>
          <li
            v-for="clue in clues.down"
            :key="clue.number"
            :class="{ selected: selectedClue === clue }"
            @click="selectClue(clue, 'down')"
          >
            {{ clue.number }}. {{ clue.text }}
          </li>
        </ul>
      </div>

      <div class="buttons">
        <button @click="checkAnswers">Check</button>
        <button @click="resetPuzzle">Reset</button>
      </div>
    </aside> -->
    <div class="buttons">
      <button @click="checkAnswers">Check</button>
      <button @click="resetPuzzle">Reset</button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, nextTick } from "vue";

type Cell = {
  r: number;
  c: number;
  correct: string;
  value: string;
  blocked: boolean;
  number?: number;
  acrossId?: number;
  downId?: number;
};

const ROWS = 12;
const COLS = 16;

/* initialize grid */
const grid = ref<Cell[][]>(
  Array.from({ length: ROWS }, (_, r) =>
    Array.from({ length: COLS }, (_, c) => ({
      r,
      c,
      correct: "",
      value: "",
      blocked: true,
    }))
  )
);

/* helper to place word with row/col coordinates */
function placeWord(
  word: string,
  row: number,
  col: number,
  dir: "across" | "down",
  number?: number,
  id?: number
) {
  // ensure indices within bounds
  const len = word.length;
  for (let i = 0; i < len; i++) {
    const r = dir === "across" ? row : row + i;
    const c = dir === "across" ? col + i : col;
    if (r < 0 || r >= ROWS || c < 0 || c >= COLS) {
      console.warn("placeWord out of bounds:", word, row, col, dir);
      return;
    }
    const cell = grid.value[r][c];
    cell.blocked = false;
    cell.correct = word[i].toUpperCase();
    cell.value = "";
    if (i === 0 && number) cell.number = number;
    if (dir === "across") cell.acrossId = id;
    if (dir === "down") cell.downId = id;
  }
}

/* --- place our puzzle words (adjust positions so they fit) --- */
/* chosen positions keep everything inside 12x12 */
placeWord("PLACEBO", 1, 4, "across", 1, 1); // 1 Across
placeWord("CONTROLLED", 1, 7, "down", 2, 2); // 2 Down
placeWord("RANDOMISATION", 6, 3, "across", 5, 5); // 5 Across (moved a bit for better fit)
placeWord("EVIDENCE", 9, 0, "across", 7, 7); // 7 Across
placeWord("BLINDING", 3, 5, "down", 3, 3); // 3 Down (starts under C column)
placeWord("RCT", 4, 12, "down", 4, 4); // 4 Down
placeWord("BIAS", 8, 2, "down", 6, 6); // 6 Down

/* Clues */
const clues = ref({
  across: [
    {
      number: 1,
      text:
        "A look-alike pill with no active ingredient, used for comparison in trials.",
    },
    {
      number: 5,
      text:
        "The ____ process determines who receives the new medication and who receives a placebo, much like flipping a coin.",
    },
    {
      number: 7,
      text:
        "Reliable information from research that helps doctors make informed decisions.",
    },
  ],
  down: [
    {
      number: 2,
      text:
        "When a study compares one group with another, it is called a ______ study.",
    },
    {
      number: 3,
      text:
        "A method used in trials to keep participants or researchers unaware of who gets the real treatment.",
    },
    {
      number: 4,
      text:
        "The gold standard study design used to test if treatments truly work.",
    },
    {
      number: 6,
      text:
        "What is it called when people’s thoughts or expectations unintentionally affect the outcome of a study.",
    },
  ],
});

/* state */
const activeCell = ref<Cell | null>(null);
const currentDirection = ref<"across" | "down">("across");
const selectedClue = ref<any>(null);
const showResults = ref(false);
const containerRef = ref<HTMLElement | null>(null);

/* helpers to flatten / find */
function flatGrid() {
  return grid.value.flat();
}
function cellAt(r: number, c: number) {
  if (r < 0 || r >= ROWS || c < 0 || c >= COLS) return null;
  return grid.value[r][c];
}

/* find list of cells for a word (ordered) */
function cellsFor(cell: Cell | null, dir: "across" | "down") {
  if (!cell) return [];
  const id = dir === "across" ? cell.acrossId : cell.downId;
  if (id == null) return [];
  const list = flatGrid().filter((x) =>
    dir === "across" ? x.acrossId === id : x.downId === id
  );
  return dir === "across"
    ? list.sort((a, b) => a.c - b.c)
    : list.sort((a, b) => a.r - b.r);
}

/* UI helpers */
function cellClass(cell: Cell | null) {
  if (!cell) return "";
  return {
    blocked: cell.blocked,
    active: activeCell.value === cell,
    highlight: activeCell.value
      ? cellsFor(activeCell.value, currentDirection.value).includes(cell)
      : false,
    correct:
      showResults.value &&
      cell.value.toUpperCase() === cell.correct &&
      !cell.blocked,
    wrong:
      showResults.value &&
      cell.value &&
      cell.value.toUpperCase() !== cell.correct &&
      !cell.blocked,
  };
}

/* focus helpers using data attributes */
function focusInput(r: number, c: number) {
  nextTick(() => {
    const sel = document.querySelector<HTMLInputElement>(
      `input[data-r="${r}"][data-c="${c}"]`
    );
    sel?.focus();
    // select text for quick overwrite (if any)
    if (sel) sel.select();
  });
}

/* when input receives focus */
function onFocus(cell: Cell) {
  activeCell.value = cell;
  // choose direction heuristic: prefer across if cell has acrossId, else down
  if (cell.acrossId && !cell.downId) currentDirection.value = "across";
  else if (!cell.acrossId && cell.downId) currentDirection.value = "down";
}

/* clicking cell toggles direction when same cell clicked twice */
let lastClicked: Cell | null = null;
function onCellClick(cell: Cell | null) {
  if (!cell || cell.blocked) return;
  if (activeCell.value === cell) {
    // toggle direction
    currentDirection.value =
      currentDirection.value === "across" ? "down" : "across";
  }
  activeCell.value = cell;
  lastClicked = cell;
  // focus input
  focusInput(cell.r, cell.c);
}

/* input handler: uppercase + move inside same word */
function onInput(cell: Cell) {
  if (!cell) return;
  cell.value = (cell.value || "").toUpperCase().slice(0, 1);
  showResults.value = false;

  // move to next in same word (if any)
  const list = cellsFor(cell, currentDirection.value);
  const idx = list.indexOf(cell);
  if (idx >= 0 && idx < list.length - 1) {
    const next = list[idx + 1];
    focusInput(next.r, next.c);
  }
}

/* find next/prev cell within same word */
function findNextIn(cell: Cell, dir: "across" | "down") {
  const list = cellsFor(cell, dir);
  const idx = list.indexOf(cell);
  return idx >= 0 ? list[idx + 1] ?? null : null;
}
function findPrevIn(cell: Cell, dir: "across" | "down") {
  const list = cellsFor(cell, dir);
  const idx = list.indexOf(cell);
  return idx > 0 ? list[idx - 1] : null;
}

/* keyboard handling */
function onKeyDown(e: KeyboardEvent, cell: Cell) {
  if (!cell) return;
  const key = e.key;

  if (key === "Backspace") {
    // if cell has value -> clear; else move to previous cell in word
    if (cell.value) {
      cell.value = "";
    } else {
      const prev = findPrevIn(cell, currentDirection.value);
      if (prev) focusInput(prev.r, prev.c);
    }
    e.preventDefault();
    return;
  }

  if (key === "ArrowLeft") {
    currentDirection.value = "across";
    const prev = findPrevIn(cell, "across");
    if (prev) focusInput(prev.r, prev.c);
    e.preventDefault();
    return;
  }
  if (key === "ArrowRight") {
    currentDirection.value = "across";
    const next = findNextIn(cell, "across");
    if (next) focusInput(next.r, next.c);
    e.preventDefault();
    return;
  }
  if (key === "ArrowUp") {
    currentDirection.value = "down";
    const prev = findPrevIn(cell, "down");
    if (prev) focusInput(prev.r, prev.c);
    e.preventDefault();
    return;
  }
  if (key === "ArrowDown") {
    currentDirection.value = "down";
    const next = findNextIn(cell, "down");
    if (next) focusInput(next.r, next.c);
    e.preventDefault();
    return;
  }

  // For other character keys we do nothing here; `onInput` will handle moving forward.
}

/* clue selection: focus first cell of clue */
function selectClue(clue: any, dir: "across" | "down") {
  selectedClue.value = clue;
  currentDirection.value = dir;
  // find first cell with that number (first cell of that word)
  const start = flat().find((c) =>
    dir === "across" ? c.acrossId === clue.number : c.downId === clue.number
  );
  if (start) focusInput(start.r, start.c);
}

/* helpers */
function flat() {
  return grid.value.flat();
}

/* check / reset */
function checkAnswers() {
  showResults.value = true;
}
function resetPuzzle() {
  flat().forEach((c) => {
    if (c) c.value = "";
  });
  showResults.value = false;
  activeCell.value = null;
  selectedClue.value = null;
}

/* wire data-r/data-c attribute in DOM after mount is automatic because inputs are bound with those attributes in template */
</script>

<style scoped>
.crossword-wrapper {
  display: flex;
  flex-wrap: wrap;
  gap: 28px;
  padding: 16px;
  justify-content: center;
  align-items: flex-start;
  /** background: #eaf6ff; */
  /**  min-height: 95vh; */
  box-sizing: border-box;
}

.crossword-container {
  position: relative;
  width: min(90vw, 760px);
  background: #fff;
  border-radius: 10px;
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.08);
  overflow: hidden;
  display: flex;
  justify-content: center;
  align-items: center;
}

/* ✅ Full background image visible (no cut-off) */
.background-image {
  width: 100%;
  height: auto;
  max-height: 100%;
  object-fit: contain; /* show entire image */
  display: block;
  z-index: 1;
  position: relative;
}

/* ✅ Crossword overlay positioned in the center of image */
.crossword-table {
  position: absolute;
  top: 30%;
  left: 15%;
  /* transform: translate(0, -39%); */
  /* border-collapse: collapse; */
  z-index: 2;
  width: 74%;
  height: 37%;
}

/* Cell design */
.crossword-table td {
  border: 1.8px solid #19b2d2e6;
  padding: 0;
  position: relative;
  text-align: center;
  vertical-align: middle;
  background: white;
}

td.blocked {
  background: transparent;
  border-color: transparent;
}

.cross-input {
  width: 84%;
  border: none;
  font-weight: 700;
  text-align: center;
  background: rgba(255, 255, 255, 0.92);
  color: #000;
  outline: none;
  font-size: clamp(12px, 3vw, 1em);
}

.cell-number {
  position: absolute;
  top: 0px;
  left: 1px;
  font-size: clamp(8px, 2.5vw, 12px);
  color: #222;
  z-index: 3;
}

/* States */
td.active {
  border-color: #19d2c8eb;
  box-shadow: 0 0 0 1px rgba(25, 210, 200, 0.2) inset;
}
td.correct {
  /** background: #dff7d8; */
  border-color: #2e7d32;
}
td.wrong {
  /** background: #ffdada; */
  border-color: #d32f2f;
}

/* Buttons */
.buttons {
  display: flex;
  justify-content: center;
  gap: 12px;
  flex-wrap: wrap;
  margin-top: 12px;
  position: absolute;
  top: 65%;
  left: 46%;
  z-index: 99;
}
button {
  background: #1976d2;
  color: white;
  border: none;
  padding: 0.5em;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  transition: 0.2s;
}
button:hover {
  background: #125aa1;
}
button:active {
  transform: scale(0.96);
}

/* 📱 Responsive */
@media (max-width: 768px) {
  .crossword-container {
    width: 95vw;
  }
  .crossword-table {
    width: 82%;
  }
  .cross-input {
    font-size: clamp(10px, 4vw, 16px);
  }
}

@media (max-width: 480px) {
  .crossword-container {
    width: 25em;
    height: 35em;
  }
  .crossword-wrapper {
    display: flex;
    flex-wrap: wrap;
    gap: 28px;
    justify-content: center;
    align-items: flex-start;
    box-sizing: border-box;
    width: 25em;
  }
  .crossword-table {
    scale: 0.75;
    top: 23.3%;
    height: 47%;
    width: 101%;
    left: 0%;
  }
  .cell-number {
    scale: 0.6;
    top: -2%;
    left: 2%;
  }
  .cross-input {
    font-size: clamp(9px, 5vw, 14px);
    scale: 0.6;
    height: 56%;
    width: 95%;
  }

  td.active {
    box-shadow: 0 0 0 1px rgba(25, 210, 200, 0.2) inset;
  }

  .background-image {
    width: 108%;
    height: 100%;
    left: -0.6em;
  }

  .buttons {
    scale: 0.6;
    top: 22%;
    left: 38%;
  }
}
</style>
