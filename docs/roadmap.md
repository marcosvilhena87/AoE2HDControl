# Roadmap

## Marco 1 — identificar World/GameState

Objetivo:

```cpp
World* world = GetWorld();
```

Critérios:

- identificar a classe real;
- localizar construtor/vtable ou evidência equivalente;
- confirmar os offsets principais;
- obter uma forma estável de localizar a instância em runtime.

## Marco 2 — localizar jogador local

Objetivo:

```cpp
auto localIndex = world->mLocalPlayerIndex;
auto localPlayer = world->GetPlayer(localIndex);
```

Critérios:

- entender `PlayerEntry`;
- validar indexação;
- resolver ponteiro real do jogador.

## Marco 3 — recursos do jogador

Prioridade:

```text
Food
Wood
Gold
Stone
Population
```

Esses campos serão usados como validação runtime.

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

## Próxima tarefa concreta

Analisar a região:

```text
0x009D3280–0x009D3330
```

e provar ou refutar se a sequência que contém `0x009D3320 → FUN_00697800` é uma vtable.
