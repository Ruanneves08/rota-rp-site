base = "/mnt/data/rota_rp_site"
os.makedirs(base + "/public", exist_ok=True)

files = {
"package.json": r'''{
  "name": "rota-rp-registros",
  "version": "1.0.0",
  "description": "Painel de registros para roleplay policial com integração Discord",
  "main": "server.js",
  "scripts": { "start": "node server.js" },
  "engines": { "node": ">=20" },
  "dependencies": {
    "better-sqlite3": "^11.8.1",
    "express": "^4.21.2"
  }
}''',
"server.js": r'''const express = require("express");
const Database = require("better-sqlite3");
const path = require("path");
const crypto = require("crypto");

const app = express();
app.use(express.json({ limit: "100kb" }));
app.use(express.static(path.join(__dirname, "public")));

const PORT = process.env.PORT || 3000;
const ADMIN_PASSWORD = process.env.ADMIN_PASSWORD;
const DISCORD_WEBHOOK_URL = process.env.DISCORD_WEBHOOK_URL || "";
const DB_PATH = process.env.DB_PATH || path.join(__dirname, "registros.db");

if (!ADMIN_PASSWORD) {
  console.warn("ATENÇÃO: defina ADMIN_PASSWORD nas variáveis de ambiente antes de publicar.");
}

const db = new Database(DB_PATH);
db.pragma("journal_mode = WAL");
db.exec(`
  CREATE TABLE IF NOT EXISTS records (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    type TEXT NOT NULL,
    protocol TEXT NOT NULL,
    officer TEXT NOT NULL,
    individual TEXT NOT NULL,
    title TEXT NOT NULL,
    details TEXT NOT NULL,
    location TEXT DEFAULT '',
    created_at TEXT NOT NULL
  );
`);

const sessions = new Map();
const allowedTypes = new Set(["Apreensão", "Boletim de Ocorrência", "Ocorrência"]);

function auth(req, res, next) {
  const token = req.headers.authorization?.replace(/^Bearer\s+/i, "");
  if (!token || !sessions.has(token)) return res.status(401).json({ error: "Faça login para continuar." });
  next();
}

app.post("/api/login", (req, res) => {
  const password = String(req.body?.password || "");
  if (!ADMIN_PASSWORD || password !== ADMIN_PASSWORD) {
    return res.status(401).json({ error: "Senha incorreta ou senha do administrador não configurada." });
  }
  const token = crypto.randomBytes(32).toString("hex");
  sessions.set(token, Date.now());
  res.json({ token });
});

app.post("/api/logout", auth, (req, res) => {
  const token = req.headers.authorization.replace(/^Bearer\s+/i, "");
  sessions.delete(token);
  res.json({ ok: true });
});

app.get("/api/records", auth, (req, res) => {
  const rows = db.prepare("SELECT * FROM records ORDER BY id DESC LIMIT 300").all();
  res.json(rows);
});

app.post("/api/records", auth, async (req, res) => {
  const b = req.body || {};
  const type = String(b.type || "");
  const officer = String(b.officer || "").trim();
  const individual = String(b.individual || "").trim();
  const title = String(b.title || "").trim();
  const details = String(b.details || "").trim();
  const location = String(b.location || "").trim();

  if (!allowedTypes.has(type) || !officer || !individual || !title || !details) {
    return res.status(400).json({ error: "Preencha os campos obrigatórios corretamente." });
  }
  if ([officer, individual, title, details, location].some(v => v.length > 4000)) {
    return res.status(400).json({ error: "Um dos campos é muito longo." });
  }

  const protocol = RP-${new Date().getFullYear()}-${crypto.randomBytes(3).toString("hex").toUpperCase()};
  const createdAt = new Date().toISOString();
  const result = db.prepare(`
    INSERT INTO records (type, protocol, officer, individual, title, details, location, created_at)
    VALUES (?, ?, ?, ?, ?, ?, ?, ?)
  `).run(type, protocol, officer, individual, title, details, location, createdAt);

  const record = db.prepare("SELECT * FROM records WHERE id = ?").get(result.lastInsertRowid);

  // Integração simples: publica cada registro em um canal do Discord por webhook.
  if (DISCORD_WEBHOOK_URL) {
    try {
      const discordMessage = {
        username: "Sistema ROTA RP",
        embeds: [{
          title: 📋 ${record.type} — ${record.protocol},
          color: record.type === "Apreensão" ? 15158332 : 3447003,
          fields: [
            { name: "Responsável", value: record.officer.slice(0, 1024), inline: true },
            { name: "Indivíduo", value: record.individual.slice(0, 1024), inline: true },
            { name: "Título", value: record.title.slice(0, 1024) },
            { name: "Local", value: (record.location || "Não informado").slice(0, 1024) },
            { name: "Relato", value: record.details.slice(0, 1024) }
          ],
          footer: { text: "Roleplay fictício • Sistema de registros" },
          timestamp: record.created_at
        }]
      };
      const response = await fetch(DISCORD_WEBHOOK_URL, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(discordMessage)
      });
      if (!response.ok) console.error("Discord webhook respondeu:", response.status);
    } catch (e) {
      console.error("Não foi possível publicar no Discord:", e.message);
    }
  }
  res.status(201).json(record);
});

app.get("*", (req, res) => res.sendFile(path.join(__dirname, "public", "index.html")));

app.listen(PORT, () => console.log(ROTA RP iniciado na porta ${PORT}));
''',
"public/index.html": r'''<!doctype html>
<html lang="pt-BR">
<head>
<meta charset="utf-8"><meta name="viewport" content="width=device-width, initial-scale=1">
<title>ROTA • Sistema RP</title>
<style>
:root{color-scheme:dark;--bg:#0b0e13;--panel:#141a23;--line:#273140;--text:#f3f5f7;--muted:#9ba8b8;--red:#c62828;--red2:#ef4444}
*{box-sizing:border-box}body{margin:0;background:radial-gradient(circle at top,#1a202b,var(--bg) 55%);font-family:Arial,sans-serif;color:var(--text)}
header{padding:22px 5vw;border-bottom:1px solid var(--line);display:flex;align-items:center;gap:14px;background:#0b0e13dd;position:sticky;top:0;backdrop-filter:blur(10px);z-index:2}
.crest{background:var(--red);border-radius:10px;padding:12px;font-weight:900;letter-spacing:1px}.brand h1{font-size:19px;margin:0}.brand p{margin:4px 0 0;color:var(--muted);font-size:12px}
main{max-width:1120px;margin:28px auto;padding:0 18px}.hero{padding:26px 0 18px}.eyebrow{color:#ff7676;text-transform:uppercase;letter-spacing:2px;font-size:11px;font-weight:bold}.hero h2{font-size:clamp(27px,5vw,42px);margin:10px 0}.hero p{color:var(--muted);max-width:680px;line-height:1.6}
.grid{display:grid;grid-template-columns: minmax(0,1fr) minmax(0,1fr);gap:18px}.panel{background:linear-gradient(145deg,#171e29,#11161e);border:1px solid var(--line);border-radius:16px;padding:20px;min-width:0}.panel h3{margin:0 0 17px;font-size:17px}
label{display:block;font-size:12px;color:#c4ccd6;margin:12px 0 6px}input,select,textarea{width:100%;padding:12px;background:#0b1017;border:1px solid #303b4b;border-radius:9px;color:var(--text);font:inherit;outline:none}input:focus,select:focus,textarea:focus{border-color:#e34a4a}textarea{min-height:100px;resize:vertical}
button{border:0;border-radius:9px;padding:12px 15px;background:var(--red);color:white;font-weight:bold;cursor:pointer}button:hover{background:var(--red2)}button.secondary{background:#293342}button:disabled{opacity:.5;cursor:not-allowed}.row{display:grid;grid-template-columns:1fr 1fr;gap:12px}.actions{display:flex;gap:9px;flex-wrap:wrap;margin-top:16px}.muted{color:var(--muted);font-size:12px;line-height:1.5}.status{margin-top:12px;font-size:13px;line-height:1.5}.record{border:1px solid var(--line);background:#0d1219;border-radius:10px;padding:13px;margin-top:10px}.record strong{display:block;margin-bottom:5px}.record small{color:var(--muted)}.record p{white-space:pre-wrap;overflow-wrap:anywhere;color:#cbd3dd;font-size:13px;line-height:1.5}.tag{display:inline-block;font-size:10px;padding:4px 7px;border-radius:5px;background:#402020;color:#ffb3b3;margin-bottom:7px}.hidden{display:none!important}.login{max-width:440px;margin:20px auto}.footer{text-align:center;color:#667384;font-size:11px;padding:35px 15px}
@media(max-width:760px){.grid{grid-template-columns:1fr}.row{grid-template-columns:1fr}header{padding:16px}.panel{padding:16px}}
</style>
</head>
<body>
<header><div class="crest">ROTA</div><div class="brand"><h1>SISTEMA OPERACIONAL RP</h1><p>Rondas Ostensivas Tobias de Aguiar • Ambiente de roleplay</p></div></header>
<main>
<section class="hero"><div class="eyebrow">Sistema de registros digitais</div><h2>Central de registros policiais</h2><p>Registre apreensões, boletins de ocorrência e ocorrências do seu servidor RP. Os registros são armazenados no sistema e podem ser enviados automaticamente para um canal do Discord.</p></section>
<section id="loginPanel" class="panel login"><h3>🔐 Acesso administrativo</h3><p class="muted">O site pode ser acessado publicamente, mas para consultar ou criar registros é necessário entrar com a senha administrativa.</p><label>Senha administrativa</label><input id="password" type="password" placeholder="Digite a senha configurada pelo responsável"><div class="actions"><button id="loginBtn">Entrar no sistema</button></div><div id="loginStatus" class="status"></div></section>
<section id="appPanel" class="hidden">
<div class="actions" style="margin:0 0 16px"><button class="secondary" id="refreshBtn">Atualizar registros</button><button class="secondary" id="logoutBtn">Sair</button></div>
<div class="grid">
<div class="panel"><h3>📝 Novo registro</h3>
<form id="recordForm">
<label>Tipo de registro *</label><select id="type"><option>Apreensão</option><option>Boletim de Ocorrência</option><option>Ocorrência</option></select>
<div class="row"><div><label>Policial responsável / QRA *</label><input id="officer" required maxlength="180" placeholder="Ex.: QRA do policial"></div><div><label>Indivíduo / envolvido *</label><input id="individual" required maxlength="180" placeholder="Nome ou QRA no RP"></div></div>
<label>Título / motivo *</label><input id="title" required maxlength="180" placeholder="Ex.: Apreensão de veículo">
<label>Local da ocorrência</label><input id="location" maxlength="180" placeholder="Ex.: Centro da cidade">
<label>Descrição detalhada *</label><textarea id="details" required maxlength="4000" placeholder="Descreva os fatos dentro do contexto do RP..."></textarea>
<div class="actions"><button type="submit" id="submitBtn">Salvar registro</button></div><div id="formStatus" class="status"></div>
</form></div>
<div class="panel"><h3>📂 Registros recentes</h3><p class="muted">Os registros mais recentes aparecem primeiro. O sistema exibe até 300 registros.</p><div id="records"><p class="muted">Carregando...</p></div></div>
</div></section>
</main><div class="footer">Sistema independente para roleplay fictício. Não é um sistema oficial de órgão público.</div>
<script>
let token = sessionStorage.getItem("rotaToken") || "";
const $ = id => document.getElementById(id);
function showApp(){ $("loginPanel").classList.toggle("hidden",!!token); $("appPanel").classList.toggle("hidden",!token); if(token) loadRecords(); }
async function api(url, options={}) {
  const headers = {...(options.headers||{}), "Content-Type":"application/json"};
  if(token) headers.Authorization = "Bearer " + token;
  const r = await fetch(url,{...options,headers});
  const data = await r.json().catch(()=>({}));
  if(!r.ok) throw new Error(data.error || "Erro na solicitação.");
  return data;
}
$("loginBtn").onclick = async () => {
  $("loginStatus").textContent = "Verificando...";
  try { const d=await api("/api/login",{method:"POST",body:JSON.stringify({password:$("password").value})}); token=d.token;sessionStorage.setItem("rotaToken",token);$("loginStatus").textContent="";showApp(); }
  catch(e){$("loginStatus").textContent=e.message;}
};
$("logoutBtn").onclick = async () => { try{await api("/api/logout",{method:"POST"});}catch(e){} token="";sessionStorage.removeItem("rotaToken");showApp(); };
$("refreshBtn").onclick = loadRecords;
$("recordForm").onsubmit = async e => {
  e.preventDefault();$("formStatus").textContent="Salvando...";$("submitBtn").disabled=true;
  const body={type:$("type").value,officer:$("officer").value,individual:$("individual").value,title:$("title").value,location:$("location").value,details:$("details").value};
  try {const r=await api("/api/records",{method:"POST",body:JSON.stringify(body)});$("formStatus").textContent="Registro salvo! Protocolo: "+r.protocol+".";$("recordForm").reset();$("type").value="Apreensão";await loadRecords();}
  catch(e){$("formStatus").textContent=e.message;}
  finally{$("submitBtn").disabled=false;}
};
async function loadRecords(){
  $("records").innerHTML='<p class="muted">Carregando registros...</p>';
  try{const data=await api("/api/records");$("records").innerHTML="";
    if(!data.length){$("records").innerHTML='<p class="muted">Nenhum registro cadastrado ainda.</p>';return;}
    for(const r of data){const el=document.createElement("article");el.className="record";
      const tag=document.createElement("span");tag.className="tag";tag.textContent=r.type;el.append(tag);
      const title=document.createElement("strong");title.textContent=r.title;el.append(title);
      const meta=document.createElement("small");meta.textContent=${r.protocol} • ${r.officer} • ${new Date(r.created_at).toLocaleString("pt-BR")};el.append(meta);
      const person=document.createElement("p");person.textContent=Indivíduo: ${r.individual}\nLocal: ${r.location||"Não informado"}\n${r.details};el.append(person);$("records").append(el);
    }
  }catch(e){$("records").innerHTML="";const p=document.createElement("p");p.className="muted";p.textContent=e.message;if(e.message.includes("login")){token="";sessionStorage.removeItem("rotaToken");showApp();}else $("records").append(p);}
}
showApp();
</script></body></html>''',
"README.md": r'''# Sistema ROTA RP — site + Discord

Projeto inicial para **roleplay fictício**. Inclui:
- Site responsivo para registrar apreensões, boletins de ocorrência e ocorrências.
- Banco SQLite para salvar registros.
- Área administrativa protegida por senha.
- Envio automático de novos registros para um canal do Discord por webhook.

## 1. Testar no computador

É necessário instalar Node.js 20 ou superior: https://nodejs.org/

1. Extraia o ZIP.
2. Abra a pasta do projeto.
3. Abra o terminal nessa pasta.
4. Execute npm install.
5. Configure as variáveis de ambiente antes de iniciar:
   - ADMIN_PASSWORD: uma senha forte escolhida por você.
   - DISCORD_WEBHOOK_URL: opcional; URL do webhook do canal Discord.
6. Execute npm start.
7. Abra http://localhost:3000.

### Windows PowerShell (teste local)

$env:ADMIN_PASSWORD="coloque-uma-senha-forte-aqui"
$env:DISCORD_WEBHOOK_URL="cole-a-url-do-webhook-aqui"
npm install
npm start


Não compartilhe a senha administrativa nem a URL do webhook. Se a URL vazar, apague o webhook antigo e crie outro no Discord.

## 2. Vincular ao Discord

Para a integração mais simples, este projeto usa um **webhook do Discord**. Ele publica uma mensagem no canal quando um registro é criado; não precisa manter um bot ligado 24 horas.

1. No Discord, abra as configurações do servidor.
2. Vá em **Integrações → Webhooks** (o nome pode variar).
3. Crie um webhook no canal onde quer receber os registros.
4. Copie a URL e configure-a como DISCORD_WEBHOOK_URL no ambiente em que o site estiver hospedado.

A publicação é automática. Este método envia registros do site para o Discord; não cria comandos de barra nem permite cadastrar registros por comandos no Discord. Para isso, seria necessário acrescentar um bot Discord completo.

## 3. Publicar na internet

O projeto pode ser hospedado em um serviço que execute Node.js, como Render ou Railway. O nome dos menus pode mudar conforme o serviço.

- Crie um serviço web a partir deste projeto/repositório.
- Build command: npm install
- Start command: npm start
- Adicione as variáveis ADMIN_PASSWORD e, se usar Discord, DISCORD_WEBHOOK_URL.
- **Persistência do banco:** o SQLite cria registros.db. Em hospedagens com disco efêmero, os dados podem sumir ao reiniciar/deploy. Configure um disco persistente e defina DB_PATH para um caminho dentro dele (por exemplo /var/data/registros.db, se esse for o caminho do disco no serviço).

Para colocar o projeto online, normalmente você precisará enviar os arquivos para um repositório Git e conectar esse repositório ao serviço de hospedagem. O link público será fornecido pelo serviço após o deploy; ele não é criado automaticamente por este código.

## Segurança e limites

- Este é um ponto de partida para RP, não um sistema oficial.
- Qualquer pessoa que obtenha a senha poderá criar e consultar registros.
- Use apenas dados fictícios do roleplay; não insira dados pessoais reais.
- Para uso com várias pessoas, recomenda-se evoluir para contas individuais, permissões por cargo, auditoria, backup e banco de dados gerenciado.
- O login usa sessões em memória; depois de reiniciar o servidor, os usuários precisarão entrar novamente.
'''
}
for rel, content in files.items():
    with open(os.path.join(base, rel), "w", encoding="utf-8") as f:
        f.write(content)

zip_path = "/mnt/data/sistema_rota_rp_site.zip"
with zipfile.ZipFile(zip_path, "w", zipfile.ZIP_DEFLATED) as z:
    for root, dirs, filenames in os.walk(base):
        for filename in filenames:
            full = os.path.join(root, filename)
            z.write(full, os.path.relpath(full, base))
print(zip_path, os.path.getsize(zip_path), "bytes")
