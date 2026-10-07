# Estruturas provisórias

Última atualização: 2026-10-06.

## World

~~~text
Classe: World
Base: BaseWorld
vtable estática: 0x009D329C
vtable RVA: 0x005D329C
tamanho observado: 0x2DC bytes
~~~

Modelo:

~~~cpp
struct World : BaseWorld
{
    void** vtable; // +0x000

    // ...

    uint16_t mLocalPlayerIndexRaw; // +0x174

    // ...

    PlayerEntry* mPlayersBegin;       // +0x184
    PlayerEntry* mPlayersEnd;         // +0x188
    PlayerEntry* mPlayersCapacityEnd; // +0x18C
};
~~~

### Partida de 8 jogadores

~~~text
World* = 0x18839030

begin = 0x1865C968
end   = 0x1865C9B0
cap   = 0x1865C9B0

size     = 9
capacity = 9
~~~

Os 9 elementos são Gaia + 8 jogadores reais.

## Cadeia até World

~~~text
[moduleBase + 0x006F2C58]
        ↓
      Owner*
        ↓ +0xC8
      World*
~~~

## PlayerEntry

Stride confirmado: 8 bytes.

~~~cpp
struct PlayerEntry
{
    WorldPlayer* object;  // +0x0
    void* controlBlock;   // +0x4
};
~~~

Na partida de 8 jogadores:

| índice | object | controlBlock | classe |
|---:|---:|---:|---|
| 0 | 0x0F588734 | 0x0F588728 | WorldPlayerGaia |
| 1 | 0x1469485C | 0x14694850 | WorldPlayerHumanOrCoop |
| 2 | 0x184A501C | 0x184A5010 | WorldPlayerComputer |
| 3 | 0x151FA41C | 0x151FA410 | WorldPlayerComputer |
| 4 | 0x13B4941C | 0x13B49410 | WorldPlayerComputer |
| 5 | 0x185E901C | 0x185E9010 | WorldPlayerComputer |
| 6 | 0x138E564C | 0x138E5640 | WorldPlayerComputer |
| 7 | 0x143F701C | 0x143F7010 | WorldPlayerComputer |
| 8 | 0x18C7584C | 0x18C75840 | WorldPlayerComputer |

Em todas:

~~~text
object = controlBlock + 0x0C
~~~

RTTI contém `_Ref_count_obj` para Gaia/Human/Computer, reforçando:

~~~text
std::vector<std::shared_ptr<WorldPlayer>> mPlayers
~~~

## Hierarquia WorldPlayer

RTTI confirma:

~~~text
WorldPlayer
WorldPlayerBase
WorldPlayerGaia
WorldPlayerHumanOrCoop
WorldPlayerComputer
WorldPlayerScenarioEditorPhantom
~~~

### Vtables

| Classe | estática | runtime com base 0x00410000 |
|---|---:|---:|
| WorldPlayerGaia | 0x009DC8A0 | 0x009EC8A0 |
| WorldPlayerHumanOrCoop | 0x009DCAE4 | 0x009ECAE4 |
| WorldPlayerComputer | 0x009DC618 | 0x009EC618 |

## Prefixo comum dos players

~~~cpp
struct WorldPlayerLike
{
    void** vtable;       // +0x00
    World* world;        // +0x04
    uint32_t field_08;   // +0x08
};
~~~

Confirmado:

~~~text
[player+0x04] = World*
~~~

Valores de `+0x08` observados:

~~~text
Gaia        = 2
HumanOrCoop = 1
Computer    = 3
~~~

Esse campo **não é a coluna “Jog.” 1–8 do lobby**: o valor do humano permaneceu `1` enquanto a coluna “Jog.” foi alterada.

Nome semântico ainda em aberto.

## SlotRecord

Stride confirmado: `0x28`.

Modelo parcial:

~~~cpp
struct SlotRecord
{
    // ...

    int32_t configIndex; // +0x18

    // +0x1C e +0x20 ainda sem nome semântico definitivo
};
~~~

Amostras:

~~~text
0x00B01AF8 → configIndex 0
0x00B01B20 → configIndex 1
0x00B01B48 → configIndex 2
0x00B01B70 → configIndex 3
...
~~~

Importante:

~~~text
SlotRecord.configIndex != worldPlayerIndex
~~~

O endereço imediatamente anterior à tabela contém outros dados/globais; não tratar `0x00B01AD0` como “SlotRecord da Gaia”.

## ResolvedPlayerConfig

Stride confirmado pela aritmética de `FUN_00598310`: `0x68`.

Modelo parcial:

~~~cpp
struct ResolvedPlayerConfig
{
    // ...

    int32_t playerNumberIndex;  // +0x4C, confirmado; zero-based para “Jog.” 1..8
    int32_t worldPlayerIndex;   // +0x50, confirmado
    int32_t humanity;           // +0x54, confirmado

    // ...
}; // stride 0x68
~~~

### Relação com SlotRecord

~~~cpp
ResolvedPlayerConfig* ResolveSlotConfig(SlotRecord* slot)
{
    int i = slot->configIndex;
    return base + 0xB0 + i * 0x68;
}
~~~

### Valores observados

| configIndex | worldPlayerIndex | humanity | interpretação |
|---:|---:|---:|---|
| 0 | 1 | 2 | humano |
| 1 | 2 | 4 | computador |
| 2 | -1 | 1 | sem WorldPlayer |
| 3 | -1 | 1 | sem WorldPlayer |
| 4 | -1 | 1 | sem WorldPlayer |
| 5 | -1 | 1 | sem WorldPlayer |
| 6 | -1 | 1 | sem WorldPlayer |
| 7 | -1 | 1 | sem WorldPlayer |

`playerNumberIndex` foi validado dinamicamente mantendo a mesma linha/configuração e alterando somente a coluna visual:

~~~text
Jog.1 → 0
Jog.2 → 1
Jog.3 → 2
~~~

Logo, a representação é zero-based e o domínio esperado para `Jog.1..8` é `0..7`.

`humanity=3` ainda não observado diretamente.

## UI do lobby e callback de número do jogador

`FUN_0064F0E0` itera explicitamente oito configurações:

~~~asm
006500F3  XOR  ESI,ESI
...
00650131  IMUL EAX,ESI,70h
00650139  ADD  EAX,EDI
...
00651506  INC  ESI
0065150D  CMP  ESI,08
00651510  JC   00650131
~~~

Portanto:

~~~text
configIndex = 0..7
row/config UI = lobbyBase + configIndex * 0x70
~~~

Estrutura confirmada do callback criado por `FUN_00644C10`:

~~~cpp
struct PlayerNumberCallback
{
    void** vtable;          // +0x00 = 0x009D1128
    void* target;           // +0x04 = objeto/contexto do lobby
    int32_t configIndex;    // +0x08 = 0..7
}; // 0x0C
~~~

O thunk virtual em `0x00653CD0` lê `+0x08`, usa `+0x04` como `this` e chama `FUN_00658E40(configIndex)`.

## Relação configuração → World

~~~text
SlotRecord
  +0x18 configIndex
      ↓
ResolvedPlayerConfig
  +0x50 worldPlayerIndex
  +0x54 humanity
      ↓
World.mPlayers[worldPlayerIndex]
      ↓
WorldPlayer*
~~~

Validações:

~~~text
SlotRecord 0x00B01AF8
  configIndex 0
  worldPlayerIndex 1
  humanity 2
  → WorldPlayerHumanOrCoop

SlotRecord 0x00B01B20
  configIndex 1
  worldPlayerIndex 2
  humanity 4
  → WorldPlayerComputer
~~~

## Gaia

~~~text
World.mPlayers[0] = WorldPlayerGaia
~~~

Gaia é uma entidade interna especial e não há evidência de um `SlotRecord` normal equivalente na tabela dos jogadores configuráveis.

## Hipótese operacional de GetLocalPlayer

Em partidas observadas:

~~~text
World+0x174 = mLocalPlayerIndex
mPlayers[index] = WorldPlayerHumanOrCoop
~~~

Modelo:

~~~cpp
WorldPlayer* GetLocalPlayer(World* world)
{
    auto i = world->mLocalPlayerIndexRaw;
    return world->mPlayers[i].get();
}
~~~

A relação é forte. A coluna visual **“Jog.”** é um conceito separado e agora está localizada em `ResolvedPlayerConfig+0x4C` como `playerNumberIndex`.

## Lifecycle

Ao trocar/criar partidas, `World` e objetos de player podem ser recriados.

Portanto:

~~~text
valid World* != necessariamente gameplay-ready World
ponteiros derivados antigos podem ficar obsoletos
~~~

O adapter deve resolver a cadeia atual e validar readiness antes de usar os objetos.
