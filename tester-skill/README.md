# ChatGuru Tester Skill

Skill para testar bots de IA na plataforma ChatGuru, simulando conversas reais de clientes e gerando relatório PDF de performance.

## Pré-requisitos

- [Claude Code](https://claude.ai/code) instalado
- Plugin Playwright MCP configurado (para automação de browser)
- Acesso ao ChatGuru (usuário logado)

## Instalação

### 1. Copiar o skill

Copie a pasta `tester-skill/` para o diretório de skills do Claude:

```bash
# Windows
cp -r tester-skill/ "$env:USERPROFILE/.claude/skills/chatguru-tester/"

# macOS / Linux
cp -r tester-skill/ ~/.claude/skills/chatguru-tester/
```

### 2. Configurar o Playwright MCP

Crie um arquivo `.mcp.json` na raiz do seu projeto:

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    }
  }
}
```

Depois, adicione `"playwright"` à lista `enabledMcpjsonServers` no seu `~/.claude/settings.json`:

```json
{
  "enabledMcpjsonServers": ["playwright"]
}
```

Reinicie o Claude Code para carregar o servidor MCP.

## Como usar

Abra o Claude Code na sua pasta de trabalho e diga algo como:

> "Preciso testar o bot da MS Visão no ChatGuru. Aqui estão os prompts: [cole os prompts] Link: https://..."

O skill será acionado automaticamente. Ele vai pedir:

1. **Prompt principal** — define o comportamento e persona do bot
2. **Prompt de qualificação** — critérios de lead qualificado/desqualificado
3. **Prompt de transferência** — quando e como transferir para humano
4. **Link do ChatGuru** — link direto para a conversa (já logado)

## O que o skill faz

1. Gera 6 cenários de teste baseados nos prompts
2. Abre o ChatGuru no seu browser via Playwright
3. Executa cada cenário (com `reiniciar` + `supersdr` antes de cada um)
4. Avalia as respostas do bot
5. Gera um relatório PDF com resultado de cada cenário e recomendações

## Comandos especiais do ChatGuru

| Comando | Função |
|---------|--------|
| `reiniciar` | Reseta a sessão atual |
| `supersdr` | Ativa o bot |

> **Importante:** Sempre envie `reiniciar` antes de cada cenário, inclusive o primeiro, para garantir que não há sessão aberta do teste anterior.

## Estrutura dos cenários gerados

| Tipo | Quantidade | Objetivo |
|------|-----------|---------|
| Desqualificado | 2 | Verificar se o bot identifica leads fora do perfil |
| Qualificado | 2 | Verificar o fluxo completo de coleta e transferência |
| Transferência | 1 | Verificar pedido explícito de humano |
| Cirurgia/Agendamento | 1 | Verificar direcionamento para fluxo específico |
