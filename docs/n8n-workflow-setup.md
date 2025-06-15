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

### 2. Edit Fields (Processamento Inicial)

```yaml
Nome: "Edit Fields"
Assignments:
  - Name: "empresa"
    Value: "={{ $json.body.empresa }}"
  - Name: "timestamp"  
    Value: "={{ $json.body.timestamp }}"
  - Name: "documentTitle"
    Value: "=Análise - {{ $json.body.empresa }} - {{ $now.format('DD HH:mm') }}"
```

### 3. BRAPI-QUOTE (Cotações B3)

```yaml
Nome: "BRAPI-QUOTE"
Method: "GET"
URL: "https://brapi.dev/api/quote/{{ $('Edit Fields').item.json['\"empresa\"'] }}"
Query Parameters:
  token: "[SEU_TOKEN_BRAPI]"
```

### 4. NEWSAPI-NEWS (Notícias)

```yaml
Nome: "NEWSAPI-NEWS"
Method: "GET"
URL: "https://newsapi.org/v2/everything"
Query Parameters:
  q: "={{ $('Edit Fields').item.json['\"empresa\"'] }}"
  language: "pt"
  sortBy: "publishedAt"
  apiKey: "[SUA_CHAVE_NEWSAPI]"
```

### 5. Edit Fields3 (Formatação de Dados)

```yaml
Nome: "Edit Fields3"
Assignments:
  - Name: "formatacaoNews"
    Value: "={{ $json.articles }}"
  - Name: "formatacaoQuote"
    Value: "={{ $('BRAPI-QUOTE').item.json.results }}"
```

### 6. Edit Fields2 (Prompt Gemini)

```yaml
Nome: "Edit Fields2"
Assignments:
  - Name: "contents"
    Value: [PROMPT_COMPLEXO_PARA_GEMINI]
  - Name: "generationConfig"
    Value: [CONFIGURAÇÕES_GEMINI]
```

### 7. HTTP Request (Gemini AI)

```yaml
Nome: "HTTP Request"
Method: "POST"
URL: "https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=[SUA_CHAVE_GEMINI]"
Headers:
  Content-Type: "application/json"
Body Parameters:
  contents: "={{ $('Edit Fields2').item.json.contents }}"
  generationConfig: "={{ $('Edit Fields2').item.json.generationConfig[0] }}"
```

### 8. Edit Fields1 (Processamento da Análise)

```yaml
Nome: "Edit Fields1"
Assignments:
  - Name: "geminiAnalysis"
    Value: "={{ $json.candidates[0].content.parts[0].text }}"
  - Name: "empresa"
    Value: "={{ $('Edit Fields').item.json.empresa }}"
  - Name: "documentTitle"
    Value: "={{ $('Edit Fields').item.json.documentTitle }}"
```

### 9. Google Drive (Criar Pasta)

```yaml
Nome: "Google Drive"
Operation: "Create Folder"
Drive ID: "My Drive"
Parent Folder ID: "1H2kob3X-wW5IUp2C11yngi4_zplU6hT-"
Folder Name: "={{ $('Edit Fields').item.json['\"empresa\"'] }}"
```

### 10. Google Docs (Criar Documento)

```yaml
Nome: "Google Docs"
Operation: "Create Document"
Drive ID: "myDrive"
Folder ID: "={{ $json.id }}"
Title: "={{ $('Edit Fields').item.json['\"documentTitle\"'] }}"
```

### 11. Google Docs1 (Atualizar Documento)

```yaml
Nome: "Google Docs1"
Operation: "Update"
Document URL: "={{ $json.id }}"
Actions:
  - Action: "Insert"
    Text: "={{ $('Edit Fields1').item.json.geminiAnalysis }}"
```

### 12. Respond to Webhook (Resposta)

```yaml
Nome: "Respond to Webhook"
Respond With: "JSON"
Response Body: [JSON_DE_RESPOSTA_ESTRUTURADO]
```

## 📄 JSON Completo do Workflow

```json
{
  "name": "My workflow",
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
      "id": "86ea7ff9-442a-45b2-945d-a294dff9b68f",
      "name": "Webhook",
      "webhookId": "d36b7798-9570-4cce-aaf6-696bd0716133"
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "={\n        \"success\": true,\n        \"message\": \"Análise iniciada com sucesso\",\n        \"empresa\": \"={{ $('Edit Fields').item.json['\"empresa\"'] }}\",\n        \"documentUrl\": \"https://docs.google.com/document/d/{{ $('Google Docs').item.json.id }}/edit\",\n        \"documentId\": \"={{ $('Google Docs').item.json.id }}\",\n        \"timestamp\": \"={{ $('Edit Fields').item.json['\"timestamp\"'] }}\"\n      }",
        "options": {}
      },
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1.4,
      "position": [2040, 0],
      "id": "ff9b5364-13ae-46b1-aec8-f505fe8ee57e",
      "name": "Respond to Webhook"
    },
    {
      "parameters": {
        "assignments": {
          "assignments": [
            {
              "id": "31c0b619-a3a5-4178-93af-b3611c5dbf51",
              "name": "\"empresa\"",
              "value": "={{ $json.body.empresa }}",
              "type": "string"
            },
            {
              "id": "9eb60aab-fdbb-48e8-8946-a065086011f1",
              "name": "\"timestamp\"",
              "value": "={{ $json.body.timestamp }}",
              "type": "string"
            },
            {
              "id": "7c500e36-9cd0-43ab-bea6-71a09c12692c",
              "name": "\"documentTitle\"",
              "value": "=Análise -  {{ $json.body.empresa }} - {{ $now.format('DD HH:mm') }}",
              "type": "string"
            }
          ]
        },
        "options": {}
      },
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.4,
      "position": [220, 0],
      "id": "98a2be06-f408-4c5d-9f7f-77bae149bae4",
      "name": "Edit Fields"
    },
    {
      "parameters": {
        "assignments": {
          "assignments": [
            {
              "id": "8f47ec9a-ea12-4960-8633-f358c73f7975",
              "name": "geminiAnalysis",
              "value": "=={{ $json.candidates[0].content.parts[0].text }}",
              "type": "string"
            },
            {
              "id": "312d859b-3ff4-49e1-884e-13659a70d42b",
              "name": "empresa",
              "value": "={{ $('Edit Fields').item.json.empresa }}",
              "type": "string"
            },
            {
              "id": "ab0c82f3-e2ae-476f-98ad-e6725099b72a",
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
      "position": [1340, 0],
      "id": "9a5df26d-1e3c-44a6-b32f-8068c66b5059",
      "name": "Edit Fields1"
    },
    {
      "parameters": {
        "driveId": "=myDrive",
        "folderId": "={{ $json.id }}",
        "title": "={{ $('Edit Fields').item.json['\"documentTitle\"'] }}"
      },
      "type": "n8n-nodes-base.googleDocs",
      "typeVersion": 2,
      "position": [1680, 0],
      "id": "73687994-9507-4691-b164-897b794c29e8",
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
              "text": "={{ $('Edit Fields1').item.json.geminiAnalysis }}"
            }
          ]
        }
      },
      "type": "n8n-nodes-base.googleDocs",
      "typeVersion": 2,
      "position": [1860, 0],
      "id": "788247b4-160d-4c6b-b913-389b9f01a955",
      "name": "Google Docs1",
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
        "jsonHeaders": "{\n  \n}",
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            {
              "name": "contents",
              "value": "={{ $('Edit Fields2').item.json.contents }}"
            },
            {
              "name": "generationConfig",
              "value": "={{ $('Edit Fields2').item.json.generationConfig[0] }}"
            }
          ]
        },
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [1160, 0],
      "id": "229e38ca-c925-4b57-a674-429628c67f53",
      "name": "HTTP Request"
    },
    {
      "parameters": {
        "assignments": {
          "assignments": [
            {
              "id": "74b68604-4064-412f-b208-2e0ffb25975b",
              "name": "contents",
              "value": "={{   [         {           \"parts\": [             {               \"text\": \"Você é um analista sênior de Investment Banking especializado no mercado brasileiro de capitais.Você receberá dados, que devem ser analisados e tratados, sobre uma empresa brasileira listada na B3, incluindo:\\n1. **Dados Sobre a empresa** - Informações sobre a empresa\\n2. **Dados de Cotação** (formatacaoQuote) - Informações financeiras e de mercado\\n3. **Dados de Notícias** (formatacaoNews) - Notícias recentes sobre a empresa \\n---\\n\\n## DADOS RECEBIDOS:\\n ### 📊 INFORMAÇÕES DE COTAÇÃO:\\n '\"+ $json.formatacaoQuote + \"' \\n ### 📰 NOTÍCIAS COLETADAS: \\n '\"+ $json.formatacaoNews + \"'\\n\\n---\\n\\n## INSTRUÇÕES DE PROCESSAMENTO:\\n\\n### 🔍 Análise das Notícias:\\n1. Avalie TODAS as notícias fornecidas no array articles\\n2. Selecione as 3 MAIS RELEVANTES baseado nos critérios:\\n   - Impacto financeiro direto na empresa\\n   - Relevância operacional (resultados, projetos, mudanças estratégicas)\\n   - Atualidade (priorize notícias mais recentes)\\n   - Fonte confiável (InfoMoney, Valor, Reuters, etc.)\\n3. Descarte notícias sobre:\\n   - Mercado geral sem menção específica da empresa\\n   - Análises técnicas genéricas do Ibovespa\\n   - Notícias repetitivas ou similares\\n\\n### 📈 Análise da Cotação:\\n1. Extraia os dados mais relevantes do JSON de cotação\\n2. Calcule métricas importantes quando possível\\n3. Interprete os indicadores no contexto do mercado brasileiro\\n\\n---\\n\\n## ESTRUTURA OBRIGATÓRIA DO RELATÓRIO:\\n\\n# 📊 ANÁLISE EMPRESARIAL - {NOME_EMPRESA} ({TICKER})\\n\\n**Data da Análise:** {DATA_ATUAL}  \\n**Ticker:** {TICKER}  \\n**Razão Social:** {NOME_COMPLETO}  \\n---\\n\\n## 🏢 PERFIL CORPORATIVO\\n\\n**Fundação:** {Ano e contexto}\\n**Sede:** {Localização}\\n**Principais Atividades:** {Core business detalhado}\\n**Posicionamento:** {Market share, ranking setorial}\\n**Modelo de Negócio:** {Revenue streams principais}\\n\\n---\\n\\n## 💰 RESUMO FINANCEIRO\\n\\n**Preço Atual:** R$ {PRECO_ATUAL}  \\n**Variação Diária:** {VARIACAO_PERCENTUAL}% ({VARIACAO_ABSOLUTA})  \\n**Abertura:** R$ {PRECO_ABERTURA}  \\n**Máxima do Dia:** R$ {MAXIMA_DIA}  \\n**Mínima do Dia:** R$ {MINIMA_DIA}  \\n**Volume:** {VOLUME_FORMATADO}  \\n\\n**Variação 52 Semanas:** R$ {MIN_52S} - R$ {MAX_52S}  \\n**Market Cap:** R$ {MARKET_CAP_FORMATADO}  \\n**P/L:** {PE_RATIO}  \\n**LPA:** R$ {EARNINGS_PER_SHARE}  \\n\\n---\\n\\n## 📈 ANÁLISE DE PERFORMANCE\\n\\n### Contexto Atual\\n{ANÁLISE_DO_PREÇO_ATUAL_EM_RELAÇÃO_AOS_RANGES}\\n\\n### Indicadores Técnicos\\n{INTERPRETAÇÃO_DOS_MÚLTIPLOS_E_MÉTRICAS}\\n\\n---\\n\\n## 📰 NOTÍCIAS MAIS RELEVANTES\\n\\n### 🔥 Notícia 1: {TÍTULO_NOTÍCIA_1}\\n**Data:** {DATA_NOTÍCIA_1}  \\n**Link:** {LINK_NOTÍCIA_1}  \\n**Resumo:** {RESUMO_EXECUTIVO_2_3_LINHAS}  \\n**Impacto:** {ANÁLISE_DO_IMPACTO_PARA_INVESTIDORES}  \\n\\n### 📊 Notícia 2: {TÍTULO_NOTÍCIA_2}  \\n**Data:** {DATA_NOTÍCIA_2}  \\n**Link:** {LINK_NOTÍCIA_2}  \\n**Resumo:** {RESUMO_EXECUTIVO_2_3_LINHAS}  \\n**Impacto:** {ANÁLISE_DO_IMPACTO_PARA_INVESTIDORES}  \\n\\n### 💼 Notícia 3: {TÍTULO_NOTÍCIA_3}\\n**Data:** {DATA_NOTÍCIA_3}  \\n**Link:** {LINK_NOTÍCIA_3}  \\n**Resumo:** {RESUMO_EXECUTIVO_2_3_LINHAS}  \\n**Impacto:** {ANÁLISE_DO_IMPACTO_PARA_INVESTIDORES}  \\n\\n---\\n\\n## 🎯 ANÁLISE INTEGRADA\\n\\n### Síntese dos Dados\\n{COMBINE_INFORMAÇÕES_DE_COTAÇÃO_E_NOTÍCIAS_PARA_CRIAR_UMA_VISÃO_HOLÍSTICA}\\n\\n### Pontos de Atenção\\n- {RISCOS_IDENTIFICADOS_NAS_NOTÍCIAS}\\n- {OPORTUNIDADES_IDENTIFICADAS}\\n- {FATORES_TÉCNICOS_RELEVANTES}\\n\\n### Catalisadores Potenciais\\n- {EVENTOS_FUTUROS_MENCIONADOS_NAS_NOTÍCIAS}\\n- {MARCOS_TÉCNICOS_IMPORTANTES}\\n\\n---\\n\\n##!DISCLAIMERS\\n\\n- **Dados de Mercado:** Baseados na última sessão disponível\\n- **Notícias:** Selecionadas por relevância e impacto potencial  \\n- **Não é Recomendação:** Este documento é apenas informativo\\n- **Verificação:** Recomenda-se confirmação em fontes oficiais\\n- **Risco:** Todo investimento envolve riscos\\n\\n---\\n\\n**Relatório Gerado por:** Sistema Automatizado de Investment Banking  \\n**Modelo:** Gemini 1.5 Flash  \\n**Timestamp:** {TIMESTAMP_GERACAO}\\n\\n---\\n\\n## REGRAS CRÍTICAS:\\n\\n###**DEVE FAZER:**\\n- Usar APENAS dados fornecidos, excluindo a seção \\\"🏢 PERFIL CORPORATIVO\\\" que deve buscar informações sobre\\n- Formatar valores monetários em Real brasileiro (R$)\\n- Datar todas as informações adequadamente\\n- Focar em aspectos relevantes para Investment Banking\\n- Manter linguagem técnica e profissional\\n- Quantificar impactos quando possível\\n\\n###**NÃO DEVE FAZER:**\\n- Inventar dados não presentes\\n- Fazer recomendações de compra/venda\\n- Usar informações de fontes externas aos dados fornecidos\\n- Repetir notícias com conteúdo similar\\n- Incluir análises especulativas sem base nos dados\\n\\n### **CRITÉRIOS PARA SELEÇÃO DE NOTÍCIAS:**\\n\\n**Prioridade Alta:**\\n- Resultados trimestrais/anuais\\n- Dividendos e proventos\\n- Mudanças na gestão\\n- Grandes investimentos/desinvestimentos\\n- Aquisições/fusões\\n- Aprovações regulatórias importantes\\n\\n**Prioridade Média:**\\n- Análises de bancos de investimento\\n- Mudanças de rating\\n- Notícias setoriais específicas\\n- Parcerias estratégicas\\n\\n**Prioridade Baixa:**\\n- Notícias do mercado geral\\n- Análises técnicas do Ibovespa\\n- Comentários genéricos sobre o setor\\n\\n---\\n\\n## EXEMPLO DE FORMATAÇÃO DE VALORES:\\n\\n- **Monetary:** R$ 32,53 (não $32.53)\\n- **Percentage:** +2,46% (não 2.46%)\\n- **Volume:** 74,8 milhões (não 74819900)\\n- **Market Cap:** R$ 400,6 bilhões (não 400583262534)\\n\\n---\\n\\n**EXECUTE A ANÁLISE AGORA COM OS DADOS FORNECIDOS.** Caso os dados não aparecem de forma clara, apresente no final do documento como eles foram apresentados\"             }           ]         }       ] }}",
              "type": "array"
            },
            {
              "id": "11f406b1-7cfa-43c4-9aa4-ad611699aff6",
              "name": "=generationConfig",
              "value": "={{ [{         \"temperature\": 0.3,         \"maxOutputTokens\": 2048,         \"topP\": 0.8,         \"topK\": 40       }] }}",
              "type": "array"
            }
          ]
        },
        "options": {}
      },
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.4,
      "position": [980, 0],
      "id": "af1a6685-d118-4a44-8d1b-131878de20cd",
      "name": "Edit Fields2"
    },
    {
      "parameters": {
        "resource": "folder",
        "name": "={{ $('Edit Fields').item.json['\"empresa\"'] }}",
        "driveId": {
          "__rl": true,
          "mode": "list",
          "value": "My Drive"
        },
        "folderId": {
          "__rl": true,
          "value": "[SUBSTITUA_PELO_ID_DA_SUA_PASTA]",
          "mode": "list",
          "cachedResultName": "projetoEstagio",
          "cachedResultUrl": "https://drive.google.com/drive/folders/[ID_DA_PASTA]"
        },
        "options": {}
      },
      "type": "n8n-nodes-base.googleDrive",
      "typeVersion": 3,
      "position": [1500, 0],
      "id": "97de815a-efc1-42c7-a06e-2dd8ded95a83",
      "name": "Google Drive",
      "credentials": {
        "googleDriveOAuth2Api": {
          "id": "[SUBSTITUA_PELO_ID_DA_SUA_CREDENCIAL]",
          "name": "Google Drive account"
        }
      }
    },
    {
      "parameters": {
        "url": "= https://brapi.dev/api/quote/{{ $('Edit Fields').item.json['\"empresa\"'] }}",
        "sendQuery": true,
        "queryParameters": {
          "parameters": [
            {
              "name": "token",
              "value": "[SUBSTITUA_PELO_SEU_TOKEN_BRAPI]"
            }
          ]
        },
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [440, 0],
      "id": "9a618cb4-cdd0-41f0-8494-224528de3348",
      "name": "BRAPI-QUOTE"
    },
    {
      "parameters": {
        "url": "=https://newsapi.org/v2/everything",
        "sendQuery": true,
        "queryParameters": {
          "parameters": [
            {
              "name": "q",
              "value": "={{ $('Edit Fields').item.json['\"empresa\"'] }}"
            },
            {
              "name": "language",
              "value": "=pt"
            },
            {
              "name": "sortBy",
              "value": "publishedAt"
            },
            {
              "name": "apiKey",
              "value": "[SUBSTITUA_PELA_SUA_CHAVE_NEWSAPI]"
            }
          ]
        },
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [640, 0],
      "id": "c7197e56-a5ce-4313-9de1-9605d772106b",
      "name": "NEWSAPI-NEWS"
    },
    {
      "parameters": {
        "assignments": {
          "assignments": [
            {
              "id": "cef7bd22-ee59-4379-b8cd-3a1265e6ee6d",
              "name": "formatacaoNews",
              "value": "={{ $json.articles }}",
              "type": "string"
            },
            {
              "id": "e334ec39-e94d-4ad7-bef7-25c30b2485a6",
              "name": "formatacaoQuote",
              "value": "={{ $('BRAPI-QUOTE').item.json.results }}",
              "type": "string"
            }
          ]
        },
        "options": {}
      },
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.4,
      "position": [820, 0],
      "id": "492e7c9d-654a-484c-adf6-601299b33fa6",
      "name": "Edit Fields3"
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
            "node": "BRAPI-QUOTE",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Edit Fields1": {
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
            "node": "Google Docs1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Google Docs1": {
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
    "HTTP Request": {
      "main": [
        [
          {
            "node": "Edit Fields1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Edit Fields2": {
      "main": [
        [
          {
            "node": "HTTP Request",
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
    },
    "BRAPI-QUOTE": {
      "main": [
        [
          {
            "node": "NEWSAPI-NEWS",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "NEWSAPI-NEWS": {
      "main": [
        [
          {
            "node": "Edit Fields3",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Edit Fields3": {
      "main": [
        [
          {
            "node": "Edit Fields2",
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
  "versionId": "e608c1c4-f1cb-4752-abad-5d737da39203",
  "meta": {
    "templateCredsSetupCompleted": true,
    "instanceId": "f703af88e39c1bdf94c33361d2b3287ee191ca47808bb38bf2428e19901c623a"
  },
  "id": "J5XzbIICEwZTFscv",
  "tags": []
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

# BRAPI Token
[SUBSTITUA_PELO_SEU_TOKEN_BRAPI] → seu_token_brapi_real

# NewsAPI Key
[SUBSTITUA_PELA_SUA_CHAVE_NEWSAPI] → sua_chave_newsapi_real

# Google Drive Folder ID  
[SUBSTITUA_PELO_ID_DA_SUA_PASTA] → id_da_pasta_google_drive

# Credencial IDs (automático via interface)
[SUBSTITUA_PELO_ID_DA_SUA_CREDENCIAL] → selecionado_automaticamente
```

### 3. Fluxo de Execução do Workflow

O workflow atual funciona da seguinte forma:

1. **Webhook** → Recebe dados da requisição
2. **Edit Fields** → Processa dados iniciais (empresa, timestamp, título)
3. **BRAPI-QUOTE** → Busca cotação da empresa na B3
4. **NEWSAPI-NEWS** → Busca notícias recentes sobre a empresa
5. **Edit Fields3** → Formata dados de cotação e notícias
6. **Edit Fields2** → Prepara prompt complexo para o Gemini
7. **HTTP Request** → Envia análise para Gemini AI
8. **Edit Fields1** → Processa resposta do Gemini
9. **Google Drive** → Cria pasta para a empresa
10. **Google Docs** → Cria documento na pasta
11. **Google Docs1** → Insere análise no documento
12. **Respond to Webhook** → Retorna resposta com link do documento

### 4. Ativar Workflow

1. **Toggle "Active"** no canto superior direito
2. **Verificar webhook URL** gerada
3. **Copiar URL** para configurar no frontend

### 5. Teste Manual

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
3. **Analise erros** clicando nas execuções que falharam

### 2. Métricas Importantes

| Métrica | Valor Normal | Ação se Anormal |
|---------|--------------|-----------------|
| Tempo de execução | 60-120 segundos | Verificar APIs |
| Taxa de sucesso | >95% | Revisar credenciais |
| Uso de quota Gemini | <80% | Monitorar limites |
| Resposta BRAPI | <2 segundos | Verificar token |
| Resposta NewsAPI | <3 segundos | Verificar cota |

### 3. Pontos de Falha Comuns

#### BRAPI API
- Verificar se token está válido
- Confirmar se ticker existe na B3
- Monitorar limites de requisições

#### NewsAPI 
- Verificar cota diária (1000 requisições gratuitas)
- Confirmar se API key está ativa
- Analisar se há notícias em português

#### Gemini AI
- Monitorar quota de requisições por minuto
- Verificar se prompt não excede limites
- Confirmar se API key tem permissões

### 4. Troubleshooting Específico

#### Erro "No articles found"
```
Causa: NewsAPI não encontrou notícias
Solução: Verificar se empresa tem cobertura de mídia
```

#### Erro "Invalid ticker"
```
Causa: BRAPI não reconhece o ticker
Solução: Verificar se ticker está correto (ex: PETR4)
```

#### Erro "Token limit exceeded"
```
Causa: Prompt do Gemini muito longo
Solução: Reduzir dados de entrada ou aumentar maxOutputTokens
```

## 🔧 Customizações Avançadas

### 1. Adicionar Mais Fontes de Dados

```javascript
// Novo nó HTTP Request - Yahoo Finance
{
  "url": "https://query1.finance.yahoo.com/v8/finance/chart/{{ $json.empresa }}.SA",
  "method": "GET"
}
```

### 2. Melhorar Filtro de Notícias

```javascript
// Filtrar notícias por relevância
const filteredNews = $json.articles.filter(article => {
  const title = article.title.toLowerCase();
  const empresa = $('Edit Fields').item.json.empresa.toLowerCase();
  return title.includes(empresa) || title.includes('dividendo') || title.includes('resultado');
});
```

### 3. Adicionar Análise Técnica

```javascript
// Calcular indicadores técnicos
const quote = $('BRAPI-QUOTE').item.json.results[0];
const rsi = calculateRSI(quote.historicalData);
const ma20 = calculateMovingAverage(quote.historicalData, 20);
```

---

<div align="center">

**Workflow n8n atualizado configurado com sucesso!** 🎉  
O sistema agora inclui integração completa com BRAPI e NewsAPI

</div>