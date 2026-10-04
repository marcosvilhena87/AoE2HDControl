# Estruturas provisórias

Última atualização: 2026-10-03.

## World

A classe `World` foi identificada por RTTI e deriva de `BaseWorld`.

Informações confirmadas:

```text
Classe: World
Base: BaseWorld
vtable: 0x009D329C
slots observados: 37
tamanho alocado observado: 0x2DC bytes
```

Modelo atual:

```cpp
struct World : BaseWorld
{
    // vtable = 0x009D329C

    // ... campos ainda não mapeados ...

    // +0x174
    PlayerIndex mLocalPlayerIndex;     // 🟢

    // ...

    // +0x184
    PlayerEntry* mPlayersBegin;        // ✅ estrutura / 🟢 semântica

    // +0x188
    PlayerEntry* mPlayersEnd;          // ✅ estrutura / 🟢 semântica

    // +0x18C
    PlayerEntry* mPlayersCapacityEnd;  // 🟡 hipótese

    // ... até pelo menos 0x2DC bytes ...
};
```

### Offsets

| Offset | Interpretação | Confiança |
|---|---|---|
| `+0x174` | `mLocalPlayerIndex` | 🟢 |
| `+0x184` | início da coleção associada a jogadores | ✅ / 🟢 nome |
| `+0x188` | fim da coleção associada a jogadores | ✅ / 🟢 nome |
| `+0x18C` | capacity/end-of-storage | 🟡 |

## Vtable de World

```text
0x009D329C
```

Foram observados 37 slots.

Entrada conhecida:

```text
slot 33
offset +0x84
→ FUN_00697800
```

## PlayerEntry

Stride observado:

```text
8 bytes
```

A composição permanece desconhecida.

Hipóteses de trabalho possíveis:

```cpp
struct PlayerEntry {
    Player* ptr;
    uint32_t unknown;
};
```

ou:

```cpp
struct PlayerEntry {
    uint32_t id;
    Player* ptr;
};
```

Nenhuma delas deve ser tratada como confirmada.

## Owner de World

Existe uma estrutura ainda não identificada que mantém o ponteiro ativo de `World` em:

```text
owner + 0xC8
```

Modelo:

```cpp
struct UnknownOwner
{
    // ...
    World* world; // +0xC8
};
```

A identidade de `UnknownOwner` é uma das principais pendências.

## Relação estrutural

```text
UnknownOwner
    │
    └── +0xC8 ──► World
                    │
                    ├── +0x174  LocalPlayerIndex
                    └── +0x184/+0x188  PlayerEntry[]
                                            │
                                            └── ? Player*
```

## Regra de modelagem

Nunca transformar um padrão de layout em nome semântico definitivo sem evidência independente.

Exemplo:

```text
begin/end + stride 8
```

confirma a existência de uma coleção contígua, mas não sozinho que seus elementos sejam `Player*`.
