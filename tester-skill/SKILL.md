---
name: chatguru-tester
description: Use this skill whenever the user wants to test, validate or evaluate AI bot prompts on the ChatGuru platform. Trigger when the user mentions testing a bot, validating prompts on ChatGuru, checking if an AI assistant is following its prompt correctly, testing client bots, simulating client conversations, or generating performance/quality reports for AI assistants. Also trigger when the user wants to test scenarios like qualified clients, disqualified clients, transfer to human, or scheduling — even if they don't explicitly say "ChatGuru". Use this skill whenever someone wants to check if their AI is working correctly before deploying it to real clients.
---

# ChatGuru Prompt Tester

This skill tests AI bots on the ChatGuru platform by simulating real client conversations, evaluating whether the AI correctly follows its configured prompts, and generating a PDF performance report.

## What you need from the user

Before starting, collect:

1. **Prompt principal** — the main system prompt that defines the AI's behavior, persona, and rules
2. **Prompt de qualificação** — the criteria that define a qualified vs. disqualified lead/client
3. **Prompt de transferência** — when and how the AI should transfer to a human agent or schedule appointments
4. **Link do ChatGuru** — the direct link to the conversation (already logged in)

If any of these are missing, ask the user before proceeding. All four are required.

---

## Step 1: Generate 6 test scenarios

Based on the three prompts, generate exactly 6 test scenarios that cover the key flows. Each scenario simulates a realistic client with a specific goal.

**Scenario distribution:**
- **2x Desqualificado** — clients who clearly don't meet the qualification criteria
- **2x Qualificado** — clients who meet the criteria and should be moved forward
- **1x Transferência** — client who explicitly requests to speak with a human
- **1x Agendamento** — client who wants to schedule an appointment

For each scenario, write:
- `nome`: short descriptive name (e.g., "Cliente desqualificado - fora do perfil")
- `objetivo`: what this client is trying to do
- `perfil`: brief client persona description
- `mensagens`: a sequence of 3–6 realistic messages the client would send
- `criterio_sucesso`: what the AI must do correctly for this scenario to pass

Make scenarios realistic — use natural, casual Portuguese. Vary the tone: some clients are direct, some are vague, some are resistant.

---

## Step 2: Execute each scenario in ChatGuru

Use **Playwright MCP** to run each scenario in the user's local browser. The process is sequential — one scenario at a time.

### Setup
1. Use `browser_navigate` to open the ChatGuru link provided by the user
2. Use `browser_snapshot` to confirm the page loaded and the chat input is visible
3. Use `browser_screenshot` to capture the initial state

### For each scenario (including the first):

Always start every scenario — including the very first one — with a reset. There may already be an open session from a previous test, so skipping `reiniciar` on the first scenario can contaminate the results.

**Reset + activate (all 6 scenarios):**
1. Use `browser_type` to type `reiniciar` and send it
2. Wait 3 seconds
3. Use `browser_type` to type `supersdr` and send it
4. Wait 5–10 seconds for the AI's activation response
5. Use `browser_screenshot` to capture the activation response

**Then send the scenario messages:**
1. Find the chat input field using `browser_snapshot`
2. For each client message in the scenario:
   - `browser_type` the message into the input field
   - Send it (Enter key or send button with `browser_click`)
   - Wait for the AI to finish responding (5–15 seconds)
   - `browser_screenshot` after each exchange
3. Record the full conversation (messages sent + AI responses read from the snapshot)

### Reading AI responses
After sending each message, use `browser_snapshot` to read the AI's response text from the chat — don't rely only on screenshots for recording the conversation. Extract the text from the last AI message bubble in the snapshot.

### Recording conversations
For each scenario, capture:
```json
{
  "scenario_id": 1,
  "scenario_name": "...",
  "messages": [
    {"role": "client", "text": "..."},
    {"role": "ai", "text": "...", "screenshot": "scenario1_step1.png"}
  ]
}
```

### Checkpoint: salvar progresso após cada cenário

Após concluir e avaliar cada cenário, salve o resultado parcial em `progresso_teste.json` no diretório de saída. Isso permite retomar o teste a partir do cenário correto se a sessão for interrompida, sem precisar repetir cenários já executados.

### Recuperação de erros do Playwright

Se ocorrer um erro de navegação ou o browser fechar inesperadamente:
1. Use `browser_navigate` para reabrir o link do ChatGuru
2. Envie `reiniciar` seguido de `supersdr` para reiniciar o cenário atual do zero
3. Continue a partir do cenário que falhou

### Important browser behavior notes
- Always wait for the AI to finish responding before sending the next message — watch for the typing indicator to disappear in the snapshot
- If the AI doesn't respond within 20 seconds, take a snapshot to check the current state before waiting more
- Use `browser_scroll` to scroll down in the chat if the latest messages are not visible
- The AI may send multiple message bubbles — wait for all of them (no typing indicator) before responding
- If the input field loses focus between messages, click it again with `browser_click` before typing

---

## Step 3: Evaluate each scenario

For each scenario, evaluate:

**1. Seguiu o prompt principal?**
- Did the AI maintain the correct persona and tone?
- Did it follow the rules and restrictions in the main prompt?
- Did it avoid saying things that contradict the prompt?

**2. Qualificação correta?**
- For disqualified scenarios: did the AI correctly identify the client as outside the criteria?
- For qualified scenarios: did the AI correctly advance the conversation?
- Was the criteria applied consistently?

**3. Transferência / agendamento?**
- For transfer scenarios: did the AI offer/execute the transfer to a human?
- For scheduling scenarios: did the AI initiate or complete the scheduling process?
- Did the AI communicate clearly to the client what was happening?

**Rating scale per criterion:** ✅ Passou | ⚠️ Parcial | ❌ Falhou

**Overall scenario rating:**
- All criteria passed → ✅ Aprovado
- One partial or one failure → ⚠️ Atenção
- Multiple failures → ❌ Reprovado

Provide a brief justification (2–3 sentences) for each rating, with direct reference to what the AI said.

---

### ⚠️ Regra crítica: silêncio após transferência NÃO é bug

Antes de marcar um cenário como ❌ REPROVADO por "bot travou" ou "sem resposta", verifique **obrigatoriamente**:

1. A última mensagem do bot foi uma mensagem de espera ou transferência? (ex: *"Aguarde um momento"*, *"Vou te transferir"*, *"Um instante, por favor"*)
2. O prompt de transferência define que o bot para de responder após transferir para um humano?

**Se SIM para ambos → o silêncio é o comportamento CORRETO.** Marque ✅ para o critério de transferência. O bot transferiu com sucesso e o humano assumiu a conversa.

**Só marque como "BOT TRAVOU"** (❌) quando o silêncio ocorre:
- Sem nenhuma mensagem de espera/transferência anterior, **E**
- Por mais de 90 segundos desde a última mensagem do cliente

### Definição precisa de timeout (bot realmente travado)

O bot está **TRAVADO** (❌) quando todas estas condições são verdadeiras:
- Ficou **mais de 90 segundos** sem enviar nenhuma mensagem
- A última mensagem do bot **não foi** uma mensagem de espera ou transferência
- A mensagem do cliente era uma resposta direta aguardando processamento

Não confundir com: silêncio após transferência confirmada pelo prompt (ver regra acima).

---

## Step 4: Generate PDF report

Use the `anthropic-skills:pdf` skill to generate a professional PDF report.

### Se gerar o PDF via pdfkit (Node.js)

Se optar por gerar o PDF diretamente com pdfkit (para layouts mais ricos), **sempre** inclua `bufferPages: true` no construtor:

```js
const doc = new PDFDocument({ margin: 50, size: 'A4', bufferPages: true });
```

Sem essa opção, `doc.switchToPage()` falha com `"switchToPage(N) out of bounds"` ao tentar adicionar rodapés ou números de página em todas as páginas, porque o pdfkit descarta páginas anteriores da memória por padrão. O erro só aparece no final e exige regerar o arquivo inteiro.

### Report structure:

```
[Logo placeholder or title]
RELATÓRIO DE PERFORMANCE — TESTE DE PROMPT
Data: [today's date]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RESUMO EXECUTIVO
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Total de cenários testados: 6
Aprovados: X  |  Atenção: X  |  Reprovados: X
Taxa de aprovação: XX%

[One paragraph summary of overall AI performance]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CENÁRIOS TESTADOS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

CENÁRIO 1: [Nome]
Tipo: Desqualificado / Qualificado / Transferência / Agendamento
Resultado: ✅ Aprovado / ⚠️ Atenção / ❌ Reprovado

Critérios:
• Seguiu prompt principal: ✅/⚠️/❌
• Qualificação correta: ✅/⚠️/❌
• Transferência/Agendamento: ✅/⚠️/❌ (se aplicável)

Análise:
[2–3 sentences explaining the result, with examples from the conversation]

Trecho da conversa:
Cliente: "..."
IA: "..."

[Repeat for all 6 scenarios]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PONTOS DE ATENÇÃO E RECOMENDAÇÕES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[Bullet list of specific issues found and how to fix them in the prompts]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CONCLUSÃO
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[Overall verdict and next steps]
```

Save the report as `relatorio_chatguru_[YYYY-MM-DD].pdf` in the current directory. Tell the user where it was saved.

---

## Tips for good evaluations

- Read the AI's full response, not just the first sentence — sometimes it qualifies or transfers later in the message
- If the AI asked a follow-up question (not a final decision), that's usually correct behavior — don't penalize it
- Context matters: a "partial" rating is appropriate when the AI was mostly right but missed a key detail
- Be specific in your analysis — quote the AI's actual words when explaining a failure
- If the AI's behavior was ambiguous (prompt could be interpreted multiple ways), note this in the recommendations section

---

## Final checklist before delivering the report

- [ ] All 6 scenarios were executed
- [ ] Each scenario has at least 3 message exchanges
- [ ] All evaluations have written justifications
- [ ] The recommendations section addresses all failures found
- [ ] The PDF was saved and the path shared with the user
