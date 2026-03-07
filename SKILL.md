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

Use browser automation to run each scenario. The process is sequential — one scenario at a time.

### Setup
1. Open the ChatGuru link provided by the user
2. Take a screenshot to confirm the page loaded correctly

### For each scenario:

**First scenario:**
1. Type and send: `supersdr` (this activates the AI)
2. Wait 5–10 seconds for the AI to respond
3. Take a screenshot of the response
4. Continue sending the client messages from the scenario, one at a time
5. After each message, wait for the AI to respond (5–15 seconds)
6. Take a screenshot after each exchange
7. Record the full conversation (messages sent + AI responses)

**Scenarios 2–6 (reset first):**
1. Type and send: `reiniciar` (this resets the conversation)
2. Wait 3 seconds
3. Type and send: `supersdr` (reactivate)
4. Wait 5–10 seconds for the activation response
5. Then proceed as above

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

### Important browser behavior notes
- Always wait for the AI to finish responding before sending the next message — look for when typing stops
- If the AI doesn't respond within 20 seconds, wait another 10 seconds before continuing
- Scroll down in the chat to see the latest messages if needed
- The AI may send multiple message bubbles — wait for all of them before responding

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

## Step 4: Generate PDF report

Use the `anthropic-skills:pdf` skill to generate a professional PDF report.

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
