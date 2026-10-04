# Funções mapeadas

Última atualização: 2026-10-03.

## FUN_006889B0

- Endereço: `0x006889B0`
- Papel: construtor/inicializador estrutural de `World`
- Convenção: `ECX = this`
- Evidência principal:

```asm
006889D6  MOV ESI,ECX
006889DE  CALL FUN_00729360
006889F1  MOV dword ptr [ESI],009D329C
```

- Contexto de criação:
  - chamada após `operator_new(0x2DC)`;
  - bloco zerado com `memset(...,0,0x2DC)`;
  - instala a vtable confirmada de `World`.
- Confiança:
  - inicialização de uma instância de `World`: ✅
  - nome-fonte/overload exato do construtor: 🟡

## FUN_00697800

- Endereço: `0x00697800`
- Classe: `World`
- Convenção: `__thiscall` compatível
- `this`: `World*` em `ECX`
- Vtable:
  - início: `0x009D329C`
  - índice: `33`
  - offset: `+0x84`
- Call site:
  - `0x006979F7 → FUN_00735A00`
- Estado: ✅ método virtual de `World`

## FUN_00735A00

- Endereço: `0x00735A00`
- `this`: `World*`
- Prólogo salva `ECX` em `local_c0`
- Campos mapeados:
  - `+0x174 → mLocalPlayerIndex`
  - `+0x184 → mPlayers.begin`
  - `+0x188 → mPlayers.end`
- Evidência:
  - string/assert `mLocalPlayerIndex.raw()`;
  - cálculo de tamanho da coleção com stride 8;
  - validação runtime de índice local e quantidade de players.
- Confiança: ✅ para os offsets acima.

## FUN_0062E5D0

- Endereço: `0x0062E5D0`
- `this`: objeto owner/root ainda sem nome de classe
- Prólogo:

```asm
0062E5FA  MOV [EBP + local_20],ECX
```

Logo:

```text
local_20 = Owner*
```

- Cria `World`:
  - `operator_new(0x2DC)`
  - `memset`
  - `FUN_006889B0`
- Armazena o novo ponteiro em:

```text
Owner + 0xC8
```

Trecho:

```asm
0062E671  MOV EAX,[EBP + local_20]
0062E674  ADD EAX,0xC8
...
0062E68B  MOV [EAX],ECX
```

- Se houver objeto antigo, chama slot virtual 0 com argumento `1`.
- Confiança:
  - `Owner+0xC8 → World*`: ✅
  - semântica exata da função (new game/load/create session): 🟡

## FUN_00659750

- Endereço: `0x00659750`
- Importância: caller que revela a origem global do `Owner*`

Trecho:

```asm
0065979D  MOV ECX,[DAT_00AF2C58]
006597A3  PUSH 0x1
006597A5  PUSH EAX
006597A8  CALL FUN_0062E5D0
```

Conclusão:

```text
[DAT_00AF2C58] = Owner*
```

Estado: ✅ para essa cadeia de chamada.

## FUN_00758C90 / vtable 0x009DC8A0

A vtable estática em:

```text
0x009DC8A0
```

tem:

```text
slot 0 → FUN_00758C90
slot 1 → FUN_00748A40
```

Seu RTTI Complete Object Locator está em:

```text
0x00A23F58
```

e o TypeDescriptor resolve para:

```text
.?AVWorldPlayerGaia@@
```

Estado: ✅ vtable associada por RTTI a `WorldPlayerGaia`.

Observação runtime importante: os três objetos apontados pelas três entradas observadas de `mPlayers` apresentaram essa mesma vtable primária. A razão/semântica ainda está aberta; não classificar entry[1]/entry[2] como Human/Computer apenas pela posição.

## FUN_00729360

- Chamada no início de `FUN_006889B0`, antes da escrita da vtable de `World`.
- Hipótese: inicializador/construtor da base `BaseWorld` ou rotina relacionada.
- Confiança: 🟡 até mapear RTTI/vtable writes internos.

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
