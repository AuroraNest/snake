<script setup lang="ts">
import { onMounted, onUnmounted, ref, watch } from 'vue'

type Point = { x: number; y: number }
type Penalty = { x: number; y: number; expiresAt: number }

const CELL = 20
const COLS = 30
const ROWS = 30
const W = COLS * CELL
const H = ROWS * CELL

// 难度
type Difficulty = 'slow' | 'normal' | 'fast'
const SPEEDS: Record<Difficulty, number> = { slow: 180, normal: 120, fast: 80 }
const difficulty = ref<Difficulty>('normal')

// 画布
const canvasRef = ref<HTMLCanvasElement | null>(null)
let ctx: CanvasRenderingContext2D | null = null

// 状态
const snake = ref<Point[]>([])
const dir = ref<Point>({ x: 1, y: 0 })
let nextDir: Point = { x: 1, y: 0 }
type Actor = { body: Point[]; dir: Point; nextDir: Point; alive: boolean; color: string; prevBody: Point[]; isPlayer?: boolean }
const bots = ref<Actor[]>([])
const foods = ref<Point[]>([])
const penalties = ref<Penalty[]>([])
const score = ref(0)
const highScore = ref(0)
const recentScores = ref<number[]>([])
const started = ref(false)
const paused = ref(false)
const gameOver = ref(false)

// 平滑 & 视觉
let prevSnake: Point[] = []
let lastTick = 0
let lastFrame = 0
let rafId: number | null = null
let grassPattern: CanvasPattern | null = null

// 存储键
const LS_RECENT = 'snake_recent_scores'
const LS_HIGH = 'snake_high_score'

function loadScores() {
  try { recentScores.value = JSON.parse(localStorage.getItem(LS_RECENT) || '[]') || [] } catch { recentScores.value = [] }
  const h = Number(localStorage.getItem(LS_HIGH) || '0'); highScore.value = Number.isFinite(h) ? h : 0
}
function pushScore(s: number) {
  const arr = [...recentScores.value, Math.max(0, Math.floor(s))].slice(-5)
  recentScores.value = arr
  localStorage.setItem(LS_RECENT, JSON.stringify(arr))
  if (s > highScore.value) { highScore.value = s; localStorage.setItem(LS_HIGH, String(highScore.value)) }
}

// 导出到本地文件（浏览器下载）
function exportCSV() {
  const lines = [ '类型,分数', `最高分,${highScore.value}`, ...recentScores.value.map((v,i)=>`最近${i+1},${v}`) ]
  const blob = new Blob([ '\uFEFF' + lines.join('\n') ], { type: 'text/csv;charset=utf-8;' })
  const a = document.createElement('a'); a.href = URL.createObjectURL(blob); a.download = 'snake_scores.csv'; a.click(); URL.revokeObjectURL(a.href)
}
function exportTXT() {
  const lines = [ `最高分: ${highScore.value}`, ...recentScores.value.map((v,i)=>`最近 ${i+1}: ${v}`) ]
  const blob = new Blob([ lines.join('\n') ], { type: 'text/plain;charset=utf-8;' })
  const a = document.createElement('a'); a.href = URL.createObjectURL(blob); a.download = 'snake_scores.txt'; a.click(); URL.revokeObjectURL(a.href)
}

// 新的初始化，避免旧实现的编码问题
function resetGame2() {
  const cx = Math.floor(COLS / 2)
  const cy = Math.floor(ROWS / 2)
  snake.value = [ { x: cx, y: cy }, { x: cx - 1, y: cy }, { x: cx - 2, y: cy } ]
  dir.value = { x: 1, y: 0 }
  nextDir = { x: 1, y: 0 }
  score.value = 0
  gameOver.value = false
  paused.value = false
  foods.value = []
  penalties.value = []
  bots.value = []
  const mkBot = (x:number, y:number, dx:number, dy:number, color:string): Actor => ({
    body: [ { x, y }, { x: x - dx, y: y - dy }, { x: x - 2*dx, y: y - 2*dy } ],
    dir: { x: dx, y: dy },
    nextDir: { x: dx, y: dy },
    alive: true,
    color,
    prevBody: [],
    isPlayer: false,
  })
  bots.value.push(mkBot(2, 2, 1, 0, '#f59e0b'))
  bots.value.push(mkBot(COLS - 3, ROWS - 3, -1, 0, '#8b5cf6'))
  placeFoods(true)
  prevSnake = snake.value.map(p => ({ x: p.x, y: p.y }))
  for (const b of bots.value) { b.prevBody = b.body.map(p => ({ x: p.x, y: p.y })) }
}

/* function resetGame() {
  const cx = Math.floor(COLS/2), cy = Math.floor(ROWS/2)
  snake.value = [ {x:cx,y:cy}, {x:cx-1,y:cy}, {x:cx-2,y:cy} ]
  dir.value = { x:1, y:0 }; nextDir = { x:1, y:0 }
  score.value = 0; gameOver.value = false; paused.value = false
  foods.value = []; penalties.value = []
  // 初始化两个机器人蛇
  bots.value = []
  const mkBot = (x:number,y:number,dx:number,dy:number,color:string): Actor => ({ body:[{x,y},{x-dx,y-dy},{x-2*dx,y-2*dy}], dir:{x:dx,y:dy}, nextDir:{x:dx,y:dy}, alive:true, color, prevBody:[], isPlayer:false })
  bots.value.push(mkBot(2,2,1,0,'#f59e0b'))
  bots.value.push(mkBot(COLS-3,ROWS-3,-1,0,'#8b5cf6'))
  placeFoods(true); prevSnake = snake.value.map(p=>({...p}))
  for(const b of bots.value) b.prevBody = b.body.map(p=>({...p}))
} */

function startGame() { resetGame2(); started.value = true; startLoop() }
function stopGame() { if (rafId) cancelAnimationFrame(rafId); rafId = null }

function getTickMs() { return SPEEDS[difficulty.value] }
function isOpposite(a: Point, b: Point) { return a.x + b.x === 0 && a.y + b.y === 0 }
function cellEq(a: Point, b: Point) { return a.x === b.x && a.y === b.y }

function handleKey(e: KeyboardEvent) {
  if (e.key === ' ' || e.code === 'Space') { if (!started.value || gameOver.value) return; e.preventDefault(); paused.value = !paused.value; return }
  if (!started.value || gameOver.value) return
  let nd = nextDir
  switch(e.key){
    case 'ArrowUp': case 'w': case 'W': e.preventDefault(); nd = {x:0,y:-1}; break
    case 'ArrowDown': case 's': case 'S': e.preventDefault(); nd = {x:0,y:1}; break
    case 'ArrowLeft': case 'a': case 'A': e.preventDefault(); nd = {x:-1,y:0}; break
    case 'ArrowRight': case 'd': case 'D': e.preventDefault(); nd = {x:1,y:0}; break
    default: return
  }
  if (!isOpposite(nd, dir.value)) nextDir = nd
}
function handleControlKeys(e: KeyboardEvent){ if(e.key==='r'||e.key==='R'){ e.preventDefault(); startGame() } }

function startLoop(){
  lastTick = performance.now(); lastFrame = lastTick
  const frame = (ts:number)=>{
    const dt = ts - lastFrame; lastFrame = ts
    if (started.value && !paused.value && !gameOver.value) {
      if (ts - lastTick >= getTickMs()) { doTickMulti(); lastTick = ts }
      updatePenalties(); maybeSpawnPenalty()
    }
    draw(Math.max(0, Math.min(1, (ts - lastTick) / getTickMs())))
    rafId = requestAnimationFrame(frame)
  }
  rafId = requestAnimationFrame(frame)
}

function randEmpty(): Point {
  const occ = new Set<string>([
    ...snake.value.map(p=>`${p.x},${p.y}`),
    ...bots.value.flatMap(b=>b.body.map(p=>`${p.x},${p.y}`)),
    ...foods.value.map(p=>`${p.x},${p.y}`),
    ...penalties.value.map(p=>`${p.x},${p.y}`),
  ])
  let x=0,y=0,c=0; do{ x=Math.floor(Math.random()*COLS); y=Math.floor(Math.random()*ROWS); c++ }while(occ.has(`${x},${y}`)&&c<2000)
  return {x,y}
}
function placeFoods(init=false){
  const min=1,max=3; if(init) foods.value=[]
  while(foods.value.length<min) foods.value.push(randEmpty())
  if(!init && foods.value.length<max && Math.random()<0.35) foods.value.push(randEmpty())
}
function maybeSpawnPenalty(){ if(penalties.value.length>=1) return; if(Math.random()<0.12){ const p=randEmpty(); penalties.value.push({x:p.x,y:p.y,expiresAt:performance.now()+ (3000+Math.random()*3000)}) } }
function updatePenalties(){ const now=performance.now(); penalties.value = penalties.value.filter(p=>p.expiresAt>now) }

// 多蛇版本推进一帧
function doTickMulti(){
  // 更新方向（玩家 + 机器人）
// function old_doTick(){
/*  if (!isOpposite(nextDir, dir.value)) dir.value = nextDir
  for (const b of bots.value){ if (b.alive){ chooseBotDir(b); if (!isOpposite(b.nextDir, b.dir)) b.dir = b.nextDir } }

  const actors = [
    { kind:'player' as const, body: snake.value, dir: dir.value },
    ...bots.value.filter(b=>b.alive).map(b=>({ kind:'bot' as const, bot:b, body:b.body, dir:b.dir }))
  ]

  // 所有当前占用位置
  const allOcc = new Set<string>([
    ...snake.value.map(p=>`${p.x},${p.y}`),
    ...bots.value.flatMap(b=>b.body.map(p=>`${p.x},${p.y}`)),
  ])

  type Plan = { idx:number; kind:'player'|'bot'; bot?:Actor; newHead:Point; willEat:boolean; fIdx:number; willPenalty:boolean; pIdx:number; alive:boolean }
  const plans: Plan[] = []

  // 阶段1：计算意图并进行身体/墙碰撞检测（考虑尾巴移动）
  actors.forEach((a,idx)=>{
    const head = a.body[0]
    const nh = { x: head.x + a.dir.x, y: head.y + a.dir.y }
    let alive = true
    if (nh.x<0||nh.x>=COLS||nh.y<0||nh.y>=ROWS) alive = false
    const fIdx = foods.value.findIndex(f=>cellEq(f,nh))
    const willEat = fIdx!==-1
    const pIdx = fIdx===-1 ? penalties.value.findIndex(p=>p.x===nh.x&&p.y===nh.y) : -1
    const willPenalty = pIdx!==-1

    // 身体碰撞：把所有当前占用加入集合，并移除本蛇尾巴（若不增长）
    const occ = new Set(allOcc)
    if (!willEat){ const tail = a.body[a.body.length-1]; occ.delete(`${tail.x},${tail.y}`) }
    if (alive && occ.has(`${nh.x},${nh.y}`)) alive = false
    plans.push({ idx, kind: a.kind, bot: (a as any).bot, newHead: nh, willEat, fIdx, willPenalty, pIdx, alive })
  })

  // 头对头碰撞：同一网格的多头全部死亡
  const headCount = new Map<string, number>()
  for (const p of plans.filter(p=>p.alive)){
    const k = `${p.newHead.x},${p.newHead.y}`
    headCount.set(k, (headCount.get(k)||0)+1)
  }
  for (const p of plans){ if (p.alive){ const k=`${p.newHead.x},${p.newHead.y}`; if ((headCount.get(k)||0) > 1) p.alive=false } }

  // 阶段2：提交移动与效果
  prevSnake = snake.value.map(p=>({...p}))
  bots.value.forEach(b=>{ b.prevBody = b.body.map(p=>({...p})) })

  for (const p of plans){
    const isPlayer = p.kind==='player'
    const body = isPlayer ? snake.value : (p.bot!.body)
    if (!p.alive){
      if (isPlayer){ onGameOver(); return } else { p.bot!.alive=false; continue }
    }
    // 先移动
    body.unshift(p.newHead)
    if (p.willEat){
      if (isPlayer) score.value += 1
      foods.value.splice(p.fIdx,1); placeFoods(false)
    } else {
      body.pop()
    }
    // 扣分点
    if (p.willPenalty){
      penalties.value.splice(p.pIdx,1)
      if (isPlayer) score.value = Math.max(0, score.value - 1)
      const newLen = Math.floor(body.length * 0.5)
      if (newLen < 1){ if (isPlayer){ onGameOver(); return } else { p.bot!.alive=false; continue } }
      body.splice(newLen)
    }
  }
} */
  if (!isOpposite(nextDir, dir.value)) dir.value = nextDir
  const head = snake.value[0]
  const newHead = { x: head.x + dir.value.x, y: head.y + dir.value.y }
  if (newHead.x<0||newHead.x>=COLS||newHead.y<0||newHead.y>=ROWS){ onGameOver(); return }

  const fIdx = foods.value.findIndex(f=>cellEq(f,newHead))
  const pIdx = fIdx===-1 ? penalties.value.findIndex(p=>p.x===newHead.x&&p.y===newHead.y) : -1
  const willEat = fIdx!==-1
  const willPenalty = pIdx!==-1
  const checkBody = snake.value.slice(0, snake.value.length - (willEat ? 0 : 1))
  if (checkBody.some(p=>cellEq(p,newHead))){ onGameOver(); return }

  prevSnake = snake.value.map(p=>({...p}))
  snake.value.unshift(newHead)
  if (willEat) {
    score.value += 1
    foods.value.splice(fIdx,1)
    placeFoods(false)
  } else {
    snake.value.pop()
  }
  if (willPenalty){
    // 命中扣分点：分数 -1，长度缩减 50%
    penalties.value.splice(pIdx,1)
    score.value = Math.max(0, score.value - 1)
    const newLen = Math.floor(snake.value.length * 0.5)
    if (newLen < 1) { onGameOver(); return }
    snake.value.splice(newLen)
  }
}

function onGameOver(){ gameOver.value = true; stopGame(); pushScore(score.value) }

// 绘制
function drawGrid(){ if(!ctx) return; ctx.strokeStyle='rgba(255,255,255,0.06)'; ctx.lineWidth=1; ctx.beginPath(); for(let x=0;x<=COLS;x++){ctx.moveTo(x*CELL+0.5,0);ctx.lineTo(x*CELL+0.5,H)} for(let y=0;y<=ROWS;y++){ctx.moveTo(0,y*CELL+0.5);ctx.lineTo(W,y*CELL+0.5)} ctx.stroke() }
function ensureGrass(){ if(!ctx||grassPattern) return; const off=document.createElement('canvas'); off.width=140; off.height=140; const g=off.getContext('2d')!; const bg=g.createLinearGradient(0,0,140,140); bg.addColorStop(0,'#0f2d1a'); bg.addColorStop(1,'#0b2415'); g.fillStyle=bg; g.fillRect(0,0,140,140); for(let i=0;i<220;i++){const x=Math.random()*140,y=Math.random()*140,w=1+Math.random()*2,h=1+Math.random()*2; g.fillStyle=`rgba(${50+Math.floor(Math.random()*30)}, ${100+Math.floor(Math.random()*40)}, ${60+Math.floor(Math.random()*30)}, ${0.06+Math.random()*0.08})`; g.fillRect(x,y,w,h)} g.strokeStyle='rgba(255,255,255,0.05)'; for(let i=0;i<50;i++){const x=Math.random()*140,y=Math.random()*140; g.beginPath(); g.moveTo(x,y); g.lineTo(x+Math.random()*6-3,y-(3+Math.random()*6)); g.stroke()} grassPattern = ctx.createPattern(off,'repeat') }
function drawBg(){ if(!ctx) return; ensureGrass(); ctx.fillStyle = grassPattern || '#0f2d1a'; ctx.fillRect(0,0,W,H); drawGrid(); const vg=ctx.createRadialGradient(W/2,H/2,Math.min(W,H)*0.35,W/2,H/2,Math.max(W,H)*0.75); vg.addColorStop(0,'rgba(0,0,0,0)'); vg.addColorStop(1,'rgba(0,0,0,0.22)'); ctx.fillStyle=vg; ctx.fillRect(0,0,W,H) }

function drawFoods(){ if(!ctx) return; for(const f of foods.value){ const cx=f.x*CELL+CELL/2, cy=f.y*CELL+CELL/2, r=CELL*0.35; const grd=ctx.createRadialGradient(cx-r*0.4,cy-r*0.4,r*0.2,cx,cy,r); grd.addColorStop(0,'#ff9a9a'); grd.addColorStop(0.4,'#ef4444'); grd.addColorStop(1,'#b91c1c'); ctx.fillStyle=grd; ctx.beginPath(); ctx.arc(cx,cy,r,0,Math.PI*2); ctx.fill(); ctx.fillStyle='#22c55e'; ctx.beginPath(); ctx.ellipse(cx+r*0.3,cy-r*0.9,r*0.28,r*0.16,-0.6,0,Math.PI*2); ctx.fill(); ctx.fillStyle='rgba(255,255,255,0.45)'; ctx.beginPath(); ctx.ellipse(cx-r*0.3,cy-r*0.35,r*0.25,r*0.18,0.5,0,Math.PI*2); ctx.fill() } }
function drawPenalties(){ if(!ctx) return; const now=performance.now(); for(const p of penalties.value){ const cx=p.x*CELL+CELL/2, cy=p.y*CELL+CELL/2, r=CELL*0.34; const grd=ctx.createRadialGradient(cx-r*0.3,cy-r*0.3,r*0.2,cx,cy,r); grd.addColorStop(0,'#fde68a'); grd.addColorStop(1,'#f59e0b'); ctx.fillStyle=grd; ctx.beginPath(); ctx.arc(cx,cy,r,0,Math.PI*2); ctx.fill(); const t=Math.max(0,Math.min(1,(p.expiresAt-now)/6000)); ctx.strokeStyle='rgba(0,0,0,0.5)'; ctx.lineWidth=2; ctx.beginPath(); ctx.arc(cx,cy,r+2,-Math.PI/2,-Math.PI/2+Math.PI*2*t); ctx.stroke(); ctx.strokeStyle='#7c2d12'; ctx.lineWidth=3; ctx.beginPath(); ctx.moveTo(cx-r*0.45,cy); ctx.lineTo(cx+r*0.45,cy); ctx.stroke() } }

function drawSnakeOne(body:Point[], prev:Point[], color:string, t=0){ if(!ctx||body.length===0) return; const centers = body.map((p,i)=>{ const pp=prev[i]??p; const rx=pp.x+(p.x-pp.x)*t, ry=pp.y+(p.y-pp.y)*t; return {x:rx*CELL+CELL/2,y:ry*CELL+CELL/2} }); const tail=[...centers].reverse(); ctx.save(); ctx.lineJoin='round'; ctx.lineCap='round'; for(let i=1;i<tail.length;i++){ const p1=tail[i-1],p2=tail[i]; const scale=0.55+0.45*(i/(tail.length-1)); ctx.strokeStyle='rgba(0,0,0,0.25)'; ctx.lineWidth=CELL*scale*0.95; ctx.beginPath(); ctx.moveTo(p1.x+1.2,p1.y+1.2); ctx.lineTo(p2.x+1.2,p2.y+1.2); ctx.stroke() } for(let i=1;i<tail.length;i++){ const p1=tail[i-1],p2=tail[i]; const scale=0.55+0.45*(i/(tail.length-1)); ctx.strokeStyle=color; ctx.lineWidth=CELL*scale; ctx.beginPath(); ctx.moveTo(p1.x,p1.y); ctx.lineTo(p2.x,p2.y); ctx.stroke() } const head=centers[0], second=centers[1]??head; const dx=head.x-second.x, dy=head.y-second.y; const len=Math.hypot(dx,dy)||1; const nx=dx/len, ny=dy/len; const eyeOff=CELL*0.28, px=-ny, py=nx; const e1={x:head.x+px*eyeOff,y:head.y+py*eyeOff}, e2={x:head.x-px*eyeOff,y:head.y-py*eyeOff}; ctx.fillStyle='#fff'; ctx.beginPath(); ctx.arc(e1.x,e1.y,CELL*0.14,0,Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.arc(e2.x,e2.y,CELL*0.14,0,Math.PI*2); ctx.fill(); ctx.fillStyle='#0b1220'; ctx.beginPath(); ctx.arc(e1.x+nx*CELL*0.06,e1.y+ny*CELL*0.06,CELL*0.07,0,Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.arc(e2.x+nx*CELL*0.06,e2.y+ny*CELL*0.06,CELL*0.07,0,Math.PI*2); ctx.fill(); ctx.fillStyle='#f43f5e'; ctx.beginPath(); const tip={x:head.x+nx*CELL*0.7,y:head.y+ny*CELL*0.7}, tl={x:head.x+nx*CELL*0.35+px*CELL*0.06,y:head.y+ny*CELL*0.35+py*CELL*0.06}, tr={x:head.x+nx*CELL*0.35-px*CELL*0.06,y:head.y+ny*CELL*0.35-py*CELL*0.06}; ctx.moveTo(tl.x,tl.y); ctx.lineTo(tip.x,tip.y); ctx.lineTo(tr.x,tr.y); ctx.closePath(); ctx.fill(); ctx.restore() }

function draw(t=0){ if(!ctx) return; drawBg(); drawFoods(); drawPenalties(); drawSnakeOne(snake.value, prevSnake, '#22d3ee', t); for(const b of bots.value){ if(b.alive) drawSnakeOne(b.body, b.prevBody, b.color, t) } if (paused.value || gameOver.value){ ctx.fillStyle='rgba(0,0,0,0.35)'; ctx.fillRect(0,0,W,H); ctx.fillStyle='#fff'; ctx.font='bold 28px system-ui, -apple-system, Segoe UI, Roboto, sans-serif'; ctx.textAlign='center'; ctx.textBaseline='middle'; ctx.fillText(gameOver.value?'游戏结束':'暂停中', W/2, H/2-20); ctx.font='16px system-ui, -apple-system, Segoe UI, Roboto, sans-serif'; ctx.fillText('R 重开，空格 暂停/继续', W/2, H/2+14) } }

onMounted(()=>{ loadScores(); const c=canvasRef.value; if(!c) return; const g=c.getContext('2d'); if(!g) return; ctx=g; resetGame2(); draw(0); window.addEventListener('keydown', handleKey); window.addEventListener('keydown', handleControlKeys); (document.body as HTMLBodyElement).dataset.prevOverflow = document.body.style.overflow; document.body.style.overflow='hidden' })
onUnmounted(()=>{ stopGame(); window.removeEventListener('keydown', handleKey); window.removeEventListener('keydown', handleControlKeys); const prev=(document.body as HTMLBodyElement).dataset.prevOverflow??''; document.body.style.overflow=prev })
watch(difficulty,()=>{ lastTick = performance.now() })

function queueDir(x:number,y:number){ const nd={x,y}; if(!isOpposite(nd,dir.value)) nextDir = nd }
</script>

<template>
  <div class="page">
    <div class="sidebar">
      <div class="scores">
        <div class="row"><span>分数</span><strong>{{ score }}</strong></div>
        <div class="row"><span>历史最高</span><strong>{{ highScore }}</strong></div>
      </div>
      <div class="controls">
        <button class="btn" @click="startGame" :disabled="started && !gameOver">{{ started && !gameOver ? '进行中' : '开始/重开' }}</button>
        <button class="btn" @click="paused = !paused" :disabled="!started || gameOver">{{ paused ? '继续' : '暂停' }}</button>
      </div>
      <div class="difficulty">
        <label for="diff">难度：</label>
        <select id="diff" v-model="difficulty">
          <option value="slow">慢速</option>
          <option value="normal">正常</option>
          <option value="fast">快速</option>
        </select>
      </div>
      <div class="recent">
        <div class="title">最近 5 次得分</div>
        <ul>
          <li v-if="recentScores.length === 0">暂无记录</li>
          <li v-for="(s, i) in recentScores" :key="i">第 {{ i + 1 }} 次：{{ s }}</li>
        </ul>
        <div class="export">
          <button class="btn" @click="exportCSV">导出 CSV</button>
          <button class="btn" @click="exportTXT">导出 TXT</button>
        </div>
      </div>
      <div class="dpad" aria-label="方向控制">
        <div class="row">
          <button class="arrow" @click="queueDir(0,-1)" title="上">↑</button>
        </div>
        <div class="row">
          <button class="arrow" @click="queueDir(-1,0)" title="左">←</button>
          <button class="arrow" @click="queueDir(1,0)" title="右">→</button>
        </div>
        <div class="row">
          <button class="arrow" @click="queueDir(0,1)" title="下">↓</button>
        </div>
      </div>
      <div class="help">
        <div>玩法变化</div>
        <ul>
          <li>同时可能有多个得分点</li>
          <li>新增敌对蛇：和它或墙相撞即死</li>
          <li>会随机出现“扣分点”（短时间内消失）</li>
          <li>吃到扣分点：分数 -1，长度 -50%</li>
        </ul>
      </div>
    </div>
    <div class="board"><canvas ref="canvasRef" :width="W" :height="H" /></div>
  </div>
</template>

<style scoped>
.page { display: grid; grid-template-columns: 260px 1fr; gap: 16px; align-items: start; }
.sidebar { background: #0f172a; border: 1px solid #1f2937; border-radius: 8px; color: #e5e7eb; padding: 12px; }
.scores .row { display: flex; justify-content: space-between; margin: 6px 0; }
.controls { display: flex; gap: 8px; margin: 10px 0 6px; flex-wrap: wrap; }
.btn { background: #1f2937; color: #e5e7eb; border: 1px solid #374151; border-radius: 6px; padding: 6px 10px; cursor: pointer; }
.btn:disabled { opacity: 0.6; cursor: default; }
.difficulty { margin: 8px 0 12px; display: flex; align-items: center; gap: 8px; }
.difficulty select { background: #111827; color: #e5e7eb; border: 1px solid #374151; border-radius: 6px; padding: 6px 8px; }
.recent .title { margin-top: 8px; font-weight: 600; }
.recent ul { list-style: none; padding: 0; margin: 6px 0 8px; }
.recent li { padding: 2px 0; color: #cbd5e1; }
.export { display:flex; gap:8px; }
.dpad { display: grid; place-items: center; gap: 4px; margin-top: 8px; }
.dpad .row { display: flex; gap: 6px; }
.arrow { width: 40px; height: 36px; border-radius: 6px; background: #1f2937; color: #e5e7eb; border: 1px solid #374151; cursor: pointer; }
.help ul { list-style: disc; margin: 6px 0 0 18px; color: #cbd5e1; }
.board { display: flex; justify-content: center; }
canvas { background: #0b1220; border: 1px solid #1f2937; border-radius: 8px; box-shadow: 0 2px 10px rgba(0,0,0,0.25); }
</style>
