# Funções mapeadas

## FUN_00735A00

- Endereço: `0x00735A00`
- Convenção provável: `__thiscall`
- `this`: recebido em `ECX`
- Campos relevantes:
  - `+0x174`
  - `+0x184`
  - `+0x188`
- Semântica associada:
  - `mLocalPlayerIndex.raw()`
  - coleção fortemente associada a `mPlayers`
- Chamador conhecido:
  - `FUN_00697800` via call site `0x006979F7`
- Outra referência observada:
  - `0x009DB0A8`
- Hipótese:
  - método de World/GameState ou de classe intimamente relacionada.

## FUN_00697800

- Endereço: `0x00697800`
- Convenção provável: `__thiscall`
- `this`: recebido em `ECX`
- Call site relevante:
  - `0x006979F7 → FUN_00735A00`
- XREF de dados relevante:
  - `0x009D3320 → FUN_00697800`
- Hipótese:
  - método virtual ou callback de uma classe ainda não identificada.

## Modelo para novas funções

```text
Nome:
Endereço:
Convenção:
this:
Callers:
Callees:
Offsets acessados:
Strings associadas:
Pseudocódigo:
Hipótese:
Confiança:
Próxima validação:
```
