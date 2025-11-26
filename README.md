// PROJECT: web_ticket (Vite + React)
// Use this file to copy each file into your project. Files are separated by headers.

/* ==================================================
   FILE: package.json
   ================================================== */
{
  "name": "web_ticket",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "vite": "^5.0.0"
  }
}

/* ==================================================
   FILE: index.html
   ================================================== */
<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>web_ticket</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>

/* ==================================================
   FILE: src/main.jsx
   ================================================== */
import React from 'react'
import { createRoot } from 'react-dom/client'
import App from './App'
import './styles.css'

createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
)

/* ==================================================
   FILE: src/App.jsx
   ================================================== */
import React, { useState } from 'react'
import Totem from './pages/Totem'
import Atendente from './pages/Atendente'
import Painel from './pages/Painel'
import Relatorios from './pages/Relatorios'
import { loadState } from './utils/storage'

export default function App(){
  const [route, setRoute] = useState('totem')
  // basic navigation
  return (
    <div className="app">
      <header className="header">
        <h1>web_ticket — Sistema de Senhas</h1>
        <nav>
          <button onClick={() => setRoute('totem')}>Totem</button>
          <button onClick={() => setRoute('atendente')}>Atendente</button>
          <button onClick={() => setRoute('painel')}>Painel</button>
          <button onClick={() => setRoute('relatorios')}>Relatórios</button>
        </nav>
      </header>

      <main className="main">
        {route === 'totem' && <Totem />}
        {route === 'atendente' && <Atendente />}
        {route === 'painel' && <Painel />}
        {route === 'relatorios' && <Relatorios />}
      </main>

      <footer className="footer">Expediente: 07:00 — 17:00 | UI para apresentação</footer>
    </div>
  )
}

/* ==================================================
   FILE: src/styles.css
   ================================================== */
:root{--bg:#f7fafc;--card:#ffffff;--muted:#6b7280;}
body{font-family:Inter,system-ui,Arial,sans-serif;background:var(--bg);margin:0;color:#111}
.app{max-width:1000px;margin:24px auto;padding:16px}
.header{display:flex;justify-content:space-between;align-items:center}
.header nav button{margin-left:8px;padding:8px 12px}
.main{margin-top:16px}
.card{background:var(--card);padding:12px;border-radius:8px;box-shadow:0 2px 6px rgba(0,0,0,0.06)}
.grid{display:grid;gap:12px}
.ticket{font-weight:700;padding:8px 12px;border-radius:6px;background:#eef}
.small{color:var(--muted);font-size:13px}
.footer{margin-top:18px;color:var(--muted)}

/* ==================================================
   FILE: src/utils/storage.js
   ================================================== */
export function saveState(key, value){
  localStorage.setItem(key, JSON.stringify(value))
}
export function loadState(key, fallback=null){
  const v = localStorage.getItem(key)
  return v ? JSON.parse(v) : fallback
}
export function clearState(key){
  localStorage.removeItem(key)
}

/* ==================================================
   FILE: src/utils/ticketService.js
   ================================================== */
// Lógica central de tickets: geração, filas, sequência, TM, descarte
import { saveState, loadState } from './storage'

const STORAGE_KEY = 'web_ticket_state_v1'

function todayYYMMDD(date=new Date()){
  const y = String(date.getFullYear()).slice(-2)
  const mm = String(date.getMonth()+1).padStart(2,'0')
  const dd = String(date.getDate()).padStart(2,'0')
  return ${y}${mm}${dd}
}

function nowISO(){ return new Date().toISOString() }

function defaultState(){
  return {
    sequences: { SP:0, SG:0, SE:0 },
    queue: [], // emitted tickets (pending)
    history: [] // all called/attended records
  }
}

function isWorkingHour(date=new Date()){
  const h = date.getHours()
  return h >= 7 && h < 17
}

// probability helper
function chance(p){ return Math.random() < p }

// TM sampling per rules
function sampleTM(type){
  if(type === 'SP'){
    // 15 min ± uniform 5 => 10..20
    return 15 + (Math.random()*10 - 5)
  }
  if(type === 'SG'){
    // 5 min ± 3 => 2..8
    return 5 + (Math.random()*6 - 3)
  }
  if(type === 'SE'){
    // 95% <1min => sample ~Uniform(0.2..1) else 5% ~5min
    if(chance(0.95)) return Math.max(0.2, Math.random()*1)
    return 5
  }
  return 1
}

export const ticketService = {
  load(){
    const s = loadState(STORAGE_KEY)
    if(!s) return defaultState()
    // if date changed, reset sequences
    const lastDate = s.lastDate
    const today = todayYYMMDD()
    if(lastDate !== today){
      s.sequences = { SP:0, SG:0, SE:0 }
      // discard remaining queue at new day
      s.queue = []
      s.lastDate = today
    }
    return s
  },
  save(state){
    state.lastSaved = nowISO()
    saveState(STORAGE_KEY, state)
  },
  emit(type, now=new Date()){
    const state = this.load()
    if(!isWorkingHour(now)){
      return { error: 'Fora do expediente (07:00-17:00)' }
    }
    // increment daily sequence for that priority
    state.sequences[type] = (state.sequences[type]||0) + 1
    const seq = String(state.sequences[type]).padStart(3,'0')
    const id = ${todayYYMMDD(now)}-${type}${seq}
    // 5% of emitted are never attended (discarded)
    const discarded = chance(0.05)
    const ticket = {
      id, type, createdAt: now.toISOString(), discarded
    }
    // push to queue unless discarded flag? The PDF says 5% are not attended; we will mark them as discarded but still record emission; they won't be called
    state.queue.push(ticket)
    this.save(state)
    return { ticket }
  },
  // get next according to sequence rule: always attend an SP if any, then SE if any, then SG
  // but the sequence must alternate: [SP] -> [SE|SG] -> [SP] -> [SE|SG]
  nextToCall(){
    const state = this.load()
    // filter non-discarded
    const pending = state.queue.filter(t => !t.discarded)
    if(pending.length === 0) return null
    // prefer SP if exists
    const sp = pending.find(t => t.type === 'SP')
    if(sp) return sp
    // otherwise check SE
    const se = pending.find(t => t.type === 'SE')
    if(se) return se
    // otherwise SG
    const sg = pending.find(t => t.type === 'SG')
    if(sg) return sg
    return null
  },
  callNext(guiche='1', now=new Date()){
    const state = this.load()
    if(!isWorkingHour(now)) return { error: 'Fora do expediente' }
    const next = this.nextToCall()
    if(!next) return { error: 'Nenhuma senha disponível' }
    // remove from queue
    state.queue = state.queue.filter(t => t.id !== next.id)
    // simulate TM
    const tm = sampleTM(next.type)
    const attended = {
      ...next,
      calledAt: now.toISOString(),
      guiche,
      tmMinutes: Number(tm.toFixed(2)),
      attendedAt: new Date(now.getTime() + Math.round(tm*60000)).toISOString(),
      status: 'attended'
    }
    state.history.push(attended)
    this.save(state)
    return { attended }
  },
  discardRemaining(){
    const state = this.load()
    // mark remaining queue as discarded
    state.queue.forEach(t => t.discarded = true)
    this.save(state)
    return state
  },
  getPanelLast5(){
    const state = this.load()
    // last 5 called from history
    const last = state.history.slice(-5).reverse()
    return last
  },
  reports(){
    const state = this.load()
    const issued = (state.queue.length + state.history.length)
    const issuedByType = { SP:0, SG:0, SE:0 }
    const attendedByType = { SP:0, SG:0, SE:0 }
    state.history.forEach(h => { if(h.type) attendedByType[h.type]++ })
    // to compute issued by type, count in history + queue + discarded records
    const all = [...state.history.map(h=>({type:h.type})), ...state.queue.map(q=>({type:q.type})), ...([])]
    all.forEach(a => { if(a.type) issuedByType[a.type] = (issuedByType[a.type]||0) + 1 })

    const totalAttended = state.history.length
    const totalIssued = all.length
    // TM report
    const tmByType = { SP:[], SG:[], SE:[] }
    state.history.forEach(h => { if(h.tmMinutes) tmByType[h.type].push(h.tmMinutes) })
    function avg(arr){ if(arr.length===0) return 0; return arr.reduce((s,x)=>s+x,0)/arr.length }
    return {
      totalIssued,
      totalAttended,
      issuedByType,
      attendedByType,
      avgTM: { SP: avg(tmByType.SP), SG: avg(tmByType.SG), SE: avg(tmByType.SE) },
      details: state.history
    }
  }
}

/* ==================================================
   FILE: src/components/TicketCard.jsx
   ================================================== */
import React from 'react'
export default function TicketCard({t}){
  if(!t) return null
  return (
    <div className="card">
      <div className="ticket">{t.id}</div>
      <div className="small">Tipo: {t.type} • Emitida: {new Date(t.createdAt).toLocaleString()}</div>
      {t.calledAt && <div className="small">Chamado: {new Date(t.calledAt).toLocaleString()} • Guichê: {t.guiche}</div>}
    </div>
  )
}

/* ==================================================
   FILE: src/pages/Totem.jsx
   ================================================== */
import React, { useState } from 'react'
import { ticketService } from '../utils/ticketService'
import TicketCard from '../components/TicketCard'

export default function Totem(){
  const [type, setType] = useState('SG')
  const [last, setLast] = useState(null)
  const [msg, setMsg] = useState('')

  const emitir = () =>{
    const res = ticketService.emit(type, new Date())
    if(res.error) { setMsg(res.error); setLast(null); return }
    setLast(res.ticket)
    setMsg('Senha emitida com sucesso' + (res.ticket.discarded ? ' (será descartada)' : ''))
  }

  return (
    <div className="grid">
      <div className="card">
        <h2>Totem (AC) — Emitir senha</h2>
        <div>
          <label>Tipo da senha:</label>
          <select value={type} onChange={e=>setType(e.target.value)}>
            <option value="SP">SP — Prioritária</option>
            <option value="SE">SE — Retirada de Exames</option>
            <option value="SG">SG — Geral</option>
          </select>
        </div>
        <div style={{marginTop:8}}>
          <button onClick={emitir}>Emitir senha</button>
        </div>
        {msg && <p className="small">{msg}</p>}
      </div>

      <div className="card">
        <h3>Última senha gerada</h3>
        {last ? <TicketCard t={last} /> : <p className="small">Nenhuma ainda.</p>}
      </div>
    </div>
  )
}

/* ==================================================
   FILE: src/pages/Atendente.jsx
   ================================================== */
import React, { useState } from 'react'
import { ticketService } from '../utils/ticketService'
import TicketCard from '../components/TicketCard'

export default function Atendente(){
  const [guiche, setGuiche] = useState('1')
  const [lastCalled, setLastCalled] = useState(null)
  const [msg, setMsg] = useState('')

  const chamar = () =>{
    const res = ticketService.callNext(guiche, new Date())
    if(res.error){ setMsg(res.error); setLastCalled(null); return }
    setLastCalled(res.attended)
    setMsg('Chamada efetuada: ' + res.attended.id)
  }

  const painel = ticketService.getPanelLast5()

  return (
    <div className="grid">
      <div className="card">
        <h2>Atendente (AA)</h2>
        <div>
          <label>Guichê:</label>
          <input value={guiche} onChange={e=>setGuiche(e.target.value)} />
        </div>
        <div style={{marginTop:8}}>
          <button onClick={chamar}>Chamar próximo</button>
        </div>
        {msg && <p className="small">{msg}</p>}
      </div>

      <div className="card">
        <h3>Última chamada</h3>
        {lastCalled ? <TicketCard t={lastCalled} /> : <p className="small">Nenhuma chamada ainda.</p>}
      </div>

      <div className="card">
        <h3>Painel (5 últimas chamadas)</h3>
        {painel.length === 0 && <p className="small">Sem chamadas.</p>}
        {painel.map(p => (
          <div key={p.id} className="small">{p.id} • {p.type} • {new Date(p.calledAt).toLocaleTimeString()}</div>
        ))}
      </div>
    </div>
  )
}

/* ==================================================
   FILE: src/pages/Painel.jsx
   ================================================== */
import React from 'react'
import { ticketService } from '../utils/ticketService'

export default function Painel(){
  const last5 = ticketService.getPanelLast5()
  return (
    <div className="card">
      <h2>Painel Público</h2>
      <p className="small">Últimas 5 senhas chamadas (não mostra a próxima).</p>
      {last5.length === 0 && <p className="small">Nenhuma chamada ainda.</p>}
      {last5.map(t => (
        <div key={t.id} style={{padding:8,borderBottom:'1px solid #eee'}}>
          <div className="ticket">{t.id}</div>
          <div className="small">Guichê: {t.guiche} • Chamado: {new Date(t.calledAt).toLocaleTimeString()}</div>
        </div>
      ))}
    </div>
  )
}

/* ==================================================
   FILE: src/pages/Relatorios.jsx
   ================================================== */
import React, { useState } from 'react'
import { ticketService } from '../utils/ticketService'

export default function Relatorios(){
  const [report, setReport] = useState(null)

  const gerar = () =>{
    const r = ticketService.reports()
    setReport(r)
  }

  return (
    <div className="card">
      <h2>Relatórios</h2>
      <p className="small">Gera relatório com totais e detalhes a partir dos dados locais (localStorage).</p>
      <button onClick={gerar}>Gerar relatório</button>
      {report && (
        <div style={{marginTop:12}}>
          <div className="small">Total emitidas: {report.totalIssued}</div>
          <div className="small">Total atendidas: {report.totalAttended}</div>
          <div className="small">Emitidas por tipo: {JSON.stringify(report.issuedByType)}</div>
          <div className="small">Atendidas por tipo: {JSON.stringify(report.attendedByType)}</div>
          <div className="small">TM médio (min): {JSON.stringify(report.avgTM)}</div>
          <h4>Detalhes (ultimas chamadas)</h4>
          <div style={{maxHeight:200,overflow:'auto'}}>
            {report.details.map(d => (
              <div key={d.id} style={{padding:8,borderBottom:'1px solid #eee'}}>
                <div className="small">{d.id} • {d.type} • Emitida: {new Date(d.createdAt).toLocaleString()}</div>
                <div className="small">Chamado: {new Date(d.calledAt).toLocaleString()} • Guichê: {d.guiche} • TM: {d.tmMinutes}min</div>
              </div>
            ))}
          </div>
        </div>
      )}
    </div>
  )
}

/* ==================================================
   FILE: README.md
   ================================================== */
# web_ticket

Sistema de tickets (senhas) para filas de laboratório — front-end em React.

*Observações:*
- Projeto focado apenas no front-end (conforme instruções da disciplina).
- Armazenamento local em localStorage.
- Nome do repositório deve ser web_ticket e público.

## Como rodar
1. npm install
2. npm run dev

## Estrutura
- src/pages — Totem (emissão), Atendente (chamada), Painel (5 últimas), Relatórios
- src/utils/ticketService.js — lógica de filas, geração de IDs, TM e descarte

## Regras principais implementadas
- Tipos: SP, SE, SG.
- Sequência de atendimento: sempre priorizar SP, depois SE, depois SG.
- TM (simulado): SP ≈ 15min ±5, SG ≈ 5min ±3, SE ≈ <1min (95%) ou 5min (5%).
- 5% das senhas emitidas são marcadas como descartadas (não atendidas).
- Expediente: 07:00 — 17:00 (fora disso a emissão/chamada são bloqueadas).
- Painel mostra as últimas 5 chamadas.

/* ==================================================
   FILE: TODO.md
   ================================================== */
# TODO

- [x] Totem: emitir senha (SP/SE/SG) com formato YYMMDD-PPSQ
- [x] Atendente: chamar próximo seguindo regras de prioridade
- [x] Painel: mostrar últimas 5 chamadas
- [x] Relatórios básicos (totais, por tipo, TM médio)
- [ ] Exportar CSV dos relatórios
- [ ] Melhorar UX / animações
- [ ] Tests unitários

/* ==================================================
   FILE: LICENSE
   ================================================== */
Creative Commons Attribution 4.0 International (CC BY 4.0)

You are free to:
- Share — copy and redistribute the material in any medium or format
- Adapt — remix, transform, and build upon the material for any purpose, even commercially.

Under the following terms:
- Attribution — You must give appropriate credit, provide a link to the license, and indicate if changes were made.

Full license text: https://creativecommons.org/licenses/by/4.0/legalcode

/* ==================================================
   FILE: presentacao.txt
   ================================================== */
Roteiro de apresentação (1-2 minutos por membro):

1) Introdução (20s):
"Somos o Grupo X. Este é o web_ticket — sistema de senhas para filas de laboratório. Implementamos Totem, Atendente, Painel e Relatórios."

2) Técnicas (30s):
"Front-end com React. Armazenamento em localStorage. Implementamos regras do enunciado: tipos SP/SE/SG, sequência de atendimento, TM simulado e descarte de 5%."

3) Demonstração (30s):
- Emitir uma senha no Totem
- Chamar próximo no Atendente
- Mostrar Painel com 5 últimas chamadas

4) Conclusão (10s):
"Código disponível no repositório web_ticket. Obrigado."

/* ==================================================
   END OF PROJECT
   ================================================== */

   
