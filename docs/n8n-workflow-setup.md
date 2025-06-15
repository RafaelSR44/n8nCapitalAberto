# ⚙️ Setup do Workflow n8n

## 📚 Índice

1. [Importação do Workflow](#-importação-do-workflow)
2. [Configuração de Credenciais](#-configuração-de-credenciais)
3. [Configuração dos Nós](#-configuração-dos-nós)
4. [JSON Completo do Workflow](#-json-completo-do-workflow)
5. [Ativação e Testes](#-ativação-e-testes)
6. [Monitoramento](#-monitoramento)

## 📥 Importação do Workflow

### 1. Método via Interface n8n

1. **Acesse seu n8n**
   - n8n Cloud: `https://[sua-instancia].app.n8n.cloud`
   - Self-hosted: `https://[seu-dominio]`

2. **Importar workflow**
   - Clique em "+" para novo workflow
   - No menu superior, clique nos "⋮" (três pontos)
   - Selecione "Import from file" ou "Import from clipboard"

3. **Colar JSON**
   - Copie o [JSON completo](#-json-completo-do-workflow) abaixo
   - Cole no campo de importação
   - Clique em "Import"

### 2. Método via CLI (Self-hosted)

```bash
# Salvar JSON em arquivo
cat > investment-banking-workflow.json << 'EOF'
[COLE_O_JSON_AQUI]
EOF

# Importar via API
curl -X POST 'http://localhost:5678/rest/workflows/import' \
  -H 'Content-Type: application/json' \
  -d @investment-banking-workflow.json
```

## 🔐 Configuração de Credenciais

### 1. Google Docs OAuth2 API

```yaml
Nome da Credencial: "Google Docs account"
Tipo: "Google Docs OAuth2 API"
Configurações:
  Client ID: [SEU_CLIENT_ID_GOOGLE]
  Client Secret: [SEU_CLIENT_SECRET_GOOGLE]
  Scope: "https://www.googleapis.com/auth/documents"
```

**Passos**:
1. Clique em "Credenciais" no menu lateral
2. "Adicionar credencial" → "Google Docs OAuth2 API"
3. Preencha as informações acima
4. Clique "Conectar minha conta"
5. Autorize no popup do Google

### 2. Google Drive OAuth2 API

```yaml
Nome da Credencial: "Google Drive account"
Tipo: "Google Drive OAuth2 API"
Configurações:
  Client ID: [SEU_CLIENT_ID_GOOGLE]
  Client Secret: [SEU_CLIENT_SECRET_GOOGLE]
  Scope: "https://www.googleapis.com/auth/drive.file"
```

**Passos**:
1. Repita processo similar ao Google Docs
2. Use as mesmas credenciais OAuth
3. Autorize no Google

## 🔧 Configuração dos Nós

### 1. Webhook (Nó de Entrada)

```yaml
Nome: "Webhook"
Path: "empresa-research"
HTTP Method: "POST"
Response Mode: "Response Node"
```

**Importante**: Após salvar, copie a URL do webhook gerada

### 2. Google Drive (Criar Pasta)

```yaml
Nome: "Google Drive"
Operation: "Create Folder"
Drive ID: "My Drive"
Parent Folder ID: "[ID_DA_SUA_PASTA_PAI]"
Folder Name: "={{ $('Edit Fields').item.json['\"empresa\"'] }}"
```

**Como obter ID da pasta pai**:
1. Acesse Google Drive
2. Navegue até a pasta desejada
3. Copie ID da URL: `https://drive.google.com/drive/folders/ID_AQUI`

### 3. HTTP Request (Gemini AI)

```yaml
Nome: "HTTP Request"
Method: "POST"
URL: "https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=[SUA_CHAVE_GEMINI]"
Headers:
  Content-Type: "application/json"
Body: "JSON Object"
```

**Substituir chave**:
- Localize `[SUA_CHAVE_GEMINI]`
- Substitua por `[SUA_CHAVE_GEMINI]`

### 4. BRAPI Quote 

```yaml
Nome: "BRAPI-QUOTE"
Method: "GET"
URL: "https://brapi.dev/api/quote/{{ $json['\"empresa\"'] }}"
Query Parameters:
  token: "[SEU_TOKEN_BRAPI]"
```

### 5. NewsAPI 

```yaml
Nome: "NEWSAPI-NEWS"
Method: "GET"
URL: "https://newsapi.org/v2/everything"
Query Parameters:
  q: "{{ $json['\"empresa\"'] }}"
  language: "pt"
  sortBy: "publishedAt"
  apiKey: "[SUA_CHAVE_NEWSAPI]"
```

## 📄 JSON Completo do Workflow

```json
{
  "name": "Investment Banking - Empresa Research",
  "nodes": [
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "empresa-research",
        "responseMode": "responseNode",
        "options": {}
      },
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [0, 0],
      "id": "webhook-start",
      "name": "Webhook",
      "webhookId": "empresa-research-webhook"
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "={\n        \"success\": true,\n        \"message\": \"Análise iniciada com sucesso\",\n        \"empresa\": \"={{ $('Edit Fields').item.json['empresa'] }}\",\n        \"documentUrl\": \"https://docs.google.com/document/d/{{ $('Google Docs').item.json.id }}/edit\",\n        \"documentId\": \"={{ $('Google Docs').item.json.id }}\",\n        \"timestamp\": \"={{ $('Edit Fields').item.json['timestamp'] }}\"\n      }",
        "options": {}
      },
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1.4,
      "position": [1540, 0],
      "id": "respond-webhook",
      "name": "Respond to Webhook"
    },
    {
      "parameters": {
        "assignments": {
          "assignments": [
            {
              "id": "empresa-assignment",
              "name": "empresa",
              "value": "={{ $json.body.empresa }}",
              "type": "string"
            },
            {
              "id": "timestamp-assignment",
              "name": "timestamp",
              "value": "={{ $json.body.timestamp }}",
              "type": "string"
            },
            {
              "id": "document-title-assignment",
              "name": "documentTitle",
              "value": "=Análise - {{ $json.body.empresa }} - {{ $now.format('DD/MM HH:mm') }}",
              "type": "string"
            }
          ]
        },
        "options": {}
      },
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.4,
      "position": [220, 0],
      "id": "edit-fields",
      "name": "Edit Fields"
    },
    {
      "parameters": {
        "assignments": {
          "assignments": [
            {
              "id": "gemini-analysis",
              "name": "geminiAnalysis",
              "value": "={{ $json.candidates[0].content.parts[0].text }}",
              "type": "string"
            },
            {
              "id": "empresa-field",
              "name": "empresa",
              "value": "={{ $('Edit Fields').item.json.empresa }}",
              "type": "string"
            },
            {
              "id": "document-title-field",
              "name": "documentTitle",
              "value": "={{ $('Edit Fields').item.json.documentTitle }}",
              "type": "string"
            }
          ]
        },
        "options": {}
      },
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.4,
      "position": [840, 0],
      "id": "edit-fields-analysis",
      "name": "Edit Fields - Analysis"
    },
    {
      "parameters": {
        "driveId": "myDrive",
        "folderId": "={{ $json.id }}",
        "title": "={{ $('Edit Fields').item.json['documentTitle'] }}"
      },
      "type": "n8n-nodes-base.googleDocs",
      "typeVersion": 2,
      "position": [1180, 0],
      "id": "google-docs-create",
      "name": "Google Docs",
      "credentials": {
        "googleDocsOAuth2Api": {
          "id": "[SUBSTITUA_PELO_ID_DA_SUA_CREDENCIAL]",
          "name": "Google Docs account"
        }
      }
    },
    {
      "parameters": {
        "operation": "update",
        "documentURL": "={{ $json.id }}",
        "actionsUi": {
          "actionFields": [
            {
              "action": "insert",
              "text": "=# Análise de Empresa - {{ $('Edit Fields').item.json.empresa }}\\n\\n**Data da Análise:** {{ $now.format('DD/MM/YYYY HH:mm:ss') }}\\n**Gerado por:** Sistema Automatizado de Investment Banking\\n\\n---\\n\\n{{ $('Edit Fields - Analysis').item.json.geminiAnalysis }}\\n\\n---\\n\\n**Disclaimers:**\\n- Esta análise foi gerada automaticamente\\n- Os dados podem estar desatualizados\\n- Recomenda-se verificação manual para decisões de investimento\\n- Este documento é para fins informativos apenas\\n\\n*Powered by n8n + Gemini AI*"
            }
          ]
        }
      },
      "type": "n8n-nodes-base.googleDocs",
      "typeVersion": 2,
      "position": [1360, 0],
      "id": "google-docs-update",
      "name": "Google Docs - Update",
      "credentials": {
        "googleDocsOAuth2Api": {
          "id": "[SUBSTITUA_PELO_ID_DA_SUA_CREDENCIAL]",
          "name": "Google Docs account"
        }
      }
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=[SUBSTITUA_PELA_SUA_CHAVE_GEMINI]",
        "sendHeaders": true,
        "specifyHeaders": "json",
        "jsonHeaders": "{}",
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            {
              "name": "contents",
              "value": "={{ $json.contents }}"
            },
            {
              "name": "generationConfig",
              "value": "={{ $json.generationConfig[0] }}"
            }
          ]
        },
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [660, 0],
      "id": "http-request-gemini",
      "name": "HTTP Request - Gemini"
    },
    {
      "parameters": {
        "assignments": {
          "assignments": [
            {
              "id": "contents-assignment",
              "name": "contents",
              "value": "={{   [         {           \"parts\": [             {               \"text\": \"Você é um analista sênior de Investment Banking especializado no mercado brasileiro de capitais. Preciso de uma análise profissional e detalhada sobre a empresa '\"+ $json['empresa']+ \"'.\\n\\n## INSTRUÇÕES ESPECÍFICAS:\\n- Foque APENAS em empresas brasileiras de capital aberto (listadas na B3)\\n- Use linguagem técnica apropriada para profissionais de IB\\n- Seja factual e preciso com dados financeiros\\n- Indique claramente quando informações estão desatualizadas\\n- Se a empresa não for brasileira/pública, informe imediatamente\\n\\n## ESTRUTURA OBRIGATÓRIA:\\n\\n### 📊 RESUMO EXECUTIVO\\n**Setor:** [Classificação setorial B3]\\n**Ticker:** [Código de negociação]\\n**Status:** [Ativo/Suspenso/etc]\\n**Market Cap:** [Se disponível]\\n\\n### 🏢 PERFIL CORPORATIVO\\n**Fundação:** [Ano e contexto]\\n**Sede:** [Localização]\\n**Principais Atividades:** [Core business detalhado]\\n**Posicionamento:** [Market share, ranking setorial]\\n**Modelo de Negócio:** [Revenue streams principais]\\n\\n### 📈 INDICADORES FINANCEIROS (Últimos dados disponíveis)\\n**Preço da Ação:** [Valor e data]\\n**Variação 52 semanas:** [Min/Max]\\n**Volume médio:** [Se disponível]\\n**P/L:** [Se aplicável]\\n**ROE:** [Se disponível]\\n**Dividend Yield:** [Se aplicável]\\n\\n### 📰 ÚLTIMAS NOTÍCIAS RELEVANTES (2-3 mais recentes)\\nPara cada notícia:\\n**[Data] - Título**\\n- Resumo: [2-3 linhas sobre impacto nos negócios]\\n- Relevância: [Alta/Média/Baixa para investidores]\\n- Fonte: [Se disponível]\\n\\n### 🎯 ANÁLISE TÉCNICA\\n**Pontos Fortes:**\\n- [3-4 pontos específicos]\\n\\n**Riscos Identificados:**\\n- [3-4 riscos setoriais/específicos]\\n\\n**Catalisadores Potenciais:**\\n- [Próximos eventos relevantes]\\n\\n### 🔮 OUTLOOK\\n**Perspectiva Trimestral:** [Próximo trimestre]\\n**Perspectiva Anual:** [12 meses]\\n**Consenso de Mercado:** [Se houver informação disponível]\\n\\n## AVISOS IMPORTANTES:\\n- Dados podem estar desatualizados\\n- Não constitui recomendação de investimento\\n- Verificar informações em fontes oficiais\\n- Para uso informativo apenas\\n\\n---\\n*Análise gerada por IA - Validação manual recomendada*\"             }           ]         }       ] }}",
              "type": "array"
            },
            {
              "id": "generation-config",
              "name": "generationConfig",
              "value": "={{ [{         \"temperature\": 0.3,         \"maxOutputTokens\": 2048,         \"topP\": 0.8,         \"topK\": 40       }] }}",
              "type": "array"
            }
          ]
        },
        "options": {}
      },
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.4,
      "position": [440, 0],
      "id": "edit-fields-prompt",
      "name": "Edit Fields - Prompt"
    },
    {
      "parameters": {
        "resource": "folder",
        "name": "={{ $('Edit Fields').item.json['empresa'] }}",
        "driveId": {
          "__rl": true,
          "mode": "list",
          "value": "My Drive"
        },
        "folderId": {
          "__rl": true,
          "value": "[SUBSTITUA_PELO_ID_DA_SUA_PASTA]",
          "mode": "list",
          "cachedResultName": "Investment Banking Reports",
          "cachedResultUrl": "https://drive.google.com/drive/folders/[ID_DA_PASTA]"
        },
        "options": {}
      },
      "type": "n8n-nodes-base.googleDrive",
      "typeVersion": 3,
      "position": [1000, 0],
      "id": "google-drive-folder",
      "name": "Google Drive",
      "credentials": {
        "googleDriveOAuth2Api": {
          "id": "[SUBSTITUA_PELO_ID_DA_SUA_CREDENCIAL]",
          "name": "Google Drive account"
        }
      }
    }
  ],
  "pinData": {},
  "connections": {
    "Webhook": {
      "main": [
        [
          {
            "node": "Edit Fields",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Edit Fields": {
      "main": [
        [
          {
            "node": "Edit Fields - Prompt",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Edit Fields - Analysis": {
      "main": [
        [
          {
            "node": "Google Drive",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Google Docs": {
      "main": [
        [
          {
            "node": "Google Docs - Update",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Google Docs - Update": {
      "main": [
        [
          {
            "node": "Respond to Webhook",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "HTTP Request - Gemini": {
      "main": [
        [
          {
            "node": "Edit Fields - Analysis",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Edit Fields - Prompt": {
      "main": [
        [
          {
            "node": "HTTP Request - Gemini",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Google Drive": {
      "main": [
        [
          {
            "node": "Google Docs",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "active": false,
  "settings": {
    "executionOrder": "v1"
  },
  "versionId": "investment-banking-v1.0",
  "meta": {
    "templateCredsSetupCompleted": true,
    "instanceId": "investment-banking-automation"
  },
  "id": "investment-banking-workflow",
  "tags": ["investment-banking", "automation", "b3", "analysis"]
}
```

## ✅ Ativação e Testes

### 1. Configurar Credenciais

Após importação, você precisa reconfigurar as credenciais:

1. **Clique em cada nó que usa credenciais**
2. **Selecione a credencial correta**
3. **Teste a conexão**

### 2. Substituir Valores Placeholder

Procure e substitua os seguintes valores no workflow:

```bash
# Gemini API Key
[SUBSTITUA_PELA_SUA_CHAVE_GEMINI] → sua_chave_gemini_real

# Google Drive Folder ID  
[SUBSTITUA_PELO_ID_DA_SUA_PASTA] → id_da_pasta_google_drive

# Credencial IDs (automático via interface)
[SUBSTITUA_PELO_ID_DA_SUA_CREDENCIAL] → selecionado_automaticamente
```

### 3. Ativar Workflow

1. **Toggle "Active"** no canto superior direito
2. **Verificar webhook URL** gerada
3. **Copiar URL** para configurar no frontend

### 4. Teste Manual

```bash
# Via curl
curl -X POST "https://[sua-instancia].app.n8n.cloud/webhook/empresa-research" \
  -H "Content-Type: application/json" \
  -d '{
    "empresa": "PETR4",
    "timestamp": "2024-01-01T00:00:00Z",
    "source": "manual_test"
  }'
```

**Resposta esperada**:
```json
{
  "success": true,
  "message": "Análise iniciada com sucesso",
  "empresa": "PETR4",
  "documentUrl": "https://docs.google.com/document/d/[DOC_ID]/edit",
  "documentId": "[DOC_ID]",
  "timestamp": "2024-01-01T00:00:00Z"
}
```

## 📊 Monitoramento

### 1. Logs de Execução

1. **Acesse "Executions"** no menu lateral
2. **Monitore execuções** em tempo real
3. **Analise erros** clicando nas execuções falharam

### 2. Métricas Importantes

| Métrica | Valor Normal | Ação se Anormal |
|---------|--------------|-----------------|
| Tempo de execução | 30-90 segundos | Verificar APIs |
| Taxa de sucesso | >95% | Revisar credenciais |
| Uso de quota Gemini | <80% | Monitorar limites |

### 3. Alertas Recomendados

```yaml
# Via webhook (opcional)
Erro_execução:
  webhook: "https://hooks.slack.com/[seu-webhook]"
  
Quota_excedida:
  email: "[seu-email]"
  
Sucesso_documento:
  log: "Documento criado: {{ $json.documentId }}"
```

## 🔧 Customizações Avançadas

### 1. Adicionar Mais Fontes de Dados

```javascript
// Novo nó HTTP Request - Alpha Vantage
{
  "url": "https://www.alphavantage.co/query",
  "parameters": {
    "function": "TIME_SERIES_DAILY",
    "symbol": "{{ $json.empresa }}.SAO",
    "apikey": "[SUA_CHAVE_ALPHA_VANTAGE]"
  }
}
```

### 2. Melhorar Prompt do Gemini

```text
// Prompt customizado para setor específico
"Analise a empresa {{ $json.empresa }} focando especificamente no setor de {{ $json.setor }}. 
Considere:
- Regulamentações específicas do setor
- Comparação com peers setoriais
- Impactos macroeconômicos relevantes
- KPIs específicos do setor"
```

### 3. Adicionar Validações

```javascript
// Nó de validação antes do processamento
if (!$json.empresa || $json.empresa.length < 4) {
  throw new Error('Ticker inválido');
}

if (!$json.empresa.match(/^[A-Z]{4}[0-9]?$/)) {
  throw new Error('Formato de ticker inválido');
}
```

## 🚨 Backup e Segurança

### 1. Backup do Workflow

```bash
# Exportar workflow via API
curl -X GET 'https://[sua-instancia].app.n8n.cloud/rest/workflows/[workflow-id]' \
  -H 'Authorization: Bearer [seu-token]' > backup-workflow.json
```

### 2. Versionamento

1. **Sempre fazer backup** antes de mudanças
2. **Usar tags** para versionar workflows
3. **Documentar mudanças** no campo "Notes"

### 3. Segurança

- ✅ Usar HTTPS para todos endpoints
- ✅ Configurar credenciais com escopo mínimo
- ✅ Monitorar logs de acesso
- ✅ Rotacionar API keys periodicamente

---

<div align="center">

**Workflow n8n configurado com sucesso!** 🎉  
Agora você pode prosseguir para os [testes finais](guia-implementacao.md#testes-e-validação)

</div>