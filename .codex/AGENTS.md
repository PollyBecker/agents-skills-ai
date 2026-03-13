# AGENTS.md

## Objetivo
Orquestrar a esteira IT Valley no Codex, garantindo ordem correta, handoff claro entre agentes e paralelismo apenas quando permitido.

## Filosofia IT Valley

> Sistemas IT Valley sao construidos **orientados a Casos de Uso / Dominios**.

- Um **dominio** e uma area de negocio (Cliente, Contato, Pedido).
- Um **caso de uso** e uma operacao completa dentro do dominio (CriarCliente, ListarClientes).
- A **unidade de trabalho (dev feature)** e 1 caso de uso funcionando de ponta a ponta em todas as camadas.
- O DTO (request/response) define o caso de uso — e o que entra e sai do sistema.

## Regra 0 (obrigatoria)
Sempre iniciar perguntando onde esta o PRD.

Pergunta inicial padrao:
`Onde esta o PRD do projeto? Me envie o caminho/arquivo para eu identificar em qual etapa da esteira devemos comecar.`

## Roteamento por etapa

### Se o PRD nao existe
1. Executar `AGENTE 01 - PRD Analyst`.
2. Entregar PRD completo.
3. Confirmar aprovacao do PRD com o usuario.
4. Avancar para `AGENTE 02 - Analista de Tela`.

### Se o PRD ja existe
1. Pular `AGENTE 01`.
2. Executar `AGENTE 02 - Analista de Tela`.

### Depois do Analista de Tela (AGENTE 02)
Executar em paralelo:
1. `AGENTE 03 - Arquiteto IT Valley Backend`
2. `AGENTE 04 - Arquiteto IT Valley Frontend`
3. `AGENTE 05 - Arquiteto Designer`

Regra: os tres agentes (03, 04, 05) usam como entrada o output do AGENTE 02.

### Depois de 03 + 04 + 05
1. Executar `AGENTE 06 - Dev Mockado`.
2. Entregar mockado navegavel com dados falsos realistas.
3. Parar e solicitar validacao do cliente antes de seguir.

## Gate de validacao (obrigatorio)
Nao avancar para etapas seguintes sem aprovacao explicita do usuario no fim do AGENTE 06.

Pergunta padrao de gate:
`O mockado foi aprovado pelo cliente? Se sim, sigo para a proxima fase.`

### Depois do mockado aprovado (AGENTE 06)
1. Executar `AGENTE 07 - Arquiteto SQL + MongoDB`.
2. Executar `AGENTE 08 - P.O. (Product Owner)` — divide em dominios e casos de uso.
3. Executar `AGENTE 08-b - Gerente de Projetos` — gera documento de gestao com todos os dominios, casos de uso e status mapeados.

### Depois do P.O. + Gerente de Projetos (08 + 08-b)
Executar em paralelo:
1. `AGENTE 09 - Dev Frontend` (trabalha com VITE_USE_MOCK=true)
2. `AGENTE 10 - Dev Backend`

Regra: AGENTE 15 (Guardiao) roda antes e durante o AGENTE 10.

## Regras especificas de arquitetura (obrigatorias)

### Backend
- O `AGENTE 03 - Arquiteto IT Valley Backend` e a fonte da verdade da arquitetura backend.
- Todo desenvolvimento backend (AGENTE 10) deve consultar continuamente o output do AGENTE 03.
- DTOs ficam em `dtos/[dominio]/[caso_de_uso]/request.py` e `response.py` (NAO em `schemas/`).
- Regras de negocio ficam na Entity (`domain/[dominio]_entity.py`) — Python puro, sem framework.
- Factory cria Entity a partir de DTO (so construcao). Mapper converte entre camadas (so conversao).
- Service e Router sao camadas opacas — NUNCA acessam campos de DTO ou Entity.
- Se houver conflito entre implementacao e arquitetura, corrigir a implementacao; nao ignorar o AGENTE 03.
- Se uma mudanca arquitetural for necessaria, atualizar primeiro o AGENTE 03 e so depois codificar.

### Frontend
- O `AGENTE 04 - Arquiteto IT Valley Frontend` e a fonte da verdade da arquitetura frontend.
- Todo desenvolvimento frontend (AGENTE 09) deve consultar continuamente o output do AGENTE 04.
- Componentes organizados por dominio (`components/cliente/`, `components/chat/`), NAO por tipo tecnico.
- DTOs imutaveis com `readonly`, `constructor(Record)`, `isValid()`, `toPayload()`.
- Services com metodos `static` — NUNCA acessam campos do DTO, so metodos publicos (camada opaca).
- Repositories alternam mock/real via `VITE_USE_MOCK`.
- Design tokens centralizados no `app.css` (cores via `@theme`, espacamentos via classes CSS).
- Import direto do arquivo, sem barrel exports.
- O AGENTE 15 valida conformidade tanto backend quanto frontend.

## Agente de conformidade arquitetural
- Usar `AGENTE 15 - Guardiao de Arquitetura` como auditor recorrente.
- Rodar o AGENTE 15 nos checkpoints:
1. Antes de iniciar AGENTE 10 (valida pacote x arquitetura).
2. Durante AGENTE 10 (a cada modulo/stories concluidos).
3. Antes de enviar para QA Unitario/Integracao.
- Nenhum pacote backend segue para QA sem aprovacao do AGENTE 15.

## Mapa rapido de entradas e saidas

- AGENTE 01
Entrada: problema bruto
Saida: PRD

- AGENTE 02
Entrada: PRD
Saida: documento de telas

- AGENTE 03 (paralelo)
Entrada: documento de telas
Saida: arquitetura backend (DTOs, domain, services, repositories)

- AGENTE 04 (paralelo)
Entrada: documento de telas + contratos backend
Saida: arquitetura frontend

- AGENTE 05 (paralelo)
Entrada: documento de telas
Saida: guia visual por tela

- AGENTE 06
Entrada: arquitetura frontend + guia visual
Saida: mockado clicavel + pasta mocks + ambiente mock

- AGENTE 07
Entrada: DTOs backend + mock validado
Saida: scripts SQL + MongoDB

- AGENTE 08
Entrada: tudo acima
Saida: pacotes de dev features por dominio/caso de uso

- AGENTE 08-b
Entrada: pacotes do P.O. (08)
Saida: documento de gestao de projeto com dominios, casos de uso e status

- AGENTE 09 (paralelo)
Entrada: pacote frontend do P.O.
Saida: codigo frontend completo

- AGENTE 10 (paralelo)
Entrada: pacote backend do P.O.
Saida: codigo backend completo

- AGENTE 15 (recorrente)
Entrada: codigo em desenvolvimento
Saida: relatorio de conformidade arquitetural

## Politica de execucao
- Nao inventar ordem fora desta esteira.
- Sempre informar em qual agente/etapa esta executando.
- Sempre listar artefatos de entrada faltantes antes de comecar um agente.
- Em caso de duvida de etapa, voltar para a pergunta do PRD e identificar ultimo artefato aprovado.
- Em qualquer duvida de arquitetura backend, retornar ao AGENTE 03 e validar com AGENTE 15.

## Estrutura esperada no repositorio
- `.codex/agents/<agente>/SKILL.md`
- `.claude/agents/<agente>/CLAUDE.md`
- `README.md`
- `AGENTS.md` (este orquestrador)
