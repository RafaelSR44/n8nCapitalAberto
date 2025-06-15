# 📋 Guia de Implementação Completo

## 📚 Índice

1. [Pré-requisitos](#-pré-requisitos)
2. [Configuração Google Cloud](#-configuração-google-cloud)
3. [Configuração n8n](#-configuração-n8n)
4. [APIs e Integrações](#-apis-e-integrações)
5. [Deploy Frontend](#-deploy-frontend)
6. [Testes e Validação](#-testes-e-validação)
7. [Troubleshooting](#-troubleshooting)

## 🔧 Pré-requisitos

### Contas Necessárias
- ✅ Conta Google Cloud Platform (gratuita)
- ✅ Conta n8n Cloud ou instância self-hosted
- ✅ Conta BRAPI (gratuita)
- ⚪ Conta NewsAPI (gratuita)

### Conhecimentos Técnicos
- Básico em JSON e APIs REST
- Familiaridade com Google Cloud Console
- Conhecimento básico do n8n

## ☁️ Configuração Google Cloud

### 1. Criar Projeto no Google Cloud

1. Acesse [Google Cloud Console](https://console.cloud.google.com/)
2. Clique em "Selecionar projeto" → "Novo projeto"
3. Nome do projeto: `investment-banking-automation`
4. Clique em "Criar"

### 2. Habilitar APIs Necessárias

⚠️ **IMPORTANTE**: Habilite todas as APIs antes de criar credenciais

#### 2.1 Google Docs API
```bash
# Via Console Web
1. Navegue para "APIs e Serviços" → "Biblioteca"
2. Pesquise "Google Docs API"
3. Clique em "Google Docs API"
4. Clique em "ATIVAR"
```

#### 2.2 Google Drive API
```bash
# Via Console Web
1. Na biblioteca de APIs, pesquise "Google Drive API"
2. Clique em "Google Drive API"
3. Clique em "ATIVAR"
```

#### 2.3 Generative Language API (Gemini)
```bash
# Via Console Web
1. Pesquise "Generative Language API"
2. Clique em "Generative Language API"
3. Clique em "ATIVAR"
```

#### 2.4 Cloud AI Companion API
```bash
# Via Console Web
1. Pesquise "Cloud AI Companion API"
2. Clique na API encontrada
3. Clique em "ATIVAR"
```

### 3. Configurar OAuth 2.0

#### 3.1 Criar Credenciais OAuth

1. Vá para "APIs e Serviços" → "Credenciais"
2. Clique em "Criar credenciais" → "ID do cliente OAuth 2.0"
3. Tipo de aplicação: "Aplicação da Web"
4. Nome: "n8n Investment Banking"

#### 3.2 URLs Autorizadas

```bash
# URIs de redirecionamento autorizados (n8n Cloud)
https://[SUA_INSTANCIA].app.n8n.cloud/rest/oauth2-credential/callback

# URIs de redirecionamento autorizados (self-hosted)
https://[SEU_DOMINIO]/rest/oauth2-credential/callback
```

#### 3.3 Configurar Tela de Consentimento

1. Vá para "Tela de consentimento OAuth"
2. Tipo: "Externo" (para testes)
3. Informações do app:
   - Nome: "Investment Banking Automation"
   - Email de suporte: [seu-email]
   - Domínio autorizado: [seu-dominio]

4. Escopos necessários:
   ```
   https://www.googleapis.com/auth/documents
   https://www.googleapis.com/auth/drive.file
   https://www.googleapis.com/auth/drive.metadata.readonly
   ```

### 4. Criar API Key para Gemini

1. Em "Credenciais", clique "Criar credenciais" → "Chave de API"
2. Anote a chave: `[INSIRA_A_CHAVE_DA_API_DO_GOOGLE]`
3. **Importante**: Restringir a chave:
   - Restrições de aplicação: "Nenhuma"
   - Restrições de API: Selecionar "Generative Language API"

## ⚙️ Configuração n8n

### 1. Credenciais Google Docs

1. No n8n, vá para "Credenciais"
2. Clique em "Adicionar credencial"
3. Busque "Google Docs OAuth2 API"
4. Preencha:
   - **Client ID**: `[SEU_CLIENT_ID_GOOGLE]`
   - **Client Secret**: `[SEU_CLIENT_SECRET_GOOGLE]`
   - **Scope**: `https://www.googleapis.com/auth/documents`

5. Clique em "Conectar minha conta"
6. Autorize no Google
7. Teste a conexão

### 2. Credenciais Google Drive

1. Adicione nova credencial "Google Drive OAuth2 API"
2. Use as mesmas credenciais OAuth:
   - **Client ID**: `[SEU_CLIENT_ID_GOOGLE]`
   - **Client Secret**: `[SEU_CLIENT_SECRET_GOOGLE]`
   - **Scope**: `https://www.googleapis.com/auth/drive.file`

3. Conecte e autorize

### 3. Criar Pasta no Google Drive

1. Acesse [Google Drive](https://drive.google.com)
2. Crie uma pasta: "Investment Banking Reports"
3. Anote o ID da pasta (última parte da URL):
   ```
   https://drive.google.com/drive/folders/1H2kob3X-wW5IUp2C11yngi4_zplU6hT-
   ID da pasta: 1H2kob3X-wW5IUp2C11yngi4_zplU6hT-
   ```

## 🔗 APIs e Integrações

### 1. BRAPI (Cotações B3)

#### 1.1 Criar Conta BRAPI
1. Acesse [BRAPI.dev](https://brapi.dev/)
2. Registre-se gratuitamente
3. Confirme email
4. Acesse dashboard e copie seu token

#### 1.2 Testar API
```bash
curl "https://brapi.dev/api/quote/PETR4?token=[SEU_TOKEN_BRAPI]"
```

### 2. NewsAPI

#### 2.1 Registrar no NewsAPI
1. Acesse [NewsAPI.org](https://newsapi.org/)
2. Crie conta gratuita
3. Copie sua API key

#### 2.2 Testar API
```bash
curl "https://newsapi.org/v2/everything?q=Petrobras&language=pt&apiKey=[SUA_CHAVE_NEWSAPI]"
```

### 3. Gemini AI

#### 3.1 Testar Gemini API
```bash
curl -X POST \
  "https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=[SUA_CHAVE_GEMINI]" \
  -H "Content-Type: application/json" \
  -d '{
    "contents": [{
      "parts": [{"text": "Analise brevemente a empresa Petrobras"}]
    }]
  }'
```

## 🌐 Deploy Frontend

### 1. Configurar Frontend

1. Abra `frontend/index.html`
2. Localize a linha:
   ```javascript
   const N8N_WEBHOOK_URL = 'https://rafaelsr44.app.n8n.cloud/webhook-test/empresa-research';
   ```
3. Substitua pela sua URL do webhook n8n

### 2. Opções de Deploy

#### 2.1 Netlify (Recomendado)
1. Faça fork deste repositório
2. Conecte ao [Netlify](https://netlify.com)
3. Deploy automático da pasta `frontend/`

#### 2.2 Vercel
1. Instale Vercel CLI: `npm i -g vercel`
2. Na pasta frontend: `vercel --prod`

#### 2.3 GitHub Pages
1. Faça upload da pasta `frontend/` para repositório
2. Ative GitHub Pages nas configurações

#### 2.4 Servidor Próprio
```bash
# Apache/Nginx
cp frontend/index.html /var/www/html/
```

## ✅ Testes e Validação

### 1. Teste Individual dos Componentes

#### 1.1 Testar Webhook n8n
```bash
curl -X POST "https://[SUA_INSTANCIA].app.n8n.cloud/webhook/empresa-research" \
  -H "Content-Type: application/json" \
  -d '{"empresa":"PETR4","timestamp":"2024-01-01T00:00:00Z"}'
```

#### 1.2 Verificar Google Drive
1. Acesse sua pasta no Google Drive
2. Deve aparecer uma subpasta com nome da empresa
3. Dentro deve ter um documento Google Docs

#### 1.3 Testar Frontend
1. Abra o frontend no navegador
2. Digite "Petrobras" no campo de busca
3. Selecione "PETR4"
4. Clique em "Iniciar Pesquisa"
5. Aguarde o link do documento

### 2. Teste End-to-End

```javascript
// Teste completo via frontend
1. Buscar empresa: "Vale" → VALE3
2. Aguardar processamento (2-3 minutos)
3. Verificar documento gerado
4. Validar conteúdo da análise
```

### 3. Validar Conteúdo Gerado

O documento deve conter:
- ✅ Resumo executivo com ticker e setor
- ✅ Dados financeiros (preço, P/L, etc.)
- ✅ Notícias recentes
- ✅ Análise técnica
- ✅ Outlook e perspectivas

## 🔍 Troubleshooting

### Problemas Comuns

#### 1. Erro "API not enabled"
```
Erro: The API is not enabled for this project
```
**Solução**: Verificar se todas as 4 APIs foram habilitadas no Google Cloud

#### 2. Erro OAuth "invalid_client"
```
Erro: invalid_client
```
**Solução**: 
- Verificar Client ID e Secret
- Confirmar URLs de redirecionamento
- Aguardar propagação (até 5 minutos)

#### 3. Webhook n8n não responde
```
Erro: Connection timeout
```
**Solução**:
- Verificar se workflow está ativo
- Confirmar URL do webhook
- Testar webhook isoladamente

#### 4. Erro "Quota exceeded" Gemini
```
Erro: Resource has been exhausted
```
**Solução**:
- Aguardar reset do quota (1 minuto)
- Verificar limites da API key

#### 5. Documento não criado
**Verificar**:
- Credenciais Google Drive configuradas
- Permissões da pasta
- ID da pasta correto no workflow

### Logs de Debug

#### n8n Workflow
1. No n8n, clique no workflow
2. Vá para "Executions"
3. Analise logs de erro

#### Gemini API
```bash
# Testar quota
curl "https://generativelanguage.googleapis.com/v1beta/models?key=[SUA_CHAVE]"
```

#### BRAPI
```bash
# Verificar status
curl "https://brapi.dev/api/available?token=[SEU_TOKEN]"
```

### Contatos de Suporte

- **n8n Community**: [community.n8n.io](https://community.n8n.io)
- **Google Cloud Support**: [cloud.google.com/support](https://cloud.google.com/support)
- **BRAPI Discord**: [discord.gg/T5kEGME](https://discord.gg/T5kEGME)

---

## 📝 Próximos Passos

Após implementação bem-sucedida:

1. 📊 Configure monitoramento de execuções
2. 🔄 Teste com diferentes empresas
3. 📈 Monitore uso das APIs
4. 🚀 Considere upgrade para contas pagas se necessário
5. 🔧 Customize análises conforme necessidade

---

<div align="center">

**Implementação concluída com sucesso!** ✅  
Prossiga para o [Setup do Workflow n8n](n8n-workflow-setup.md)

</div>