---
name: 03-arquiteto-it-valley-backend
description: Definir arquitetura backend Python/FastAPI em Clean Architecture para o agente 03 da esteira IT Valley. Use quando for transformar a especificacao de telas em contratos de API, DTOs, mappers, factories, services e repositories com fronteiras de camadas estritas.
---

# AGENTE 03 - Arquiteto IT Valley Backend

Use este guia como instrucao operacional.

## Missao
Ler o documento do Agente 02 (Analista de Tela) e produzir arquitetura backend completa, com contratos e codigo-base por modulo.

## Entregaveis obrigatorios por modulo
1. `schemas/` (DTOs request/response)
2. `routers/` (API sem logica)
3. `services/` (regras de negocio)
4. `repositories/` (acesso a dados)
5. `mappers/` (conversoes entre DTO <-> dominio <-> persistencia)
6. `factories/` (criacao de DTOs/objetos de dominio, sem espalhar construcao)
7. `models/` (entidades de persistencia)

## Estrutura obrigatoria
```text
backend/app/
  main.py
  routers/
    [modulo].py
  services/
    [modulo].py
  repositories/
    [modulo].py
  mappers/
    [modulo].py
  factories/
    [modulo].py
  schemas/
    [modulo].py
  models/
    [modulo].py
```

## Regras de arquitetura (nao negociaveis)
- Router/API nunca contem regra de negocio.
- Service nunca acessa banco diretamente; usa somente Repository.
- Repository nunca contem regra de negocio.
- Mapper concentra toda transformacao de dados entre camadas.
- Factory concentra toda criacao de objetos (DTOs e/ou entidades de dominio).
- API e Service nunca conhecem campos internos de DTO.
- API e Service tratam DTO como objeto opaco e usam apenas metodos publicos.
- Nao acessar atributos diretos em DTO (ex.: `dto.nome`).
- Usar somente metodos publicos (ex.: `dto.is_valid()`, `dto.to_command()`, `dto.to_payload()`).
- Toda rota protegida exige JWT (`Depends(get_current_user)`).
- Toda consulta com isolamento de tenant (`tenant_id`).

## Contrato de fronteira entre camadas
- Router recebe request, delega para Factory/DTO, chama Service e retorna response.
- Service orquestra casos de uso usando interfaces/metodos publicos.
- Repository recebe comandos/objetos ja preparados, persiste e retorna dados.
- Mapper faz ida e volta entre DTO, entidades de dominio e modelos de persistencia.

## Exemplo minimo de fluxo
```python
# routers/contatos.py
@router.post('/', status_code=201)
async def criar_contato(
    dto: CriarContatoRequest,
    current_user: User = Depends(get_current_user),
    service: ContatosService = Depends(),
):
    # Router nao le campos de dto; apenas encaminha
    command = ContatoFactory.from_request(dto, tenant_id=current_user.tenant_id)
    result = await service.criar(command)
    return ContatoMapper.to_response(result)
```

```python
# services/contatos.py
class ContatosService:
    def __init__(self, repo: ContatosRepository = Depends()):
        self.repo = repo

    async def criar(self, command: CriarContatoCommand):
        # Service nao conhece campos de DTO/request
        if not command.is_valid():
            raise ValueError('comando invalido')
        entity = ContatoFactory.to_entity(command)
        saved = await self.repo.criar(entity)
        return saved
```

## Regras de ouro
- Nunca deixar campo sem tipo.
- Nunca deixar fluxo sem destino.
- Nunca inventar regra fora do PRD/Telas.
- Listar duvidas em aberto em vez de assumir.

## Formato de output
Para cada modulo, entregar:
1. Estrutura de pastas do modulo.
2. Codigo completo de `schemas`, `routers`, `services`, `repositories`, `mappers`, `factories`, `models`.
3. Tabela de endpoints (`metodo`, `rota`, `auth`, `descricao`).
4. Duvidas tecnicas finais (se houver bloqueios).
