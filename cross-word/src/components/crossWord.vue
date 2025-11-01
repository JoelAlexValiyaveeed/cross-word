<template>
  <div class="crossword-wrap">
    <div class="controls">
      <div class="status">Unsolved: {{unsolvedCount}}</div>
      <div class="buttons">
        <button @click="checkAll">Check</button>
        <button @click="revealAll">Reveal</button>
        <button @click="revealHint">Reveal Letter</button>
        <button @click="resetPuzzle">Reset</button>
      </div>
    </div>

    <div class="board" @keydown="onKeydown">
      <div
        v-for="(row, r) in grid"
        :key="r"
        class="row"
      >
        <div
          v-for="(cell, c) in row"
          :key="c"
          :class="['cell', {block: cell.block, focused: focused.r===r && focused.c===c, wrong: cell.state==='wrong'}]"
          @click="focusCell(r,c)"
        >
          <template v-if="!cell.block">
            <input
              ref="inputs"
              :data-r="r"
              :data-c="c"
              maxlength="1"
              v-model="cell.char"
              @input="onInput(r,c,$event)"
              @focus="setFocused(r,c)"
              @keydown.stop
            />
            <div class="num" v-if="cell.clueNum">{{cell.clueNum}}</div>
          </template>
        </div>
      </div>
    </div>

    <div class="clues">
      <div class="clue-col">
        <h3>Across</h3>
        <ul>
          <li v-for="cl in across" :key="cl.num" @click="selectClue(cl, 'across')">
            <strong>{{cl.num}}.</strong> {{cl.text}}
          </li>
        </ul>
      </div>

      <div class="clue-col">
        <h3>Down</h3>
        <ul>
          <li v-for="cl in down" :key="cl.num" @click="selectClue(cl, 'down')">
            <strong>{{cl.num}}.</strong> {{cl.text}}
          </li>
        </ul>
      </div>
    </div>

  </div>
</template>

<script setup>
import { reactive, ref, computed, onMounted, nextTick } from 'vue'

// ===== SAMPLE PUZZLE DEFINITION =====
// width, height, and list of across/down entries with coordinates
const samplePuzzle = {
  width: 10,
  height: 10,
  blocks: [
    /* array of [r,c] for black squares (0-indexed) */
    [0,3],[0,7],[1,7],[2,7],[3,1],[3,5],[4,5],[5,2],[5,6],[6,6],[7,0],[7,4],[8,4],[9,2]
  ],
  // answers used for checking (placed left-to-right / top-to-bottom)
  entries: [
    {num:1, dir:'across', r:0, c:0, len:3, answer:'SEA', text:'Large body of salt water'},
    {num:4, dir:'across', r:0, c:4, len:3, answer:'PORT', text:'Harbor (but 4-letter to demo)'.slice(0,3)},
    {num:6, dir:'across', r:1, c:0, len:7, answer:'SHIPYRD'.slice(0,7), text:'Place where ships are built'},
    {num:9, dir:'down', r:0, c:0, len:4, answer:'SALT', text:'Opposite of fresh'},
    {num:10, dir:'across', r:2, c:0, len:5, answer:'DOCKS', text:'Where ships tie up'},
    // ... add more entries or generate automatically
  ]
}

// ===== BUILD GRID =====
function buildGrid(puzzle){
  const grid = Array.from({length: puzzle.height}, (_,r)=>
    Array.from({length: puzzle.width}, (_,c)=>({
      block:false,
      char:'',
      solution:'',
      clueNum:null,
      state:'', // '', 'correct', 'wrong'
    }))
  )
  puzzle.blocks.forEach(([r,c])=>{
    if (r>=0 && r<puzzle.height && c>=0 && c<puzzle.width) grid[r][c].block = true
  })

  // Place entries (solution letters)
  puzzle.entries.forEach(e=>{
    let r=e.r, c=e.c
    for(let i=0;i<e.len;i++){
      if(!grid[r]||!grid[r][c]) break
      grid[r][c].solution = e.answer[i] ? e.answer[i].toUpperCase() : ''
      // mark clue number for the starting cell
      if(i===0) grid[r][c].clueNum = e.num
      if(e.dir==='across') c++
      else r++
    }
  })

  return grid
}

const puzzle = samplePuzzle
const grid = reactive(buildGrid(puzzle))

// collect clues across/down from puzzle.entries
const across = computed(()=>puzzle.entries.filter(e=>e.dir==='across'))
const down = computed(()=>puzzle.entries.filter(e=>e.dir==='down'))

const focused = reactive({r:0,c:0})
const inputs = ref([])

onMounted(()=>{
  // focus first non-block cell
  for(let r=0;r<grid.length;r++){
    for(let c=0;c<grid[r].length;c++){
      if(!grid[r][c].block){
        focused.r=r; focused.c=c; return
      }
    }
  }
})

function setFocused(r,c){ focused.r=r; focused.c=c }
function focusCell(r,c){ if(grid[r][c].block) return; focused.r=r; focused.c=c; nextTick(()=>{
  const el = getInputEl(r,c); el && el.focus()
}) }

function getInputEl(r,c){ const list = document.querySelectorAll('.board input');
  for(const el of list){ if(Number(el.dataset.r)===r && Number(el.dataset.c)===c) return el }
  return null
}

function onInput(r,c,e){
  const val = (e.target.value || '').toUpperCase().slice(0,1)
  grid[r][c].char = val
  // auto-move to next cell to the right if present
  moveFocus(r,c,'right')
}

function moveFocus(r,c,dir){
  const deltas = {left:[0,-1], right:[0,1], up:[-1,0], down:[1,0]}
  const [dr,dc] = deltas[dir]
  let nr=r+dr, nc=c+dc
  while(nr>=0 && nr<grid.length && nc>=0 && nc<grid[0].length){
    if(!grid[nr][nc].block){ focusCell(nr,nc); break }
    nr+=dr; nc+=dc
  }
}

function onKeydown(e){
  const r=focused.r, c=focused.c
  if(e.key==='ArrowRight') moveFocus(r,c,'right')
  else if(e.key==='ArrowLeft') moveFocus(r,c,'left')
  else if(e.key==='ArrowUp') moveFocus(r,c,'up')
  else if(e.key==='ArrowDown') moveFocus(r,c,'down')
  else if(e.key==='Backspace'){ // clear and move left
    grid[r][c].char=''
    moveFocus(r,c,'left')
    nextTick(()=>getInputEl(...Object.values(focused))?.focus())
  }
}

function checkAll(){
  let anyWrong=false
  for(let r=0;r<grid.length;r++){
    for(let c=0;c<grid[r].length;c++){
      const cell = grid[r][c]
      if(cell.block) continue
      if(cell.solution){
        if(cell.char.toUpperCase()===cell.solution.toUpperCase()) cell.state='correct'
        else { cell.state='wrong'; anyWrong=true }
      }
    }
  }
  return !anyWrong
}

function revealAll(){
  for(let r=0;r<grid.length;r++){
    for(let c=0;c<grid[r].length;c++){
      const cell = grid[r][c]
      if(!cell.block) cell.char = cell.solution || ''
    }
  }
}

function revealHint(){
  // reveal first empty or wrong letter
  for(let r=0;r<grid.length;r++){
    for(let c=0;c<grid[r].length;c++){
      const cell = grid[r][c]
      if(cell.block) continue
      if(!cell.char || (cell.solution && cell.char.toUpperCase()!==cell.solution.toUpperCase())){
        cell.char = cell.solution || ''
        cell.state = 'correct'
        focusCell(r,c)
        return
      }
    }
  }
}

function resetPuzzle(){
  for(let r=0;r<grid.length;r++){
    for(let c=0;c<grid[r].length;c++){
      const cell = grid[r][c]
      if(cell.block) continue
      cell.char=''
      cell.state=''
    }
  }
}

function selectClue(cl, dir){
  // focus start cell of clue
  focusCell(cl.r, cl.c)
}

const unsolvedCount = computed(()=>{
  let cnt=0
  for(let r=0;r<grid.length;r++){
    for(let c=0;c<grid[r].length;c++){
      const cell = grid[r][c]
      if(cell.block) continue
      if(cell.solution && cell.char.toUpperCase()!==cell.solution.toUpperCase()) cnt++
    }
  }
  return cnt
})

</script>

<style scoped>
.crossword-wrap{max-width:1100px;margin:1rem auto;font-family:system-ui,Segoe UI,Roboto,Arial}
.controls{display:flex;justify-content:space-between;align-items:center;margin-bottom:12px}
.buttons button{margin-left:8px;padding:6px 10px;border-radius:6px;border:1px solid #ccc;background:#fff}
.board{display:inline-block;border:4px solid #222;padding:8px;background:#f4f4f4}
.row{display:flex}
.cell{width:36px;height:36px;border:1px solid #999;position:relative;display:flex;align-items:center;justify-content:center;background:#fff}
.cell.block{background:#222;border-color:#222}
.cell input{width:100%;height:100%;border:0;text-align:center;font-size:18px;font-weight:600;outline:none;background:transparent}
.cell .num{position:absolute;top:2px;left:2px;font-size:10px;color:#333}
.cell.focused{box-shadow:0 0 0 3px rgba(50,120,230,0.12)}
.cell.wrong input{background:rgba(255,0,0,0.08)}
.clues{display:flex;gap:24px;margin-top:16px}
.clue-col{flex:1}
.clue-col h3{margin:0 0 8px}
.clue-col ul{list-style:none;padding:0;margin:0}
.clue-col li{padding:4px 0;cursor:pointer}
.status{font-weight:600}
</style>
