# Estruturas provisórias

Última atualização: 2026-10-03.

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

Runtime observado:

~~~text
begin = 0x105010E0
end   = 0x105010F8
cap   = 0x105010F8
size = 3
capacity = 3
~~~

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

Amostra:

~~~text
entry[0] = { 0x18AA701C, 0x18AA7010 }
entry[1] = { 0x146EF234, 0x146EF228 }
entry[2] = { 0x18B1F09C, 0x18B1F090 }
~~~

Nas três:

~~~text
object = controlBlock + 0x0C
~~~

RTTI contém _Ref_count_obj para WorldPlayerGaia, WorldPlayerHumanOrCoop e WorldPlayerComputer, reforçando fortemente:

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

### Classes concretas observadas

| Entry | Objeto runtime | Vtable runtime | Vtable estática | Classe |
|---|---:|---:|---:|---|
| 0 | 0x18AA701C | 0x0136C8A0 | 0x009DC8A0 | WorldPlayerGaia |
| 1 | 0x146EF234 | 0x0136CAE4 | 0x009DCAE4 | WorldPlayerHumanOrCoop |
| 2 | 0x18B1F09C | 0x0136C618 | 0x009DC618 | WorldPlayerComputer |

Estado: ✅ para a partida observada.

## Prefixo comum dos players

~~~cpp
struct WorldPlayerLike
{
    void** vtable;       // +0x00
    World* world;        // +0x04
    uint32_t field_08;   // +0x08, semântica ainda aberta
};
~~~

player+0x04 apontou para o World* ativo.

Valores de +0x08 observados:

~~~text
Gaia        = 2
HumanOrCoop = 1
Computer    = 3
~~~

Ainda não nomear esse campo como id/index sem validação cruzada.

## Hipótese operacional de GetLocalPlayer

Na primeira partida totalmente carregada:

~~~text
World+0x174 = 1
mPlayers[1] = WorldPlayerHumanOrCoop
~~~

Hipótese forte:

~~~cpp
WorldPlayer* GetLocalPlayer(World* world)
{
    auto i = world->mLocalPlayerIndex;
    return world->mPlayers[i].get();
}
~~~

Falta repetir em uma partida totalmente carregada com o humano em outro slot.

## Lifecycle

Ao criar uma nova partida, Owner+0xC8 mudou para nova instância de World.

A nova instância já possuía a vtable correta antes de +0x174 refletir o slot esperado.

Portanto:

~~~text
valid World* != necessariamente gameplay-ready World
~~~
