# Estruturas provisórias

## UnknownWorldLike

Modelo de trabalho atual:

```cpp
struct UnknownWorldLike
{
    void* vtable; // hipótese

    // unknown

    // +0x174
    PlayerIndex mLocalPlayerIndex;

    // unknown

    // +0x184
    PlayerEntry* mPlayersBegin;

    // +0x188
    PlayerEntry* mPlayersEnd;

    // +0x18C
    // possível end-of-storage/capacity — ainda não confirmado
};
```

### Evidências

| Offset | Interpretação | Confiança |
|---|---|---|
| `+0x174` | `mLocalPlayerIndex` | 🟢 |
| `+0x184` | início da coleção de jogadores | ✅ / 🟢 quanto ao nome |
| `+0x188` | fim da coleção de jogadores | ✅ / 🟢 quanto ao nome |
| `+0x18C` | capacity/end-of-storage | 🟡 |

## PlayerEntry

Tamanho/stride observado:

```text
8 bytes
```

Composição ainda desconhecida.

Hipóteses possíveis:

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

Não tratar nenhuma delas como confirmada.

## Relação conceitual

```text
UnknownWorldLike
    ├── mLocalPlayerIndex
    └── mPlayers
          ↓
      PlayerEntry[]
          ↓
        Player ?
```
