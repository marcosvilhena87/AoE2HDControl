# Roadmap

Última atualização: 2026-10-03.

## Marco 1 — identificar World/GameState

Status: ✅ identificação estrutural principal concluída.

Já obtido:

```text
RTTI: World : BaseWorld
vtable: 0x009D329C
tamanho observado: 0x2DC bytes
construtor/inicializador: FUN_006889B0 (🟢)
owner + 0xC8 → World*
```

Ainda falta para fechar o marco operacional:

- identificar a classe do `owner`;
- descobrir de onde vem a instância de `owner`;
- obter uma cadeia runtime estável até `World*`.

Meta:

```cpp
World* GetWorld();
```

### Próxima tarefa concreta — maior retorno

Na função que contém a criação de `World` e a região `0x0062E65A`:

> localizar a primeira atribuição/escrita que define `local_20` / `owner`.

Queremos reconstruir:

```text
? global/root
      ↓
   owner
      ↓ +0xC8
   World*
```

Se `owner` vier de um global/singleton estável, o problema central de localização da instância pode ficar praticamente resolvido.

---

## Marco 2 — localizar jogador local

Status: em andamento.

Já obtido:

```text
World + 0x174 ≈ mLocalPlayerIndex
World + 0x184 = início de coleção
World + 0x188 = fim de coleção
stride = 8 bytes
```

Meta:

```cpp
auto localIndex = world->mLocalPlayerIndex;
auto localPlayer = world->GetPlayer(localIndex);
```

Pendências:

1. confirmar comportamento runtime de `+0x174`;
2. decompor `PlayerEntry`;
3. identificar qual campo de cada entry é `Player*`;
4. validar indexação pelo jogador local;
5. inspecionar `+0x18C`.

---

## Marco 3 — recursos do jogador

Prioridade:

```text
Food
Wood
Gold
Stone
Population
```

Uso principal: validação runtime.

Estratégia:

```text
Player*
  ↓
campos candidatos
  ↓
comparar com valores visíveis na UI
  ↓
confirmar offsets/tipos
```

---

## Marco 4 — objetos/unidades

Mapear:

- object id;
- owner;
- type;
- position;
- HP;
- coleção de objetos do jogador/world.

Meta conceitual:

```cpp
GetObjectById()
GetObjectsByType()
GetObjectsByTypes()
GetOwningPlayer()
GetTownCenters()
```

---

## Marco 5 — comandos

Investigar:

```text
RGE_Command
TRIBE_Command
```

Objetivos futuros:

- move;
- attack;
- build;
- gather;
- train;
- research;
- target object.

---

## Ordem de investigação recomendada

```text
1. origem de owner/local_20
2. cadeia estável owner+0xC8 → World*
3. validar World+0x174
4. decompor PlayerEntry de 8 bytes
5. resolver Local Player*
6. mapear recursos
7. mapear objetos/unidades
8. comandos
```

Evitar dispersar esforço em funções ainda sem ligação estrutural clara — por exemplo `FUN_00659750` — enquanto a cadeia até `World*` estiver a um passo de ser fechada.
