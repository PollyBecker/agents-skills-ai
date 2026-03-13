# CLAUDE.md

Repositorio com agentes IT Valley em pastas individuais.

## Estrutura
- `agents/<agente>/CLAUDE.md`: prompt completo por agente

## Ordem da esteira (core)
- `01-prd-analyst`
- `02-analista-de-tela`
- `03-arquiteto-it-valley-backend`
- `04-arquiteto-it-valley-frontend`
- `05-arquiteto-designer`
- `06-dev-mockado`
- `07-arquiteto-sql-plus-mongodb`
- `08-p-o-product-owner`
- `08-b-gerente-de-projetos`
- `09-dev-frontend`
- `10-dev-backend`
- `11-qa-unitario`
- `12-qa-integracao`
- `13-qa-tela`
- `14-playwright-e2e`
- `15-guardiao-de-arquitetura`

## Opcionais
- `opc-a-ui-ux-opcional`
- `opc-b-mensageria-opcional`
- `opc-c-engenheiro-de-dados-plus-bi-opcional`

## Filosofia IT Valley

> Sistemas IT Valley sao construidos **orientados a Casos de Uso / Dominios**.

- Um **dominio** e uma area de negocio (Cliente, Contato, Pedido).
- Um **caso de uso** e uma operacao completa dentro do dominio (CriarCliente, ListarClientes).
- A **unidade de trabalho (dev feature)** e 1 caso de uso funcionando de ponta a ponta em todas as camadas.
- O DTO (request/response) define o caso de uso — e o que entra e sai do sistema.
- Arquitetura: **Clean Architecture com padroes taticos do DDD** (Entity rica, Repository com interface, isolamento de dominio).

## Estrutura de projeto IT Valley
- Todo projeto tem duas pastas na raiz: `backend/` e `frontend/`
- Backend: tudo na raiz de `backend/` (main.py, routers/, services/, models/, etc.) — SEM pasta `app/` intermediaria
- Frontend: tudo na raiz de `frontend/` (src/, package.json, etc.)
- Isso permite rodar `uvicorn main:app --reload` direto de dentro de `backend/`

## Estrutura backend
```
backend/
  dtos/                          # Request/Response por caso de uso (era schemas/)
    [dominio]/
      [caso_de_uso]/
        request.py               # 1 classe Pydantic (o que entra)
        response.py              # 1 classe Pydantic (o que sai)
      base.py                   # campos compartilhados
  domain/                        # Entities com regras de negocio (Python puro)
    [dominio]_entity.py
  factories/                     # DTO -> Entity (so construcao)
    [dominio]_factory.py
  mappers/                       # Entity <-> Model <-> Response (so conversao)
    [dominio]_mapper.py
  services/                      # Orquestracao (camada opaca)
    [dominio]_service.py
  routers/                       # API (camada opaca)
    [dominio].py
  models/                        # SQLAlchemy
    [dominio].py
  data/
    interfaces/base_repository.py
    repositories/sql/
    repositories/mongo/
    connections/
```

## Camadas opacas (NUNCA mudam quando campo muda)
- **Service** — so chama metodos publicos, nao conhece campos
- **Router/API** — recebe DTO, delega para Service, retorna response
- **Repository (base)** — CRUD generico

## Regras fundamentais IT Valley
1. Service e Router sao camadas opacas — nunca conhecem campos do DTO ou Entity
2. Entity (domain/) contem TODAS as regras de negocio — Python puro, sem framework
3. Factory cria Entity a partir de DTO — so construcao
4. Mapper converte entre camadas — so conversao
5. UI e a fabrica de DTOs - ela cria, ela conhece os campos
6. Mock antes de backend - cliente valida o fluxo antes do codigo real
7. VITE_USE_MOCK - flag obrigatoria em todo projeto
8. tenant_id em TODA tabela e TODA query - sem excecao
9. BI nunca em banco relacional - sempre Gold layer do datalake
10. Arquiteto SQL so age apos DTOs completos - banco segue os DTOs
11. Playwright so apos QA 11+12+13 - nao pular etapas
12. Dev Front e Dev Back sao times separados - coordenados pelo P.O.
13. Duvidas em aberto sao valiosas - nunca resolver com suposicoes
14. Imports explicitos, sem barrel exports (__init__.py)

## Estrutura frontend — Clean Code IT Valley

### Filosofia: Simples > Complexo. Dominio > Tecnico.

### Estrutura de pastas
```
src/lib/
├── components/
│   ├── ui/           # Genericos reutilizaveis
│   └── [dominio]/    # Componentes por dominio de negocio
├── dtos/             # Classes com readonly, constructor, isValid, toPayload
├── services/         # Metodos static, chama Repository (camada opaca)
├── repositories/     # Mock vs real via VITE_USE_MOCK
├── mocks/            # Dados falsos realistas
└── utils/            # Helpers puros (so se usados em 2+ lugares)
```

### Regras
- Componentes organizados por dominio, NAO por tipo tecnico
- NAO usar: atoms/molecules/organisms, barrel exports, pasta por componente
- DTOs imutaveis com readonly + constructor(Record) + isValid() + toPayload()
- Services com metodos static — NUNCA acessam campos do DTO (camada opaca)
- Repositories alternam mock/real via VITE_USE_MOCK
- Design tokens centralizados no app.css (cores via @theme, espacamentos via classes CSS)
- Import direto do arquivo, sem barrel exports

### Fluxo de dados
```
+page.svelte → Service.metodo() → Repository.metodo() → new DTO(data)
                                                        ↓
             <Componente dto={dto}> ← $derived(...)
```
