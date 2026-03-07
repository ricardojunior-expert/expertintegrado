---
name: prompt-builder
description: Cria os 3 prompts padrão do sistema (Orchestrator, Qualifier e Transfer/Protractor) preenchendo os templates com informações da documentação do cliente. Use sempre que o usuário fornecer documentação de cliente (PDF, DOCX, ou texto colado) e precisar montar ou gerar os prompts. Ativa com frases como "montar os prompts", "preencher os prompts", "criar prompts do cliente", "gerar prompts", "onboarding do cliente", "documentação do cliente", ou qualquer menção a prompts junto com documentação ou briefing de cliente. Quando o usuário compartilhar um arquivo e mencionar "prompt" ou "agente", use esta skill.
---

## O que esta skill faz

Lê a documentação do cliente (PDF, DOCX ou texto) e preenche os 3 templates de agente do sistema:

1. **Orchestrator** — agente conversacional principal que conduz o atendimento
2. **Qualifier** — agente analítico de qualificação de leads
3. **Transfer/Protractor** — agente de decisão de roteamento e transferência

Resultado: 3 arquivos `.md` prontos para uso em produção.

---

## Passo 1 — Receber a documentação

O usuário pode enviar a documentação de 3 formas:
- **PDF** → use a skill de PDF para extrair o texto completo
- **DOCX / Word** → use a skill de DOCX para extrair o texto completo
- **Texto colado no chat** → use diretamente

Se nenhuma documentação foi fornecida, peça:
> "Para montar os prompts, preciso da documentação do cliente. Pode compartilhar o PDF, Word ou colar o texto aqui?"

---

## Passo 2 — Extrair as informações

Leia toda a documentação e construa um mapa de extração. Use o guia abaixo.

---

### 2A. Contexto geral do cliente

| Campo | O que buscar | Se não encontrar |
|-------|-------------|------------------|
| `NOME_EMPRESA` | Nome comercial ou marca | ⚠️ PREENCHER |
| `AREA_NEGOCIO_TEXTO` | Texto exato que descreve o foco (ex: "Oftalmologia", "Direito Previdenciário", "Recrutamento") | Inferir pelo setor |
| `PERSONA_NOME` | Nome da persona do agente (ex: Helena, Fernando) | Deixar vazio — não é obrigatório |
| `PERSONA_GENERO` | Gênero gramatical (ela/ele) | Default: feminino |
| `PERSONA_EH_HUMANO` | Cliente quer que o agente se passe por humano? | Default: admite ser IA |
| `TIPO_DE_AGENDAMENTO` | O que acontece após qualificação: consulta, demo, ligação, assinatura de contrato | ⚠️ PREENCHER |

> ⚠️ **Documentos com seção "Parâmetros Técnicos", "Para o Implementador" ou equivalente:** essa seção contém configurações de plataforma (triggers de ativação, timing de follow-up, delays de mensagem, comportamentos automáticos do sistema) — **não são instruções de prompt**. Extrair dela apenas: **(a) nome/persona do agente** e **(b) mensagem inicial**, se presentes. Ignorar todo o restante.

---

### 2B. Orchestrator — guia completo de extração

#### Objetivo e persona
- Substituir `Gestão Empresarial` pela área real do cliente (`AREA_NEGOCIO_TEXTO`)
- Se a persona tem nome, adicionar `Você é: {{operator_name}}` após o `<objetivo>`
- Se o cliente quer que a IA se passe por humano, alterar a regra 9 de `<regras_gerais>` para: `"Se perguntarem se você é uma IA, diga que não — você é [NOME], atendente da equipe."`

#### Seções a preencher

| Placeholder / Seção | O que buscar | Se não encontrar |
|--------------------|-------------|------------------|
| `<conhecimento>` | Serviços, produtos, especialidades, unidades/endereços, horários, especialistas, tabelas de preço | Inferir pelos serviços mencionados na doc |
| `[FLUXO_DE_CONVERSA]` | Roteiro numerado da conversa, etapas do atendimento, script de vendas, jornada do cliente | ⚠️ PREENCHER — perguntar ao usuário |
| `[OBJECAO_COMUM_1..3]` + `[RESPOSTA_PARA_OBJECAO_N]` | FAQ, seção de objeções, "dúvidas frequentes", exemplos de resposta a resistências comuns | Inferir as 3 mais prováveis para o setor |
| `[FRASES_CARACTERISTICAS]` ×3 | Tom de voz, frases de exemplo, guia de comunicação, "como falar com o cliente" | Inferir pelo tom da documentação |
| `[REGRAS_CRITICAS]` ×3 | O que o agente NUNCA pode fazer: diagnósticos, prometer resultado, revelar valores, dar orientação jurídica | Inferir pelo tipo de negócio |
| `[PODE_INFORMAR]` ×2 | Serviços oferecidos, o que a empresa divulga publicamente | Inferir pelos serviços descritos |
| `[NAO_PODE_INFORMAR]` ×2 | Informações confidenciais, dados clínicos, valores, prazos de processo | ⚠️ PREENCHER |
| `[INFORMACOES_RESTRITAS]` | Texto direto para a frase "como honorários, valores, prazos..." | Inferir pelo tipo de negócio |

#### Seção `<auto_monitoramento>` — extras do cliente
| Placeholder | O que buscar | Exemplos encontrados nos prompts reais |
|-------------|-------------|----------------------------------------|
| `[REGRAS_AUTO_MONITORAMENTO_EXTRAS]` | Regras específicas de auto-checagem: "pode receber arquivos?", "prometeu resultado?", "ensinou a fazer sozinho?" | "Recusei receber laudo/CNIS/arquivo? → reformule aceitando" (Robson) |

Se não houver nada específico, **remover a linha `[REGRAS_AUTO_MONITORAMENTO_EXTRAS]`** do template.

#### Seção `<regras_protractor>` — situações adicionais
| Placeholder | O que buscar | Exemplos encontrados nos prompts reais |
|-------------|-------------|----------------------------------------|
| `[SITUACOES_ADICIONAIS_PROTRACTOR]` | Situações específicas do negócio que devem gerar transferência | "Paciente já existente" (MS Visão), "Dúvidas sobre convênio" (MS Visão), "Envio de senha" (Robson), "Solicitação de documentos médicos" (MS Visão) |

Se não houver nada específico além dos 5 padrões, **remover a linha** do template.

#### Quando o cliente define múltiplos tipos de contato

Se o documento definir **procedimentos distintos por tipo de contato** (ex.: "Para paciente novo → X; para dúvida de convênio → Y; para reclamação → Z"), estruturar `[FLUXO_DE_CONVERSA]` com branches condicionais:

- Tipos que **percorrem o fluxo normal** (coleta de dados → Qualifier) → representar como sub-seção dentro do fluxo principal
- Tipos que **transferem imediatamente** (emergência, reclamação, parceria, documentos, assuntos institucionais) → incluir em `[SITUACOES_ADICIONAIS_PROTRACTOR]`, não no fluxo principal
- Tipos com **mini-fluxo próprio** (ex.: cirurgia tem triagem diferente de consulta) → criar sub-seção `TIPO DE CONTATO: [Nome]` dentro de `[FLUXO_DE_CONVERSA]`

Formato sugerido para branches dentro de `[FLUXO_DE_CONVERSA]`:
```
TIPO DE CONTATO: [Nome] (ex.: Paciente Novo)
→ [Procedimento]
→ Acionar Qualifier quando: [condição]

TIPO DE CONTATO: [Nome] (ex.: Interesse em Cirurgia)
→ [Procedimento]
→ Encaminhar para: [Qualifier / Protractor / humano direto]
```

---

### 2C. Qualifier — guia de extração

| Placeholder | O que buscar | Se não encontrar |
|-------------|-------------|------------------|
| `[TIPO_DE_AGENDAMENTO]` | Tipo exato de reunião/consulta após qualificação | ⚠️ PREENCHER |
| `[DADO_QUALIFICATORIO_1..N]` | Dados que **determinam se qualifica ou não** (prazo, mínimo de tempo, ausência de litígio, renda, localização) | ⚠️ PREENCHER — crítico para o sistema funcionar |
| `[DADO_COMPLEMENTAR_1..N]` | Dados extras para contrato/próxima etapa (endereço, chave PIX, email, documentos) | Inferir pelo tipo de negócio |
| Critérios 🟢 e Condições | Perfil ideal de cliente (ICP), thresholds mensuráveis | ⚠️ PREENCHER |
| Motivos 🔴 | Deal-breakers, perfis que não se encaixam | Inferir como inverso dos critérios |

**Decidir se adicionar seções opcionais:**
- `<mapeamento_perguntas>`: adicionar se houver muitas perguntas distintas (>10) mapeadas para campos. Ver Value Promotora como modelo.
- `<acao_pos_qualificacao>`: adicionar se houver ações técnicas pós-qualificação (cancelar agendamentos, acionar setor específico). Ver Value Promotora como modelo.
- Campos extras no `<formato_resposta>`: adicionar `beneficio_identificado`, `via`, `checklist_status`, `beneficio_alternativo`, `pendencias` se o negócio tem múltiplos tipos de serviço. Ver Robson Gonçalves como modelo.

---

### 2D. Transfer/Protractor — guia de extração

Este template é quase totalmente genérico. Verificar apenas:

| Decisão | Como identificar |
|---------|-----------------|
| **Remover `TRANSFERIR_PARA_AGENT`?** | Cliente não mencionou múltiplos agentes automatizados, não tem "suporte", "SDR", "time de vendas" separados → remover seção toda |
| **Agentes customizados?** | Cliente tem agentes além de sdr/suporte/vendas → substituir os tipos |
| **Adicionar `<regras_qualifier>`?** | Quando o fluxo precisar que o Protractor saiba explicitamente o que fazer com cada resultado do Qualifier → adicionar seção |

**Estágios de CRM:** Se o documento descrever estágios de funil ou CRM (ex.: "Base → Tentando contato → Qualificado → Perdido"), identificar o **nome do estágio de desqualificação** (ex.: "Perdido") e usá-lo no texto do `<contexto_finalizacao>` — ex.: *"Lead desqualificado → mover para estágio 'Perdido' e encerrar sessão."*

---

### 2E. Decisão sobre seções opcionais

Use este guia para decidir o que incluir:

| Feature | Incluir quando | Sinal na documentação |
|---------|---------------|----------------------|
| `<regras_agendamento>` no Orchestrator | Cliente usa sistema de agenda | "agendar", "horário", "reunião", "consulta presencial/online", "Scheduler" |
| `<rastreamento_de_estado>` | Sempre incluir | N/A — é padrão |
| `<auto_monitoramento>` | Sempre incluir | N/A — é padrão |
| `<conhecimento>` inline | Informações simples (serviços, endereços) | Lista de produtos, unidades, especialistas |
| `<conhecimento>` com arquivos | Base de conhecimento complexa | Referências a FAQs, catálogos, tabelas |
| `TRANSFERIR_PARA_AGENT` no Protractor | Cliente tem múltiplos bots | Menção a equipes ou setores automatizados |
| `<regras_qualifier>` no Protractor | Fluxo mais rígido pós-qualifier | Cliente quer controle explícito das ações |

---

## Passo 3 — Preencher os templates

Leia cada template em `templates/`:
- `templates/orchestrator_template.md`
- `templates/qualifier_template.md`
- `templates/transfer_template.md`

Substitua cada placeholder pelo valor extraído. Regras de preenchimento:

**Quando a informação está na documentação:**
→ Usar diretamente, adaptando o tom/formato conforme necessário.

**Quando a informação pode ser inferida pelo contexto:**
→ Gerar o conteúdo mais provável com base no setor e proposta de valor do cliente.

**Quando a informação não está e não pode ser inferida:**
→ Preencher com: `⚠️ PREENCHER: [descrição curta do que é necessário aqui]`

**Para placeholders em lista** (ex: `[FRASES_CARACTERISTICAS]` ×3, objeções ×3):
→ Gerar exatamente o número indicado, mantendo o formato do template.

**Para critérios de qualificação:**
→ Manter o formato: `- CRITÉRIO N: [NOME] / Condição: [condição específica e mensurável]`

**Para a seção `<conhecimento>`:**
→ Se a documentação tem informações inline (serviços, endereços, especialistas) → inserir diretamente no bloco
→ Se a documentação referencia arquivos externos → manter como referência de arquivo (ex: `- tabela_precos.pdf`)

---

## Passo 4 — Gerar os arquivos de output

Criar a pasta `prompts/` no diretório atual (se não existir) e salvar os 3 arquivos:

```
prompts/
├── [nome_empresa]_orchestrator.md
├── [nome_empresa]_qualifier.md
└── [nome_empresa]_transfer.md
```

Usar o nome da empresa em minúsculas sem espaços ou acentos (ex: `clinica_sorrir_orchestrator.md`).

Ao finalizar, exibir um resumo breve:
- Quantos campos foram preenchidos com dados reais
- Quais campos foram inferidos (listar)
- Quais campos ficaram como `⚠️ PREENCHER` (listar com o que é necessário)

Perguntar se o usuário quer ajustar algum campo específico.
