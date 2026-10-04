# Estruturas provisórias

Última atualização: 2026-10-03.

## World

A classe `World` foi identificada por RTTI e deriva de `BaseWorld`.

```text
Classe: World
Base: BaseWorld
vtable estática: 0x009D329C
vtable RVA:     0x005D329C
slots observados: 37
tamanho alocado observado: 0x2DC bytes
```

Modelo atual:

```cpp
struct World : BaseWorld
{
    // +0x000
    void** vtable;                    // ✅ 0x009D329C estático

    // ...

    // +0x174
    /* PlayerIndex-like */ uint16_t mLocalPlayerIndexRaw; // ✅ 16 bits baixos consumidos

    // ...

    // +0x184
    PlayerEntry* mPlayersBegin;       // ✅

    // +0x188
    PlayerEntry* mPlayersEnd;         // ✅

    // +0x18C
    PlayerEntry* mPlayersCapacityEnd; // 🟡 provável std::vector end-of-storage

    // ... até pelo menos 0x2DC bytes ...
};
```

### Offsets

| Offset | Interpretação | Confiança |
|---|---|---|
| `+0x000` | vtable `World` | ✅ |
| `+0x174` | `mLocalPlayerIndex` / raw nos 16 bits baixos | ✅ |
| `+0x184` | `mPlayers.begin` | ✅ |
| `+0x188` | `mPlayers.end` | ✅ |
| `+0x18C` | capacity/end-of-storage | 🟡 |

## Cadeia até World

Global estático:

```text
DAT_00AF2C58
```

RVA:

```text
0x006F2C58
```

Modelo:

```cpp
struct UnknownOwner
{
    // ...
    World* world; // +0xC8 ✅
};
```

Acesso:

```text
[moduleBase + 0x006F2C58]
        ↓
      Owner*
        ↓ +0xC8
      World*
```

A classe do owner ainda não foi identificada.

## PlayerEntry

Stride confirmado:

```text
8 bytes
```

Runtime observado:

```text
entry[0] = { 0x18AA701C, 0x18AA7010 }
entry[1] = { 0x146EF234, 0x146EF228 }
entry[2] = { 0x18B1F09C, 0x18B1F090 }
```

Em todas:

```text
object = controlBlock + 0x0C
```

Modelo de trabalho:

```cpp
struct PlayerEntry
{
    void* object;        // +0x0 ✅ observado
    void* controlBlock;  // +0x4 ✅ observado
}; // sizeof = 8
```

O control block do primeiro elemento apresentou:

```text
+0x00 → vtable/control RTTI-ish pointer
+0x04 → 1
+0x08 → 1
+0x0C → início do objeto inline
```

Isso é fortemente compatível com `std::shared_ptr<T>` MSVC x86/`make_shared`.

Confiança:

- layout de dois ponteiros: ✅
- objeto inline a `control+0x0C` nas amostras: ✅
- interpretação como `std::shared_ptr<T>`: 🟢
- tipo template exato: 🟡

## Player object / RTTI

Os três `object` observados começaram com:

```text
runtime vtable = 0x0136C8A0
```

Na execução em que:

```text
moduleBase = 0x00D90000
```

isso corresponde a:

```text
static vtable = 0x009DC8A0
```

RTTI:

```text
vtable[-1]          = 0x00A23F58
TypeDescriptor      = 0x00ACBAF4
decorated type name = ".?AVWorldPlayerGaia@@"
```

Logo:

```text
0x009DC8A0 → WorldPlayerGaia RTTI ✅
```

### Cuidado semântico

Apesar de a partida ter Gaia + humano + IA, **as três entradas observadas apresentaram a mesma vtable primária `WorldPlayerGaia`**.

Portanto ainda não sabemos onde a distinção entre:

```text
Gaia
HumanOrCoop
Computer
```

fica armazenada.

Não assumir:

```text
entry[1] = WorldPlayerHumanOrCoop
entry[2] = WorldPlayerComputer
```

até localizar evidência direta.

RTTI adicional encontrado no binário:

```text
0x00ACBB14 → WorldPlayerHumanOrCoop
0x00ACBB3C → WorldPlayerScenarioEditorPhantom
```

Essas classes ainda não foram ligadas diretamente aos objetos runtime acima.

## Valores próximos ao início dos player objects

Nos objetos observados:

```text
[player + 0x04] = World*
```

O valor bateu com o `World*` validado `0x18431CD8` na execução analisada.

Em `player+0x08`, foram observados valores diferentes entre os três objetos (`2`, `1`, `3` na amostra). Esse campo é candidato a id/índice/discriminador, mas sua semântica ainda é 🟡.

Modelo parcial:

```cpp
struct WorldPlayerLike
{
    void** vtable; // +0x00
    World* world;  // +0x04 ✅ runtime
    uint32_t field_08; // +0x08 🟡 possível id/índice
    // ...
};
```

## Relação estrutural

```text
DAT_00AF2C58
      ↓
    Owner*
      ↓ +0xC8
    World
      ├── +0x174  local player index
      └── +0x184/+0x188  PlayerEntry[]
                              │
                              ├── object
                              └── control block
```

## Regra de modelagem

Nunca transformar um padrão de layout em nome semântico definitivo sem evidência independente.

Exemplo:

```text
begin/end + stride 8
```

confirma a coleção contígua; RTTI e validação runtime são usados para dar nome aos objetos.
