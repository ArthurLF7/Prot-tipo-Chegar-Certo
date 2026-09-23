/* Chegar Certo — PoC Hackathon Health AI
 * Servidor zero-dependencias (apenas modulos nativos do Node).
 * Persistencia em data.json (base ficticia). Mock de IA deterministico,
 * com ponto de troca por LLM real (ver aiInterpret()).
 */
const http = require("http");
const fs = require("fs");
const path = require("path");

const PORT = process.env.PORT || 3000;
const DATA_FILE = path.join(__dirname, "data.json");
const PUBLIC_DIR = path.join(__dirname, "public");

/* ---------- SEED (dados 100% sinteticos) ---------- */
function seed() {
  const tomorrow = new Date(Date.now() + 24 * 60 * 60 * 1000);
  const dateStr = tomorrow.toISOString().slice(0, 10);
  return {
    patients: [
      { id: "p1", name: "Maria Silva", phone: "(11) 99999-9999", insurance: "Unimed", insurance_card: "UN-123456" },
      { id: "p2", name: "João Santos", phone: "(11) 98888-7777", insurance: "Amil", insurance_card: "AM-654321" },
      { id: "p3", name: "Ana Oliveira", phone: "(11) 97777-6666", insurance: "SulAmérica", insurance_card: "SA-112233" }
    ],
    appointments: [
      { id: "a1", patient_id: "p1", date: dateStr, time: "14:00", specialty: "Clínica Médica", attendance_status: "pendente", checkin_status: "pendente" },
      { id: "a2", patient_id: "p2", date: dateStr, time: "15:00", specialty: "Clínica Médica", attendance_status: "pendente", checkin_status: "nao_respondeu" },
      { id: "a3", patient_id: "p3", date: dateStr, time: "16:00", specialty: "Clínica Médica", attendance_status: "pendente", checkin_status: "pendente" }
    ],
    precheckins: [
      { id: "c1", patient_id: "p1", appointment_id: "a1", attendance_confirmed: null, registration_status: "pendente", triage_status: "pendente", triage_answers: [], completed_at: null },
      { id: "c2", patient_id: "p2", appointment_id: "a2", attendance_confirmed: null, registration_status: "pendente", triage_status: "pendente", triage_answers: [], completed_at: null, no_response: true },
      { id: "c3", patient_id: "p3", appointment_id: "a3", attendance_confirmed: null, registration_status: "pendente", triage_status: "pendente", triage_answers: [], completed_at: null }
    ],
    audit_log: []
  };
}

function load() {
  try {
    if (!fs.existsSync(DATA_FILE)) { const s = seed(); fs.writeFileSync(DATA_FILE, JSON.stringify(s, null, 2)); return s; }
    return JSON.parse(fs.readFileSync(DATA_FILE, "utf8"));
  } catch (e) { const s = seed(); fs.writeFileSync(DATA_FILE, JSON.stringify(s, null, 2)); return s; }
}
function save(db) { fs.writeFileSync(DATA_FILE, JSON.stringify(db, null, 2)); }

function enriched(db) {
  return db.patients.map((p) => {
    const ap = db.appointments.find((a) => a.patient_id === p.id);
    const pc = db.precheckins.find((c) => c.patient_id === p.id);
    const logs = db.audit_log.filter((l) => l.patient_id === p.id);
    let status = "pendente";
    if (pc.no_response) status = "nao_respondeu";
    else if (pc.completed_at) status = "pronto";
    else if (pc.attendance_confirmed === false) status = "cancelado";
    else if (pc.attendance_confirmed === true || pc.registration_status !== "pendente" || pc.triage_status !== "pendente") status = "em_andamento";
    return { patient: p, appointment: ap, precheckin: pc, audit: logs, status };
  });
}

/* ---------- MOCK DE IA (deterministico, offline) ----------
 * Ponto de troca: se LLM_API_KEY estiver configurada, este seria o local
 * para chamar o provedor real. Mantido mock por padrao para a demo
 * funcionar sem servicos externos.
 */
const INSURANCE_CANON = [
  ["bradesco", "Bradesco Saúde"], ["unimed", "Unimed"], ["amil", "Amil"],
  ["sulam", "SulAmérica"], ["sul am", "SulAmérica"], ["hapvida", "Hapvida"],
  ["notre", "NotreDame Intermédica"], ["porto", "Porto Saúde"], ["golden", "Golden Cross"],
  ["cassi", "CASSI"], ["geap", "GEAP"], ["particular", "Particular (sem convênio)"]
];

function aiInterpret(text, patient) {
  const raw = (text || "").trim();
  const t = raw.toLowerCase().normalize("NFD").replace(/[\u0300-\u036f]/g, "");

  // Intencao: confirmar que esta tudo certo
  if (/^(sim|tudo certo|esta tudo|tá tudo|confirmo|correto|tudo correto|nao precisa|sem alter)/.test(t) && t.length < 60 && !/mudou|troc|alter|novo|nova|agora e|agora eh|corrig|atualiz/.test(t)) {
    return { intent: "confirm_all", field: null, old_value: null, new_value: null, reply: "Obrigado por confirmar! Seus dados foram validados. ✅" };
  }
  // Telefone?
  const phoneMatch = raw.match(/\(?\d{2}\)?[\s-]?\d{4,5}-?\d{4}/);
  if (/telefone|celular|whatsapp|numero|fone/.test(t) && phoneMatch) {
    return { intent: "correction", field: "phone", old_value: patient.phone, new_value: phoneMatch[0], reply: `Entendi! Seu telefone mudaria de ${patient.phone} para ${phoneMatch[0]}. Confirma?` };
  }
  if (phoneMatch && /mudou|novo|troque|atualiz|corrig|meu/.test(t)) {
    return { intent: "correction", field: "phone", old_value: patient.phone, new_value: phoneMatch[0], reply: `Entendi! Seu telefone mudaria de ${patient.phone} para ${phoneMatch[0]}. Confirma?` };
  }
  // Carteirinha?
  const cardMatch = raw.match(/(carteirinha[^:]*[:\s]+)([A-Za-z0-9-]{4,})/i);
  if (/carteirinha|carteira|matricula/.test(t)) {
    const nv = cardMatch ? cardMatch[2].toUpperCase() : raw.replace(/.*?carteirinha[^a-z0-9]*/i, "").trim().toUpperCase().slice(0, 20);
    if (nv) return { intent: "correction", field: "insurance_card", old_value: patient.insurance_card, new_value: nv, reply: `Entendi! A carteirinha mudaria de ${patient.insurance_card} para ${nv}. Confirma?` };
  }
  // Convenio (caso principal da demo + generico)
  const mentionsInsurance = /convenio|plano|unimed|bradesco|amil|sulamerica|sul america|hapvida|notre|porto|golden|cassi|geap|particular|saude/.test(t);
  if (mentionsInsurance || /mudou|troc|alter|novo|nova|agora/.test(t)) {
    let canonical = null;
    for (const [key, label] of INSURANCE_CANON) { if (t.includes(key)) { canonical = label; break; } }
    if (!canonical) {
      const m = raw.match(/(?:para|p\/|pra)\s+([A-Za-zÀ-ú ]{3,40})/i) || raw.match(/(?:agora (?:e|é)\s+)([A-Za-zÀ-ú ]{3,40})/i);
      if (m) canonical = m[1].replace(/[.]+$/, "").trim();
    }
    if (canonical && canonical.toLowerCase() !== patient.insurance.toLowerCase()) {
      return { intent: "correction", field: "insurance", old_value: patient.insurance, new_value: canonical, reply: `Entendi! Seu convênio mudaria de ${patient.insurance} para ${canonical}. Confirma a alteração?` };
    }
    if (mentionsInsurance) {
      return { intent: "unknown", field: null, old_value: null, new_value: null, reply: `Só confirmando: seu convênio atual aqui é "${patient.insurance}". Qual é o novo convênio? (ex.: "Meu convênio mudou para Bradesco Saúde.")` };
    }
  }
  // Nome?
  if (/meu nome|me chamo|corrija meu nome/.test(t)) {
    return { intent: "unknown", field: "name", old_value: patient.name, new_value: null, reply: "Claro! Qual é o nome correto? Vou encaminhar para validação da recepção." };
  }
  return { intent: "unknown", field: null, old_value: null, new_value: null, reply: "Não entendi exatamente o que mudou. Pode me dizer assim: \"Meu convênio mudou para Bradesco Saúde\" ou \"Meu telefone mudou para (11) 98888-0000\". Se preferir, uma pessoa da recepção pode ajudar. 🤝" };
}

/* ---------- HTTP ---------- */
const MIME = { ".html": "text/html; charset=utf-8", ".css": "text/css; charset=utf-8", ".js": "text/javascript; charset=utf-8", ".json": "application/json; charset=utf-8", ".svg": "image/svg+xml" };

function send(res, code, body, type = "application/json; charset=utf-8") {
  res.writeHead(code, { "Content-Type": type, "Access-Control-Allow-Origin": "*" });
  res.end(body);
}
function bodyOf(req) { return new Promise((resolve) => { let b = ""; req.on("data", (c) => (b += c)); req.on("end", () => { try { resolve(b ? JSON.parse(b) : {}); } catch { resolve({}); } }); }); }

const server = http.createServer(async (req, res) => {
  const url = new URL(req.url, `http://${req.headers.host}`);
  const p = url.pathname;

  if (req.method === "OPTIONS") { res.writeHead(204, { "Access-Control-Allow-Origin": "*", "Access-Control-Allow-Methods": "GET,POST", "Access-Control-Allow-Headers": "Content-Type" }); return res.end(); }

  // ---- API ----
  if (p === "/api/state" && req.method === "GET") return send(res, 200, JSON.stringify({ data: enriched(load()) }));
  if (p === "/api/reset" && req.method === "POST") { const s = seed(); save(s); return send(res, 200, JSON.stringify({ ok: true, data: enriched(s) })); }

  if (p === "/api/ai-interpret" && req.method === "POST") {
    const { patientId, text } = await bodyOf(req);
    const db = load();
    const pat = db.patients.find((x) => x.id === patientId);
    if (!pat) return send(res, 404, JSON.stringify({ error: "Paciente não encontrado" }));
    return send(res, 200, JSON.stringify(aiInterpret(text, pat)));
  }

  if (p === "/api/attendance" && req.method === "POST") {
    const { patientId, confirmed } = await bodyOf(req);
    const db = load();
    const pc = db.precheckins.find((c) => c.patient_id === patientId);
    const ap = db.appointments.find((a) => a.patient_id === patientId);
    if (!pc) return send(res, 404, JSON.stringify({ error: "Paciente não encontrado" }));
    pc.attendance_confirmed = !!confirmed; pc.no_response = false;
    if (ap) { ap.attendance_status = confirmed ? "confirmada" : "recusada"; if (confirmed && ap.checkin_status === "nao_respondeu") ap.checkin_status = "em_andamento"; }
    save(db); return send(res, 200, JSON.stringify({ ok: true, data: enriched(db) }));
  }

  if (p === "/api/correction" && req.method === "POST") {
    const { patientId, field, newValue } = await bodyOf(req);
    const db = load();
    const pat = db.patients.find((x) => x.id === patientId);
    const pc = db.precheckins.find((c) => c.patient_id === patientId);
    const allowed = ["insurance", "phone", "insurance_card", "name"];
    if (!pat || !pc) return send(res, 404, JSON.stringify({ error: "Paciente não encontrado" }));
    if (!allowed.includes(field) || !newValue) return send(res, 400, JSON.stringify({ error: "Campo ou valor inválido" }));
    const oldValue = pat[field];
    if (String(oldValue) === String(newValue)) return send(res, 200, JSON.stringify({ ok: true, noop: true, data: enriched(db) }));
    pat[field] = newValue;
    pc.registration_status = "corrigido"; pc.no_response = false;
    db.audit_log.push({ id: "log" + (db.audit_log.length + 1), patient_id: patientId, field, old_value: oldValue, new_value: newValue, changed_at: new Date().toISOString(), source: "Pré-check-in do paciente" });
    const ap = db.appointments.find((a) => a.patient_id === patientId);
    if (ap && ap.checkin_status !== "concluido") ap.checkin_status = "em_andamento";
    save(db); return send(res, 200, JSON.stringify({ ok: true, data: enriched(db) }));
  }

  if (p === "/api/validate-data" && req.method === "POST") {
    const { patientId } = await bodyOf(req);
    const db = load();
    const pc = db.precheckins.find((c) => c.patient_id === patientId);
    if (!pc) return send(res, 404, JSON.stringify({ error: "Paciente não encontrado" }));
    if (pc.registration_status === "pendente") pc.registration_status = "validado";
    pc.no_response = false;
    save(db); return send(res, 200, JSON.stringify({ ok: true, data: enriched(db) }));
  }

  if (p === "/api/triage" && req.method === "POST") {
    const { patientId, answers } = await bodyOf(req);
    const db = load();
    const pc = db.precheckins.find((c) => c.patient_id === patientId);
    if (!pc) return send(res, 404, JSON.stringify({ error: "Paciente não encontrado" }));
    if (!Array.isArray(answers) || answers.length < 3 || answers.some((a) => !String(a || "").trim())) return send(res, 400, JSON.stringify({ error: "Responda às 3 perguntas da pré-triagem." }));
    // Guarda de seguranca: bloqueia pedidos de diagnostico/prescricao (coleta apenas)
    const joined = answers.join(" ").toLowerCase();
    if (/diagnostic|prescrev|receita|remedio|tratamento|doenca grave|emergencia|dor no peito|falta de ar|desmaio/.test(joined.normalize("NFD").replace(/[\u0300-\u036f]/g, ""))) {
      pc.triage_answers = answers; pc.triage_status = "concluida";
      save(db);
      return send(res, 200, JSON.stringify({ ok: true, escalated: true, message: "Informações registradas. Por segurança, a equipe fará avaliação presencial — a IA não realiza diagnóstico.", data: enriched(db) }));
    }
    pc.triage_answers = answers; pc.triage_status = "concluida"; pc.no_response = false;
    save(db); return send(res, 200, JSON.stringify({ ok: true, data: enriched(db) }));
  }

  if (p === "/api/complete" && req.method === "POST") {
    const { patientId } = await bodyOf(req);
    const db = load();
    const pc = db.precheckins.find((c) => c.patient_id === patientId);
    const ap = db.appointments.find((a) => a.patient_id === patientId);
    if (!pc) return send(res, 404, JSON.stringify({ error: "Paciente não encontrado" }));
    if (pc.attendance_confirmed !== true) return send(res, 400, JSON.stringify({ error: "Confirme a presença antes de concluir." }));
    if (pc.registration_status === "pendente") pc.registration_status = "validado";
    else if (pc.registration_status === "corrigido") pc.registration_status = "validado"; // mantem historico no audit_log
    pc.completed_at = new Date().toISOString();
    if (ap) ap.checkin_status = "concluido";
    save(db); return send(res, 200, JSON.stringify({ ok: true, data: enriched(db) }));
  }

  // ---- Estáticos ----
  let file = p === "/" ? "/index.html" : p;
  const fp = path.join(PUBLIC_DIR, decodeURIComponent(file).replace(/^\//, ""));
  if (!fp.startsWith(PUBLIC_DIR) || !fs.existsSync(fp) || fs.statSync(fp).isDirectory()) {
    return send(res, 200, fs.readFileSync(path.join(PUBLIC_DIR, "index.html")), MIME[".html"]);
  }
  return send(res, 200, fs.readFileSync(fp), MIME[path.extname(fp).toLowerCase()] || "application/octet-stream");
});

server.listen(PORT, () => console.log(`Chegar Certo PoC rodando em http://localhost:${PORT}`));
