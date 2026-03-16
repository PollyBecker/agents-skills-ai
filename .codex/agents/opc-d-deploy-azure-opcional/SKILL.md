---
name: opc-d-deploy-azure-opcional
description: Executar o papel opcional 'Deploy Azure App Service' na esteira IT Valley. Cria App Service, configura deploy continuo via GitHub Actions, publica e verifica em producao.
---

# OPC-D - Deploy Azure App Service (opcional)

Use este guia como instrucao operacional opcional.

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
az resource update --resource-group rg-webapps --name scm --namespace Microsoft.Web \
  --resource-type basicPublishingCredentialsPolicies --parent sites/app-{nome} \
  --set properties.allow=true

az resource update --resource-group rg-webapps --name ftp --namespace Microsoft.Web \
  --resource-type basicPublishingCredentialsPolicies --parent sites/app-{nome} \
  --set properties.allow=true
```

### 4. Configurar startup e always-on
```bash
az webapp config set --resource-group rg-webapps --name app-{nome} \
  --startup-file "node build/index.js" --always-on true
```

### 5. Configurar variaveis de ambiente
```bash
az webapp config appsettings set --resource-group rg-webapps --name app-{nome} \
  --settings ORIGIN=https://app-{nome}.azurewebsites.net {demais variaveis}
```

### 6. Obter Publish Profile (sem BOM)
```powershell
[System.IO.File]::WriteAllText('C:\temp\pub-profile.xml',
  (az webapp deployment list-publishing-profiles --resource-group rg-webapps --name app-{nome} --xml))
```
```bash
gh secret set AZURE_WEBAPP_PUBLISH_PROFILE < /c/temp/pub-profile.xml
```

### 7. Criar GitHub Actions workflow
Arquivo `.github/workflows/master_app-{nome}.yml` com build + deploy usando `azure/webapps-deploy@v3`.

### 8. Disparar, monitorar e verificar
```bash
gh workflow run master_app-{nome}.yml --ref master
gh run watch {run_id} --exit-status
curl -s https://app-{nome}.azurewebsites.net
```

### 9. Notificar por email
```bash
curl -X POST https://app-sendnotificationcarlos.azurewebsites.net/api/notify \
  -H "Authorization: Bearer itvalley-notify-2026-secret" \
  -H "Content-Type: application/json" \
  -d '{"subject":"Deploy concluido","message":"app-{nome} rodando em producao"}'
```

## Checklist
- [ ] App Service criado no plano app-n8n-itvalley
- [ ] Basic Auth habilitado (SCM + FTP)
- [ ] Variaveis configuradas (sem .env no repo)
- [ ] Publish profile sem BOM como secret no GitHub
- [ ] GitHub Actions workflow funcionando
- [ ] Deploy continuo ativo
- [ ] Sistema respondendo HTTP 200
- [ ] Email enviado

## Erros comuns
| Erro | Solucao |
|------|---------|
| Publish profile invalid | Habilitar Basic Auth primeiro |
| BOM no publish profile | Usar `[System.IO.File]::WriteAllText()` |
| 502 Bad Gateway | Setar startup-file correto |
| DB timeout | Liberar firewall Azure SQL (0.0.0.0) |
| $env/static falha em CI | Usar $env/dynamic/private |

## Regras de Ouro
- NUNCA commitar .env ou secrets no repositorio
- SEMPRE habilitar Basic Auth ANTES de pegar o publish profile
- SEMPRE usar $env/dynamic/private em SvelteKit
- SEMPRE verificar producao antes de notificar
- SEMPRE enviar email ao concluir
