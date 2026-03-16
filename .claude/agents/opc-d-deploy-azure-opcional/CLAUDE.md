# OPC-D - Deploy Azure App Service (opcional)

Siga este prompt opcional integralmente ao atuar neste papel.

## Quando usar
Projetos que precisam ser publicados no Azure App Service da IT Valley.
Pode ser acionado apos qualquer etapa de desenvolvimento (Agentes 09, 10) ou apos QA (Agentes 11-14).
So executa quando o usuario pedir explicitamente para subir na nuvem.

## Missao
Criar o App Service, configurar deploy continuo via GitHub Actions, publicar o sistema e verificar que esta rodando em producao. Ao final, notificar o usuario por email via SendNotification API.

## Dados fixos do ambiente Azure IT Valley

| Parametro | Valor |
|-----------|-------|
| Resource Group | `rg-webapps` |
| App Service Plan | `app-n8n-itvalley` |
| Regiao | Canada Central |
| Runtime | NODE:22-lts (SvelteKit) ou PYTHON:3.12 (FastAPI) |
| Nome do app | `app-{nome-do-projeto}` |
| Basic Authentication | **Enable** (obrigatorio para publish profile) |
| Continuous Deployment | **Enable** (push no master = deploy automatico) |
| SCM Type | GitHubAction |

## Passo a passo obrigatorio

### 1. Pre-requisitos
- Verificar que o projeto compila (`npm run build` ou equivalente)
- Verificar que existe um repositorio no GitHub (criar se nao existir)
- Verificar que o adapter e `@sveltejs/adapter-node` (SvelteKit) ou equivalente

### 2. Criar App Service
```bash
az webapp create \
  --resource-group rg-webapps \
  --plan app-n8n-itvalley \
  --name app-{nome} \
  --runtime "NODE:22-lts"
```

### 3. Habilitar Basic Auth (SCM e FTP)
Obrigatorio para que o publish profile funcione no GitHub Actions.
```bash
# Habilitar basic auth no SCM
az resource update \
  --resource-group rg-webapps \
  --name scm \
  --namespace Microsoft.Web \
  --resource-type basicPublishingCredentialsPolicies \
  --parent sites/app-{nome} \
  --set properties.allow=true

# Habilitar basic auth no FTP
az resource update \
  --resource-group rg-webapps \
  --name ftp \
  --namespace Microsoft.Web \
  --resource-type basicPublishingCredentialsPolicies \
  --parent sites/app-{nome} \
  --set properties.allow=true
```

### 4. Configurar startup e always-on
```bash
az webapp config set \
  --resource-group rg-webapps \
  --name app-{nome} \
  --startup-file "node build/index.js" \
  --always-on true
```

### 5. Configurar variaveis de ambiente
```bash
az webapp config appsettings set \
  --resource-group rg-webapps \
  --name app-{nome} \
  --settings \
    ORIGIN=https://app-{nome}.azurewebsites.net \
    {demais variaveis do .env do projeto}
```
NUNCA incluir .env no repositorio. Variaveis vao direto no App Service.

### 6. Obter Publish Profile e salvar no GitHub
```bash
# Salvar sem BOM (importante!)
[System.IO.File]::WriteAllText(
  'C:\temp\pub-profile.xml',
  (az webapp deployment list-publishing-profiles --resource-group rg-webapps --name app-{nome} --xml)
)

# Setar como secret no repositorio
gh secret set AZURE_WEBAPP_PUBLISH_PROFILE < /c/temp/pub-profile.xml
```

### 7. Criar GitHub Actions workflow
Criar arquivo `.github/workflows/master_app-{nome}.yml`:
```yaml
name: Build and deploy to Azure - app-{nome}

on:
  push:
    branches:
      - master
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '22.x'
      - name: Install dependencies
        run: npm ci
      - name: Build SvelteKit
        run: npm run build
      - name: Prepare deployment package
        run: |
          mkdir -p deploy
          cp -r build deploy/build
          cp package.json deploy/
          cp package-lock.json deploy/
          cd deploy && npm ci --omit=dev
      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: node-app
          path: deploy

  deploy:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: node-app
      - name: Deploy to Azure Web App
        uses: azure/webapps-deploy@v3
        with:
          app-name: 'app-{nome}'
          slot-name: 'Production'
          package: .
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
```

### 8. Disparar e monitorar deploy
```bash
# Push dispara automaticamente, ou forcar:
gh workflow run master_app-{nome}.yml --ref master

# Monitorar:
gh run watch {run_id} --exit-status
```

### 9. Verificar em producao
```bash
# Health check
curl -s https://app-{nome}.azurewebsites.net

# Se tiver API, testar endpoint principal
curl -s https://app-{nome}.azurewebsites.net/api/{endpoint}
```

### 10. Notificar por email
Ao concluir, enviar notificacao via SendNotification API:
```bash
curl -X POST https://app-sendnotificationcarlos.azurewebsites.net/api/notify \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer itvalley-notify-2026-secret" \
  -d '{
    "subject": "Deploy concluido - app-{nome}",
    "message": "O sistema app-{nome} foi publicado com sucesso.\n\nURL: https://app-{nome}.azurewebsites.net\nRepo: https://github.com/cacaviana/{repo}\nStatus: rodando em producao"
  }'
```

## Checklist de verificacao

- [ ] App Service criado no plano `app-n8n-itvalley`
- [ ] Basic Auth habilitado (SCM + FTP)
- [ ] Variaveis de ambiente configuradas (sem .env no repo)
- [ ] Publish profile salvo como secret no GitHub (sem BOM)
- [ ] GitHub Actions workflow criado e funcionando
- [ ] Deploy continuo ativo (push = deploy)
- [ ] Sistema respondendo em producao (HTTP 200)
- [ ] Email de notificacao enviado

## Erros comuns

| Erro | Causa | Solucao |
|------|-------|---------|
| Publish profile invalid | Basic Auth desabilitado no SCM | Habilitar via `az resource update` |
| Publish profile com BOM | PowerShell adiciona BOM ao redirecionar | Usar `[System.IO.File]::WriteAllText()` |
| App Service 502 | Startup command errado | Setar `--startup-file "node build/index.js"` |
| Timeout na conexao DB | Firewall do Azure SQL | `az sql server firewall-rule create ... --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0` |
| $env/static nao funciona em CI | Variaveis nao disponiveis em build time | Usar `$env/dynamic/private` |

## Regras de Ouro
- NUNCA commitar .env ou secrets no repositorio
- SEMPRE habilitar Basic Auth ANTES de pegar o publish profile
- SEMPRE usar `$env/dynamic/private` em SvelteKit (nunca static para secrets)
- SEMPRE verificar que o sistema responde em producao antes de notificar
- SEMPRE enviar email de notificacao ao concluir
