# Funções mapeadas

Última atualização: 2026-10-03.

## FUN_006889B0

- Endereço: `0x006889B0`
- Papel atual: inicializador/construtor fortemente associado a `World`
- Evidência:
  - chamada imediatamente após `operator_new(0x2DC)`;
  - o bloco recém-alocado é zerado/inicializado antes da chamada;
  - o resultado entra na cadeia que termina em `owner+0xC8 = World*`.
- Interpretação:
  - `World::World` ou inicializador principal de `World`.
- Confiança: 🟢 muito provável
- Próxima validação:
  - localizar escrita da vtable `0x009D329C`;
  - mapear classes-base/inicializadores chamados;
  - confirmar o objeto retornado/propagado.

## FUN_00697800

- Endereço: `0x00697800`
- Classe: `World`
- Convenção: provável `__thiscall`
- `this`: `World*` em `ECX`
- Vtable:
  - início: `0x009D329C`
  - índice: `33`
  - offset: `+0x84`
- Call site relevante:
  - `0x006979F7 → FUN_00735A00`
- Status:
  - método virtual de `World`: ✅ confirmado
- Observação:
  - o fluxo indica que `FUN_00735A00` recebe o mesmo `World*`.

## FUN_00735A00

- Endereço: `0x00735A00`
- Convenção provável: `__thiscall`
- `this`: `World*`
- Campos relevantes:
  - `+0x174`
  - `+0x184`
  - `+0x188`
- Semântica associada:
  - `+0x174 ≈ mLocalPlayerIndex`
  - `+0x184/+0x188 ≈ mPlayers begin/end`
  - stride dos elementos: 8 bytes
- Chamador conhecido:
  - `FUN_00697800` via `0x006979F7`
- Outra referência observada:
  - `0x009DB0A8`
- Confiança:
  - método sobre `World`: ✅
  - nomes exatos dos campos: 🟢

## FUN_00659750

- Endereço: `0x00659750`
- Já inspecionada no Ghidra.
- Variáveis locais observadas incluem:
  - `local_8`
  - `local_10`
  - `local_14`
- Ainda não há evidência suficiente para atribuir papel semântico seguro.
- Confiança: 🟡 não classificada semanticamente
- Regra:
  - não conectá-la a `World`, `Player` ou comandos sem nova evidência.

## Rotina contendo 0x0062E65A

A função que contém essa região executa um caminho de criação/substituição de `World`:

```text
operator_new(0x2DC)
        ↓
memset / inicialização
        ↓
FUN_006889B0
        ↓
novo World*
        ↓
owner + 0xC8
```

Se já existir um ponteiro antigo em `owner+0xC8`, ele é destruído por chamada virtual através do primeiro slot da vtable.

Próxima investigação prioritária:

> descobrir de onde vem `local_20` / `owner`.

## Modelo para novas funções

```text
Nome:
Endereço:
Classe:
Convenção:
this:
Vtable slot:
Callers:
Callees:
Offsets acessados:
Strings/RTTI associados:
Assembly-chave:
Pseudocódigo:
Hipótese:
Confiança:
Próxima validação:
```
