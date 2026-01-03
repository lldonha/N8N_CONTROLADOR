# CAAL Workflow Tools for n8n

Ferramentas CAAL-style para criação e debug automático de workflows n8n usando Claude Code via SSH.

## 🎯 Workflows Incluídos

### 1. CAAL - Workflow Creator (Simple)
**Arquivo:** `caal_simple.json`
**Webhook:** `POST /webhook/create-workflow-caal`

Cria workflows automaticamente a partir de descrição natural com loop de auto-correção.

**Features:**
- ✅ Criação automática via Claude Code SSH
- ✅ Auto-teste e correção (até 3 tentativas)
- ✅ Session ID persistence (contexto mantido)
- ✅ Force `availableInMCP: true` para MCP
- ✅ Ativação condicional (verifica se tem trigger)

**Exemplo:**
```bash
curl -X POST http://localhost:5678/webhook/create-workflow-caal \
  -H "Content-Type: application/json" \
  -d '{
    "description": "webhook que adiciona dois números",
    "requirements": "usar operador + direto"
  }'
```

---

### 2. CAAL - Workflow Debugger
**Arquivo Manual:** `caal_debugger.json`
**Arquivo Gerado (CAAL):** `caal_debugger_generated.json` ⭐ **RECOMENDADO**
**Webhook:** `POST /webhook/debug-workflow-caal` (manual) ou `/debug-workflow-caal-v2` (gerado)

Debugs workflows existentes automaticamente.

**Features:**
- ✅ GET workflow existente via API
- ✅ Análise de erro com Claude Code
- ✅ Auto-correção (até 2 tentativas)
- ✅ Mantém `availableInMCP: true`
- ✅ Session persistence para contexto

**Versão Híbrida (Recomendada):**
- Combina arquitetura CAAL-gerada (15 nodes) com configuração manual
- Documentação completa (notes em todos os nodes)
- Credenciais e URLs pré-configuradas
- Ver `COMPARISON.md` para análise detalhada

**Exemplo:**
```bash
curl -X POST http://localhost:5678/webhook/debug-workflow-caal-v2 \
  -H "Content-Type: application/json" \
  -d '{
    "workflow_id": "MYGVeQNhZ8JkGOyu",
    "issue": "Code node retorna undefined em vez de array"
  }'
```

---

## 🔧 Setup

### Requisitos

1. **n8n** com acesso à API
2. **Claude Code CLI** instalado
3. **SSH** configurado (credencial "JARVIS")
4. **Working directory:** `E:\Contexto CLAUDE`

### Credenciais Necessárias

**SSH Password:** JARVIS (ID: V4sZYDRz7ppSgxbk)
- Host: localhost ou servidor SSH
- User: seu usuário
- Working dir: `E:\Contexto CLAUDE`

**n8n API:** HTTP Header Auth
- Configurar em todos os nodes HTTP Request
- Header: `X-N8N-API-KEY`
- Value: sua API key do n8n

### Importar Workflows

1. Acessar n8n: `http://localhost:5678`
2. Import from File
3. Selecionar `caal_simple.json` ou `caal_debugger.json`
4. Configurar credenciais:
   - SSH Password (JARVIS)
   - n8n API (todos os HTTP Request nodes)
5. Ativar workflow

---

## 🏗️ Arquitetura

### Workflow Creator
```
Webhook → Respond → Generate Session
              ↓
        Claude Create (SSH)
              ↓
        Parse Workflow
              ↓
        Create Workflow (API)
              ↓
        Activate (se tem trigger)
              ↓
        Set Counter → Test Loop (max 3x)
```

### Workflow Debugger
```
Webhook → Respond → Generate Session
              ↓
        Get Workflow (API)
              ↓
        Prepare Debug Context
              ↓
        Claude Debug (SSH)
              ↓
        Parse Fixed Workflow
              ↓
        Update Workflow (API)
              ↓
        Set Counter → Test Loop (max 2x)
```

---

## 📝 Código Crítico

### JSON Wrapper Parsing

Claude Code retorna output em camadas:
```
[OK] Banner text...
{"type":"result","result":"actual content with ```json"}
```

**Template de parsing:**
```javascript
const stdout = $json.stdout || '';
let result = stdout;

try {
  const jsonStart = stdout.indexOf('{"type"');
  if (jsonStart > -1) {
    const jsonStr = stdout.substring(jsonStart);
    const parsed = JSON.parse(jsonStr);
    result = parsed.result || stdout;
  }
} catch (e) {}

const jsonMatch = result.match(/```json\s*\n([\s\S]*?)\n```/);
if (!jsonMatch) {
  throw new Error('No JSON found in Claude response');
}

const workflow = JSON.parse(jsonMatch[1]);
```

### Force availableInMCP

```javascript
if (!workflow.settings) {
  workflow.settings = {};
}
workflow.settings.availableInMCP = true;
```

---

## 🐛 Troubleshooting

### PowerShell && Error
**Erro:** `O token '&&' não é um separador de instruções válido`

**Causa:** SSH node roda PowerShell (Windows)

**Fix:** Usar `cwd` parameter em vez de `cd &&`

### JSON Wrapper Parsing Error
**Erro:** `No JSON found in Claude response`

**Causa:** Banner text antes do JSON wrapper

**Fix:** Usar template de parsing com `indexOf('{"type"')`

### Activation Error
**Erro:** `Workflow cannot be activated because it has no trigger node`

**Causa:** Workflow com Manual Trigger

**Fix:** Verificar `triggerCount > 0` antes de ativar

---

## 📊 Testes Realizados

**Workflow Creator - Execução 11995:**
- ✅ Input: "webhook that adds two numbers"
- ✅ Workflow criado: `MYGVeQNhZ8JkGOyu`
- ✅ Erro detectado: Code node syntax
- ✅ Auto-corrigido na 1ª tentativa
- ✅ Ativado com `availableInMCP: true`
- ✅ Tempo: ~120s

---

## 📊 Comparação CAAL vs Manual

**Arquivo:** `COMPARISON.md`

Análise detalhada comparando o workflow debugger criado:
- **Manualmente** (por humano)
- **Via CAAL** (auto-gerado em 3.5 minutos)
- **Híbrido** (melhor dos dois mundos)

**Principais descobertas:**
- CAAL venceu em: arquitetura (15 vs 17 nodes), documentação, manutenibilidade
- Manual venceu em: usabilidade imediata, configuração pronta
- Híbrido combina: estrutura CAAL + configuração manual
- **ROI:** 200% mais rápido (5 min vs 15 min)

Ver detalhes completos em: `COMPARISON.md`

---

## 🔗 Referências

**CAAL Original:**
- GitHub: https://github.com/CoreWorxLab/CAAL
- Docs: `G:\Meu Drive\MEMORIAS E TODO\04-REPO-CAAL-DETALHES.md`

**Sessões de Implementação:**
- Creator: `G:\Meu Drive\MEMORIAS E TODO\SESSAO-2026-01-03-CAAL-WORKFLOW-CREATOR.md`
- Debugger + Comparison: Esta sessão (2026-01-03)

---

## 📄 License

MIT

---

**Status:** ✅ Operacional e testado
**Última atualização:** 2026-01-03
