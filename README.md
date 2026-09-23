# Chegar Certo — Pré-check-in Inteligente (PoC · Hackathon Health AI)

> **Decisão registrada:** não havia projeto/stack pré-existente no ambiente (diretório vazio, sem README).
> Escolhi a solução mais simples e demonstrável: **Node.js puro, zero dependências** (só módulos nativos `http`/`fs`),
> frontend em HTML/CSS/JS puro (sem build), persistência em `data.json` e **mock determinístico de IA** com ponto de
> troca por LLM real. Assim a demo funciona **offline**, sem `npm install` e sem serviços externos.

## 1. O problema
O paciente agenda a consulta, mas os dados do agendamento (convênio, telefone…) podem estar desatualizados.
A divergência só aparece na chegada → retrabalho, atraso e problema na abertura da guia.

## 2. A solução
Transformar o lembrete da consulta em um **pré-check-in inteligente**:
1. Paciente recebe o lembrete e abre o link;
2. Confirma presença;
3. Confere os dados cadastrais (a IA interpreta respostas em linguagem natural);
4. Corrige divergências com confirmação explícita ANTES × DEPOIS;
5. Responde a uma micro pré-triagem (máx. 3 perguntas, só coleta — sem diagnóstico);
6. A recepção vê tudo em um painel com status 🟢/🟡/🔴 e auditoria.

## 3. Como executar
Pré-requisito: **Node.js 18+** (nenhuma outra dependência).

```bat
cd chegar-certo
node server.js
rem ou: npm start
```
Abrir no navegador:
- Paciente (lembrete): http://localhost:3000/#/
- Pré-check-in direto da Maria: http://localhost:3000/#/paciente/p1
- Painel da recepção: http://localhost:3000/#/recepcao
- Reset da demo (restaura a base fictícia): botão **↺ Reset demo** no topo ou `POST /api/reset`.

## 4. Arquitetura
```
chegar-certo/
  server.js        → HTTP + API REST + mock de IA + persistência (data.json)
  data.json        → base fictícia (criada a partir do seed se ausente)
  package.json     → apenas script "start" (zero deps)
  public/
    index.html     → shell + rodapé de segurança
    styles.css     → visual HealthTech
    app.js         → roteamento (#/, #/paciente/:id, #/recepcao) + wizard + painel
```
Fluxo de dados: `app.js --fetch--> /api/* ----> data.json`. Sem banco externo de propósito (PoC local).

**Endpoints:** `GET /api/state` · `POST /api/attendance` · `POST /api/ai-interpret` ·
`POST /api/correction` · `POST /api/validate-data` · `POST /api/triage` · `POST /api/complete` · `POST /api/reset`.

## 5. Estrutura dos dados
`Patient { id, name, phone, insurance, insurance_card }` ·
`Appointment { id, patient_id, date, time, specialty, attendance_status, checkin_status }` ·
`PreCheckin { id, patient_id, appointment_id, attendance_confirmed, registration_status (pendente|corrigido|validado), triage_status, triage_answers[3], completed_at, no_response? }` ·
`AuditLog { id, patient_id, field, old_value, new_value, changed_at, source }`.
Seed: **Maria Silva** (14:00, Unimed, pendente — caso da demo), **João Santos** (15:00, `no_response` — "não respondeu"),
**Ana Oliveira** (16:00, pendente — paciente extra).

## 6. Como funciona a IA / mock
`aiInterpret(text, patient)` em `server.js`: normaliza o texto, detecta intenção (`correction` | `confirm_all` | `unknown`),
extrai campo/valor (convênio via mapa canônico Bradesco/Unimed/Amil/…, telefone via regex, carteirinha via padrão)
e devolve `{ field, old_value, new_value, reply }`. O frontend **sempre pede confirmação** antes de salvar.
**Troca por LLM real:** substituir o corpo de `aiInterpret()` pela chamada ao provedor (ex. se `LLM_API_KEY` definida),
mantendo o mesmo contrato de retorno — nenhum outro código precisa mudar.
A pré-triagem tem guarda de segurança: se a resposta contiver pedido de diagnóstico/prescrição, registra e escala
para avaliação humana. A IA nunca diagnostica, prescreve ou decide clinicamente.

## 7. Fluxo da demo (roteiro obrigatório, 17 passos)
1. Abrir o painel da recepção → Maria com cadastro **pendente**.
2. Abrir o pré-check-in como paciente (link da Maria).
3. Lembrete: consulta amanhã 14:00.
4. Paciente confirma presença ("Sim, vou comparecer").
5. Sistema apresenta dados (Unimed, telefone, carteirinha).
6. Paciente digita: **"Meu convênio mudou para Bradesco Saúde."**
7. IA identifica a alteração (convênio).
8. Sistema mostra ANTES × DEPOIS e pede confirmação.
9. Paciente confirma.
10. Base fictícia atualizada (Maria → Bradesco Saúde).
11. Log de auditoria registrado (campo, antes, depois, data/hora, origem "Pré-check-in do paciente").
12. Paciente responde às 3 perguntas da pré-triagem.
13. Pré-check-in finalizado ("Informações registradas para a equipe…").
14. Voltar ao painel da recepção.
15. Maria: presença confirmada · cadastro validado · pré-triagem concluída (🟢 Pronto).
16. João Santos: **"Não respondeu — validar na chegada"** (🟡, nada preenchido automaticamente).
17. (Bônus) Botão Reset restaura tudo para repetir a demo sem tocar no código.

## 8. Limitações
- Sem login/autenticação, sem reagendamento (só "contatar a clínica"), sem integrações (convênio, prontuário, WhatsApp/SMS reais).
- Persistência em arquivo local — não é para produção nem multi-instância.
- IA é um mock por regras (PT-BR); cobre bem o roteiro da demo, mas não é um NLU geral.
- Métricas são apenas contadores no painel (pré-check-ins, presenças, divergências, triagens, não-respostas) — **nenhum resultado clínico/operacional é afirmado**.

## 9. O que é mock
- **IA mock:** `aiInterpret()` — regras determinísticas, funciona offline.
- **Lembrete/WhatsApp/SMS:** simulado — a home do paciente *é* a tela do link.
- **Base de pacientes:** `data.json`, 100% sintética, recriável via Reset.
- **Tudo que é mock é substituível** sem mudar a UI (contratos JSON estáveis).

## 10. Próximos passos
- Plugar LLM real em `aiInterpret()` (manter confirmação humana + trilhas de auditoria).
- Envio real de lembretes (WhatsApp/SMS) com link tokenizado por paciente.
- Auth leve para recepção + validação assistida na chegada ("Não respondeu").
- Persistência real (SQLite/Postgres), exportação de métricas e LGPD/consentimento formal.
- Validação com dados reais anonimizados e medição de no-show/retrabalho antes de qualquer afirmação de impacto.

---
**Segurança e limites:** dados da PoC são sintéticos. A IA não realiza diagnóstico, não prescreve, não classifica doenças
e não substitui profissional de saúde. Exceções são encaminhadas para atendimento humano: recepção (11) 4002-8922.
