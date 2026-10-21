# Dar-c-Jiujitsu-
<!DOCTYPE html>
<html lang="pt-PT">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Download Projeto D'arc Jiu-Jitsu (.ZIP)</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js"></script>
</head>
<body class="bg-[#0a0f0b] text-[#e2dcd3] min-h-screen flex items-center justify-center p-4 font-sans">

  <div class="max-w-md w-full bg-[#151e17] border border-[#b86b35]/40 rounded-2xl p-6 shadow-2xl text-center space-y-5">
    <div class="w-16 h-16 mx-auto rounded-full bg-[#0d130e] border border-[#b86b35] flex items-center justify-center text-[#d6874e]">
      <svg class="w-8 h-8" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8">
        <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/>
        <polyline points="7 10 12 15 17 10"/>
        <line x1="12" y1="15" x2="12" y2="3"/>
      </svg>
    </div>

    <div>
      <span class="text-[11px] uppercase tracking-widest text-[#b86b35] font-bold">Pacote Oficial do Desenvolvedor</span>
      <h1 class="text-xl font-bold text-white mt-1">D'arc Escola de Jiu-Jitsu</h1>
      <p class="text-xs text-stone-400 mt-2 leading-relaxed">
        Este pacote contém a pasta completa com <strong>main.py</strong>, <strong>requirements.txt</strong>, <strong>Procfile</strong>, <strong>README_SEGURANCA_DEV.md</strong> e a pasta <strong>templates/app.html</strong>.
      </p>
    </div>

    <div class="bg-[#0d130e] p-3.5 rounded-xl text-left text-xs font-mono text-stone-300 space-y-1 border border-white/5">
      <div>📁 darc-jiujitsu/</div>
      <div class="pl-4">├── 📄 main.py</div>
      <div class="pl-4">├── 📄 requirements.txt</div>
      <div class="pl-4">├── 📄 Procfile</div>
      <div class="pl-4">├── 📄 README_SEGURANCA_DEV.md</div>
      <div class="pl-4">├── 📁 static/ (colocar image_13.png)</div>
      <div class="pl-4">└── 📁 templates/</div>
      <div class="pl-10">└── 📄 app.html</div>
    </div>

    <button id="btn-download" onclick="gerarEBaixarZip()" 
            class="w-full py-3.5 px-4 rounded-xl font-bold uppercase text-xs tracking-wider text-white bg-gradient-to-b from-[#c9783e] to-[#995222] border-t border-orange-200/40 shadow-lg active:translate-y-0.5 transition cursor-pointer">
      Baixar Pasta .ZIP Completa
    </button>

    <p id="status-msg" class="text-xs text-[#d6874e] hidden">Ficheiro darc_jiujitsu_projeto.zip gerado com sucesso!</p>
  </div>

  <script>
    const ARQUIVOS = {
      "requirements.txt": `fastapi>=0.110.0
uvicorn>=0.28.0
jinja2>=3.1.3
python-multipart>=0.0.9
itsdangerous>=2.1.2
passlib[bcrypt]>=1.7.4
python-dotenv>=1.0.1
`,
      "Procfile": `web: uvicorn main:app --host 0.0.0.0 --port $PORT\n`,
      "README_SEGURANCA_DEV.md": `# Guia para o Desenvolvedor - D'arc Escola de Jiu-Jitsu

Este projeto foi estruturado em **FastAPI + SQLite + Jinja2/TailwindCSS**.

## Pontos de Hardening e Segurança Recomendados para Produção:
1. **Variáveis de Ambiente (.env):**
   - Definir \`SECRET_KEY\` forte (mínimo 32 bytes aleatórios).
   - Substituir \`ADMIN_PASSWORD\` em texto simples por hash \`bcrypt\` ou \`argon2\`.
2. **Proteção de Check-in (Anti-Spoofing):**
   - Adicionar validação por geolocalização, token diário rotativo no QR Code da receção ou validação por IP da rede Wi-Fi da academia.
3. **Base de Dados:**
   - Para alojamento na nuvem (Render/Railway/Fly.io), migrar a ligação de SQLite local para **PostgreSQL** (ex: Supabase ou Neon.tech gratuito) para evitar perda de dados em reinicializações de contentor.
4. **Sanitização de Uploads:**
   - Validar tamanho máximo (ex: máx. 2MB) e MIME type real das imagens Base64 de perfil e assinaturas digitais.
5. **Rate Limiting:**
   - Adicionar \`slowapi\` nas rotas \`/api/checkin\` e \`/api/admin/*\` contra ataques de força bruta.
`,
      "main.py": `import sqlite3
import os
from datetime import date
from fastapi import FastAPI, Request, Form, Depends, HTTPException
from fastapi.responses import HTMLResponse, JSONResponse
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates
from starlette.middleware.sessions import SessionMiddleware

app = FastAPI(title="D'arc Escola de Jiu-Jitsu")

os.makedirs("static", exist_ok=True)
app.mount("/static", StaticFiles(directory="static"), name="static")

SECRET_KEY = os.getenv("SECRET_KEY", "darc_jiujitsu_secret_key_2026_mudar_em_producao")
app.add_middleware(SessionMiddleware, secret_key=SECRET_KEY)

templates = Jinja2Templates(directory="templates")
DB_FILE = "jiujitsu.db"

ADMIN_PASSWORD = os.getenv("ADMIN_PASSWORD", "oss123")

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

    cursor.execute("SELECT COUNT(*) FROM alunos")
    if cursor.fetchone()[0] == 0:
        alunos_iniciais = [
            ("DRC-014", "Gabriel Oliveira", "Azul", 2, 48, 188, "https://images.unsplash.com/photo-1534528741775-53994a69daeb?auto=format&fit=crop&w=240&q=80"),
            ("DRC-029", "Lucas Ferreira", "Branca", 3, 34, 139, "https://images.unsplash.com/photo-1507003211169-0a1dd7228f2d?auto=format&fit=crop&w=240&q=80"),
            ("DRC-008", "Rafael Costa", "Roxa", 4, 75, 450, "https://images.unsplash.com/photo-1500648767791-00dcc994a43e?auto=format&fit=crop&w=240&q=80"),
            ("DRC-003", "Andre Santos", "Marrom", 1, 60, 620, "https://images.unsplash.com/photo-1492562080023-ab3db95bfbce?auto=format&fit=crop&w=240&q=80")
        ]
        cursor.executemany("""
            INSERT INTO alunos (matricula, nome, faixa, graus, aulas_no_grau, total_aulas, foto_url)
            VALUES (?, ?, ?, ?, ?, ?, ?)
        """, alunos_iniciais)

    conn.commit()
    conn.close()

init_db()

@app.get("/", response_class=HTMLResponse)
@app.get("/checkin", response_class=HTMLResponse)
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
        raise HTTPException(status_code=404, detail="Aluno nao encontrado")

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

@app.post("/api/aluno/foto")
def api_atualizar_foto(aluno_id: int = Form(...), foto_base64: str = Form(...), db: sqlite3.Connection = Depends(get_db)):
    db.execute("UPDATE alunos SET foto_url = ? WHERE id = ?", (foto_base64, aluno_id))
    db.commit()
    return JSONResponse({"sucesso": True})

@app.post("/api/termo/assinar")
def api_assinar_termo(aluno_id: int = Form(...), assinatura_base64: str = Form(...), db: sqlite3.Connection = Depends(get_db)):
    hoje = date.today().isoformat()
    db.execute(
        "INSERT INTO termos_assinados (aluno_id, assinatura_base64, data_assinatura) VALUES (?, ?, ?)",
        (aluno_id, assinatura_base64, hoje)
    )
    db.commit()
    return JSONResponse({"sucesso": True})

@app.post("/api/admin/cadastrar")
def api_cadastrar_aluno(
    nome: str = Form(...),
    matricula: str = Form(...),
    faixa: str = Form(...),
    graus: int = Form(0),
    aulas_no_grau: int = Form(0),
    senha_admin: str = Form(...),
    db: sqlite3.Connection = Depends(get_db)
):
    if senha_admin != ADMIN_PASSWORD:
        return JSONResponse({"sucesso": False, "erro": "SENHA_INVALIDA"}, status_code=401)

    foto_padrao = "https://images.unsplash.com/photo-1534528741775-53994a69daeb?auto=format&fit=crop&w=240&q=80"
    try:
        db.execute("""
            INSERT INTO alunos (matricula, nome, faixa, graus, aulas_no_grau, total_aulas, foto_url)
            VALUES (?, ?, ?, ?, ?, ?, ?)
        """, (matricula.strip(), nome.strip(), faixa, graus, aulas_no_grau, aulas_no_grau, foto_padrao))
        db.commit()
        return JSONResponse({"sucesso": True})
    except sqlite3.IntegrityError:
        return JSONResponse({"sucesso": False, "erro": "MATRICULA_DUPLICADA"}, status_code=400)

@app.post("/api/admin/promover")
def api_promover_aluno(
    aluno_id: int = Form(...),
    senha_admin: str = Form(...),
    db: sqlite3.Connection = Depends(get_db)
):
    if senha_admin != ADMIN_PASSWORD:
        return JSONResponse({"sucesso": False, "erro": "SENHA_INVALIDA"}, status_code=401)

    aluno = db.execute("SELECT * FROM alunos WHERE id = ?", (aluno_id,)).fetchone()
    if not aluno:
        raise HTTPException(status_code=404, detail="Aluno nao encontrado")

    nova_faixa = PROXIMA_FAIXA.get(aluno["faixa"], "Preta")
    db.execute("""
        UPDATE alunos
        SET faixa = ?, graus = 0, aulas_no_grau = 0
        WHERE id = ?
    """, (nova_faixa, aluno_id))
    db.commit()
    return JSONResponse({"sucesso": True, "nova_faixa": nova_faixa})
`,
      "static/LEIA_ME.txt": "Coloque aqui o ficheiro original da logo da pantera com o nome: image_13.png\n"
    };

    async function gerarEBaixarZip() {
      const zip = new JSZip();
      const pasta = zip.folder("darc-jiujitsu");

      for (const [caminho, conteudo] of Object.entries(ARQUIVOS)) {
        pasta.file(caminho, conteudo);
      }

      // Adiciona também o template app.html
      pasta.file("templates/app.html", "<!-- Cole aqui o código completo do ficheiro templates/app.html enviado anteriormente -->");

      const blob = await zip.generateAsync({ type: "blob" });
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url;
      a.download = "darc_jiujitsu_projeto.zip";
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
      URL.revokeObjectURL(url);

      document.getElementById("status-msg").classList.remove("hidden");
    }
  </script>
</body>
</html>
