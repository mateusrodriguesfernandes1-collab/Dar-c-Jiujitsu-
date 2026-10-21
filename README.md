"""
====================================================================
D'ARC ESCOLA DE JIU-JITSU - APLICATIVO WEB COMPLETO (PYTHON + SQLITE)
====================================================================
Como usar no GitHub / Localmente:
1. Instale as dependências: pip install fastapi uvicorn jinja2 python-multipart
2. Execute o arquivo: python app_completo.py
3. Acesse no navegador: http://127.0.0.1:8000
====================================================================
"""

import sqlite3
import os
from datetime import date
from fastapi import FastAPI, Request, Form, Depends, HTTPException
from fastapi.responses import HTMLResponse, JSONResponse
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates
import uvicorn

app = FastAPI(title="D'arc Escola de Jiu-Jitsu")

# Cria pastas necessárias para assets e templates se não existirem
os.makedirs("static", exist_ok=True)
os.makedirs("templates", exist_ok=True)

app.mount("/static", StaticFiles(directory="static"), name="static")

DB_FILE = "jiujitsu.db"
ADMIN_PASSWORD = os.getenv("ADMIN_PASSWORD", "oss123")

# Parâmetros Oficiais IBJJF por Aulas
REGRAS_AULAS = {
    "Branca": 35,
    "Azul": 65,
    "Roxa": 75,
    "Marrom": 85,
    "Preta": 0
}

PROXIMA_FAIXA = {
    "Branca": "Azul",
    "Azul": "Roxa",
    "Roxa": "Marrom",
    "Marrom": "Preta",
    "Preta": "Coral"
}

def get_db():
    conn = sqlite3.connect(DB_FILE)
    conn.row_factory = sqlite3.Row
    try:
        yield conn
    finally:
        conn.close()

def init_db():
    conn = sqlite3.connect(DB_FILE)
    cursor = conn.cursor()

    cursor.execute("""
    CREATE TABLE IF NOT EXISTS alunos (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        matricula TEXT UNIQUE NOT NULL,
        nome TEXT NOT NULL,
        faixa TEXT NOT NULL DEFAULT 'Branca',
        graus INTEGER NOT NULL DEFAULT 0,
        aulas_no_grau INTEGER NOT NULL DEFAULT 0,
        total_aulas INTEGER NOT NULL DEFAULT 0,
        foto_url TEXT NOT NULL
    )
    """)

    cursor.execute("""
    CREATE TABLE IF NOT EXISTS presencas (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        aluno_id INTEGER NOT NULL,
        data_aula TEXT NOT NULL,
        FOREIGN KEY (aluno_id) REFERENCES alunos (id),
        UNIQUE(aluno_id, data_aula)
    )
    """)

    cursor.execute("""
    CREATE TABLE IF NOT EXISTS termos_assinados (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        aluno_id INTEGER NOT NULL,
        assinatura_base64 TEXT NOT NULL,
        data_assinatura TEXT NOT NULL,
        FOREIGN KEY (aluno_id) REFERENCES alunos (id)
    )
    """)

    # Alunos de teste iniciais
    cursor.execute("SELECT COUNT(*) FROM alunos")
    if cursor.fetchone()[0] == 0:
        alunos_iniciais = [
            ("DRC-014", "Gabriel Oliveira", "Azul", 2, 48, 188, "https://images.unsplash.com/photo-1534528741775-53994a69daeb?auto=format&fit=crop&w=240&q=80"),
            ("DRC-029", "Lucas Ferreira", "Branca", 3, 34, 139, "https://images.unsplash.com/photo-1507003211169-0a1dd7228f2d?auto=format&fit=crop&w=240&q=80"),
            ("DRC-008", "Rafael Costa", "Roxa", 4, 75, 450, "https://images.unsplash.com/photo-1500648767791-00dcc994a43e?auto=format&fit=crop&w=240&q=80"),
            ("DRC-003", "André Santos", "Marrom", 1, 60, 620, "https://images.unsplash.com/photo-1492562080023-ab3db95bfbce?auto=format&fit=crop&w=240&q=80")
        ]
        cursor.executemany("""
            INSERT INTO alunos (matricula, nome, faixa, graus, aulas_no_grau, total_aulas, foto_url)
            VALUES (?, ?, ?, ?, ?, ?, ?)
        """, alunos_iniciais)

    conn.commit()
    conn.close()

init_db()

# Criação automática do HTML de template integrado para facilitar o deploy no GitHub
TEMPLATE_HTML = """<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>D'arc Escola de Jiu-Jitsu</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Cinzel:wght@600;700;800&family=Inter:wght@300;400;500;600;700&display=swap" rel="stylesheet">
  <style>
    :root {
      --bg-base: #0a0f0b;
      --surface-base: #151e17;
      --surface-raised: #1c281f;
      --surface-inset: #0d130e;
      --accent-wood: #b86b35;
      --accent-wood-glow: rgba(184, 107, 53, 0.38);
      --accent-moss: #2d4532;
      --text-sand: #e2dcd3;
      --text-muted: #8c968e;
      --forest-overlay: linear-gradient(180deg, rgba(10, 18, 12, 0.74) 0%, rgba(20, 35, 23, 0.88) 55%, rgba(7, 11, 8, 0.96) 100%);
    }
    [data-theme="moss"] { --surface-inset: #0b110c; --accent-wood: #b86b35; }
    [data-theme="graphite"] { --surface-inset: #0d1114; --accent-wood: #a96638; }
    [data-theme="sand"] { --surface-inset: #120e0b; --accent-wood: #c87a41; }
    * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
    body { font-family: 'Inter', sans-serif; background-color: #050705; color: var(--text-sand); margin: 0; padding: 0; display: flex; justify-content: center; align-items: center; min-height: 100vh; }
    .font-cinzel { font-family: 'Cinzel', serif; }
    .app-shell { width: 100%; max-width: 430px; min-height: 100vh; max-height: 932px; position: relative; overflow-x: hidden; display: flex; flex-direction: column; background-image: var(--forest-overlay), url('https://images.unsplash.com/photo-1511497584788-876760111969?auto=format&fit=crop&w=1000&q=85'); background-size: cover; background-position: center; box-shadow: 0 25px 70px rgba(0, 0, 0, 0.95); }
    @media (min-width: 460px) { .app-shell { min-height: 860px; height: 92vh; border-radius: 38px; } }
    .surface-raised { background: linear-gradient(165deg, rgba(34, 48, 37, 0.88) 0%, rgba(18, 26, 20, 0.94) 100%); backdrop-filter: blur(14px); border: 1px solid rgba(255, 255, 255, 0.1); border-radius: 18px; box-shadow: 6px 8px 20px rgba(0, 0, 0, 0.65); }
    .surface-inset { background: var(--surface-inset); border: 1px solid rgba(0, 0, 0, 0.85); border-radius: 12px; box-shadow: inset 3px 4px 8px rgba(0, 0, 0, 0.85); }
    .btn-primary { background: linear-gradient(180deg, #c9783e 0%, #995222 100%); color: #f7f3eb; border-radius: 11px; font-weight: 600; text-transform: uppercase; font-size: 0.74rem; cursor: pointer; display: inline-flex; align-items: center; justify-content: center; gap: 8px; }
    .theme-pill { padding: 5px 12px; border-radius: 9999px; font-size: 0.68rem; font-weight: 600; border: 1px solid rgba(255, 255, 255, 0.1); background: rgba(15, 22, 16, 0.75); color: var(--text-muted); cursor: pointer; }
    .theme-pill.active-pill { background: #b8aa96; color: #121713; }
    .nav-tab { border-radius: 10px; color: var(--text-muted); font-size: 0.66rem; font-weight: 600; padding: 8px 4px; display: flex; flex-direction: column; align-items: center; cursor: pointer; }
    .nav-tab.active { background: linear-gradient(180deg, #b86b35 0%, #8a4b1f 100%); color: #fff; }
    .step-node { width: 28px; height: 28px; border-radius: 9999px; background: #0d130e; border: 2px solid #2b392d; display: flex; align-items: center; justify-content: center; z-index: 2; }
    .step-node.completed { background: radial-gradient(circle, #d6874e 0%, #8a4b1f 100%); border-color: #f0ab75; }
    .step-line { flex: 1; height: 3px; background: #141d16; margin: 0 -2px; z-index: 1; }
    .step-line.completed { background: linear-gradient(90deg, #8a4b1f, #d6874e); }
    .student-avatar-ring { width: 74px; height: 74px; border-radius: 9999px; padding: 3px; background: linear-gradient(145deg, #c9783e, #243527); cursor: pointer; flex-shrink: 0; }
    .student-avatar-inner { width: 100%; height: 100%; border-radius: 9999px; background: #0d130e; overflow: hidden; }
    .no-scrollbar::-webkit-scrollbar { display: none; }
  </style>
</head>
<body data-theme="moss">
  <div class="app-shell">
    <header class="px-4 pt-4 pb-3 border-b border-white/10 bg-black/50 backdrop-blur-md sticky top-0 z-30 space-y-3">
      <div class="flex items-center justify-between">
        <div class="flex items-center gap-3">
          <div class="w-10 h-10 rounded-full border border-[#b86b35] flex items-center justify-center bg-[#131b14]">🐆</div>
          <div>
            <h1 class="font-cinzel text-lg font-bold tracking-[0.18em] text-[#e8dfd1] leading-none">D'ARC</h1>
            <p class="text-[9px] uppercase tracking-[0.2em] text-[#b86b35] mt-1 font-semibold">Escola de Jiu-Jitsu</p>
          </div>
        </div>
        <div class="flex items-center gap-1.5">
          <button onclick="mudarPaleta('moss', this)" class="theme-pill active-pill">musgo</button>
          <button onclick="mudarPaleta('graphite', this)" class="theme-pill">grafite</button>
          <button onclick="mudarPaleta('sand', this)" class="theme-pill">areia</button>
        </div>
      </div>
    </header>

    <main class="flex-1 overflow-y-auto no-scrollbar px-4 py-4 space-y-4">
      <section id="view-aluno" class="space-y-4">
        <div class="surface-raised p-4 flex items-center justify-between gap-3">
          <div class="flex items-center gap-3.5">
            <div class="student-avatar-ring">
              <div class="student-avatar-inner">
                <img id="img-avatar-aluno" src="" alt="Aluno" class="w-full h-full object-cover">
              </div>
            </div>
            <div>
              <h2 id="display-nome-aluno" class="text-base font-bold text-[#f2ece4]"></h2>
              <p id="display-matricula" class="text-[11px] text-[#8c968e] mt-0.5"></p>
              <div class="flex items-center gap-2 mt-2">
                <span class="text-xs text-[#8c968e]">Faixa:</span>
                <strong id="display-faixa-aluno" class="text-xs font-bold text-[#f2ece4]"></strong>
              </div>
            </div>
          </div>
          <select id="select-aluno-ativo" onchange="trocarAlunoAtivo()" class="surface-inset text-[11px] text-[#e2dcd3] px-2 py-1.5"></select>
        </div>

        <div class="surface-raised p-4 space-y-4">
          <div class="flex items-center justify-between">
            <h3 class="text-sm font-semibold text-[#e8dfd1]">Progresso de Graduação</h3>
            <span id="display-meta-aulas" class="text-[11px] font-bold text-[#d6874e]"></span>
          </div>
          <div class="px-2 pt-1">
            <div class="flex items-center" id="stepper-graus"></div>
            <div class="grid grid-cols-4 text-center text-[10px] text-[#8c968e] mt-2">
              <span>1º Grau</span><span>2º Grau</span><span>3º Grau</span><span>4º Grau</span>
            </div>
          </div>
          <div class="surface-inset p-3 space-y-2">
            <div class="flex justify-between text-[11px]">
              <span class="text-[#8c968e]">Frequência no grau:</span>
              <span id="display-status-exame" class="text-[#d6874e] font-semibold"></span>
            </div>
            <div class="w-full h-2 rounded-full bg-black/80 overflow-hidden">
              <div id="barra-progresso-grau" class="h-full rounded-full transition-all duration-300" style="background: linear-gradient(90deg, #2d4532 0%, #b86b35 100%);"></div>
            </div>
          </div>
          <button onclick="registrarPresencaHoje()" class="btn-primary w-full py-3">Confirmar Presença</button>
        </div>
        <div id="toast-feedback" class="hidden surface-raised p-3.5 border-l-4 border-l-[#b86b35]">
          <span id="toast-texto" class="text-xs text-[#e2dcd3]"></span>
        </div>
      </section>

      <section id="view-planos" class="hidden space-y-4">
        <div class="surface-raised p-4">
          <h2 class="font-cinzel text-base font-bold text-white">Planos de Aulas e Critérios IBJJF</h2>
          <p class="text-xs text-[#8c968e] mt-1">Branca: 35 | Azul: 65 | Roxa: 75 | Marrom: 85 aulas por grau.</p>
        </div>
      </section>

      <section id="view-loja" class="hidden space-y-4">
        <div class="surface-raised p-4">
          <h2 class="font-cinzel text-base font-bold text-white">Loja Oficial D'arc</h2>
          <p class="text-xs text-[#8c968e] mt-1">Kimonos, Rash Guards e Faixas oficiais.</p>
        </div>
      </section>

      <section id="view-termo" class="hidden space-y-4">
        <div class="surface-raised p-4 space-y-3">
          <h2 class="font-cinzel text-base font-bold text-white">Termo com Assinatura Digital</h2>
          <div class="surface-inset p-1"><canvas id="canvas-assinatura" class="w-full h-32 touch-none"></canvas></div>
          <button onclick="confirmarAssinatura()" class="btn-primary w-full py-3">Salvar Assinatura</button>
        </div>
      </section>

      <section id="view-admin" class="hidden space-y-4">
        <div class="surface-raised p-4 space-y-3">
          <h2 class="font-cinzel text-sm font-bold text-white">Painel do Mestre</h2>
          <div id="lista-admin-alunos" class="space-y-2"></div>
        </div>
      </section>
    </main>

    <footer class="p-3 bg-black/80 backdrop-blur-md border-t border-white/10 sticky bottom-0 z-30">
      <nav class="nav-pill-container grid grid-cols-5 gap-1">
        <button onclick="navegarPara('aluno')" id="tab-aluno" class="nav-tab active"><span>Aluno</span></button>
        <button onclick="navegarPara('planos')" id="tab-planos" class="nav-tab"><span>Aulas</span></button>
        <button onclick="navegarPara('loja')" id="tab-loja" class="nav-tab"><span>Loja</span></button>
        <button onclick="navegarPara('termo')" id="tab-termo" class="nav-tab"><span>Termos</span></button>
        <button onclick="navegarPara('admin')" id="tab-admin" class="nav-tab"><span>Mestre</span></button>
      </nav>
    </footer>
  </div>

  <script>
    const META_POR_FAIXA = { "Branca": 35, "Azul": 65, "Roxa": 75, "Marrom": 85 };
    const PROXIMA_FAIXA = { "Branca": "Azul", "Azul": "Roxa", "Roxa": "Marrom", "Marrom": "Preta" };
    let alunos = [];
    let indiceAtivo = 0;

    async function inicializarDados() {
      const resp = await fetch('/api/dados');
      const data = await resp.json();
      alunos = data.alunos;
      const sel = document.getElementById('select-aluno-ativo');
      sel.innerHTML = '';
      alunos.forEach((a, idx) => {
        const opt = document.createElement('option');
        opt.value = idx;
        opt.textContent = a.nome.split(' ')[0];
        sel.appendChild(opt);
      });
      renderizarAlunoAtivo();
    }

    function renderizarAlunoAtivo() {
      const a = alunos[indiceAtivo];
      if (!a) return;
      const meta = META_POR_FAIXA[a.faixa] || 35;
      const pct = Math.min(100, Math.round((a.aulas_no_grau / meta) * 100));
      const restantes = Math.max(0, meta - a.aulas_no_grau);
      document.getElementById('display-nome-aluno').textContent = a.nome;
      document.getElementById('display-matricula').textContent = 'ID: ' + a.matricula;
      document.getElementById('img-avatar-aluno').src = a.foto_url;
      document.getElementById('display-faixa-aluno').textContent = 'Faixa ' + a.faixa;
      document.getElementById('display-meta-aulas').textContent = a.aulas_no_grau + ' / ' + meta;
      document.getElementById('barra-progresso-grau').style.width = pct + '%';
      document.getElementById('display-status-exame').textContent = a.graus >= 4
        ? 'Apto Exame Faixa ' + PROXIMA_FAIXA[a.faixa]
        : 'Faltam ' + restantes + ' aulas p/ ' + (a.graus + 1) + 'º Grau';

      const stepper = document.getElementById('stepper-graus');
      stepper.innerHTML = '';
      for (let i = 1; i <= 4; i++) {
        const completo = i <= a.graus;
        const node = document.createElement('div');
        node.className = 'step-node ' + (completo ? 'completed' : '');
        stepper.appendChild(node);
        if (i < 4) {
          const line = document.createElement('div');
          line.className = 'step-line ' + (i < a.graus ? 'completed' : '');
          stepper.appendChild(line);
        }
      }
    }

    function trocarAlunoAtivo() {
      indiceAtivo = parseInt(document.getElementById('select-aluno-ativo').value, 10);
      renderizarAlunoAtivo();
    }

    async function registrarPresencaHoje() {
      const a = alunos[indiceAtivo];
      const form = new FormData();
      form.append('aluno_id', a.id);
      const resp = await fetch('/api/checkin', { method: 'POST', body: form });
      const res = await resp.json();
      const box = document.getElementById('toast-feedback');
      document.getElementById('toast-texto').textContent = res.sucesso ? 'Presença confirmada! Oss!' : 'Presença já registrada hoje.';
      box.classList.remove('hidden');
      box.classList.add('flex');
      await inicializarDados();
    }

    function mudarPaleta(tema, btnEl) {
      document.body.setAttribute('data-theme', tema);
      btnEl.parentElement.querySelectorAll('.theme-pill').forEach(b => b.classList.remove('active-pill'));
      btnEl.classList.add('active-pill');
    }

    function navegarPara(aba) {
      ['aluno', 'planos', 'loja', 'termo', 'admin'].forEach(id => {
        document.getElementById('view-' + id).classList.add('hidden');
        document.getElementById('tab-' + id).classList.remove('active');
      });
      document.getElementById('view-' + aba).classList.remove('hidden');
      document.getElementById('tab-' + aba).classList.add('active');
    }

    inicializarDados();
  </script>
</body>
</html>
"""

# Escreve o template automaticamente no disco ao iniciar o app
with open(os.path.join("templates", "app.html"), "w", encoding="utf-8") as f:
    f.write(TEMPLATE_HTML)

# Rotas do FastAPI
@app.get("/", response_class=HTMLResponse)
def carregar_app(request: Request):
    return templates.TemplateResponse("app.html", {"request": request})

@app.get("/api/dados")
def obter_dados(db: sqlite3.Connection = Depends(get_db)):
    hoje = date.today().isoformat()
    alunos_rows = db.execute("SELECT * FROM alunos ORDER BY nome ASC").fetchall()
    presencas_hoje = {
        r["aluno_id"] for r in db.execute("SELECT aluno_id FROM presencas WHERE data_aula = ?", (hoje,)).fetchall()
    }
    termos_rows = {
        r["aluno_id"] for r in db.execute("SELECT aluno_id FROM termos_assinados").fetchall()
    }

    alunos = []
    for a in alunos_rows:
        item = dict(a)
        item["presenteHoje"] = a["id"] in presencas_hoje
        item["termoAssinado"] = a["id"] in termos_rows
        alunos.append(item)

    return JSONResponse({"alunos": alunos, "regras": REGRAS_AULAS, "hoje": hoje})

@app.post("/api/checkin")
def api_registrar_presenca(aluno_id: int = Form(...), db: sqlite3.Connection = Depends(get_db)):
    hoje = date.today().isoformat()
    aluno = db.execute("SELECT * FROM alunos WHERE id = ?", (aluno_id,)).fetchone()
    if not aluno:
        raise HTTPException(status_code=404, detail="Aluno não encontrado")

    ja_presente = db.execute(
        "SELECT id FROM presencas WHERE aluno_id = ? AND data_aula = ?", (aluno_id, hoje)
    ).fetchone()

    if ja_presente:
        return JSONResponse({"sucesso": False, "codigo": "JA_REGISTRADO"})

    db.execute("INSERT INTO presencas (aluno_id, data_aula) VALUES (?, ?)", (aluno_id, hoje))

    novas_aulas = aluno["aulas_no_grau"] + 1
    total_aulas = aluno["total_aulas"] + 1
    graus = aluno["graus"]
    faixa = aluno["faixa"]
    meta = REGRAS_AULAS.get(faixa, 35)
    subiu_grau = False

    if meta > 0 and novas_aulas >= meta and graus < 4:
        graus += 1
        novas_aulas = 0
        subiu_grau = True

    db.execute("""
        UPDATE alunos
        SET aulas_no_grau = ?, total_aulas = ?, graus = ?
        WHERE id = ?
    """, (novas_aulas, total_aulas, graus, aluno_id))
    db.commit()

    return JSONResponse({"sucesso": True, "subiu_grau": subiu_grau, "novo_grau": graus})

if __name__ == "__main__":
    uvicorn.run("app_completo:app", host="127.0.0.1", port=8000, reload=True)

