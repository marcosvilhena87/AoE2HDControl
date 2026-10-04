# Engenharia reversa — estado consolidado

Última atualização: 2026-10-03.

## Alvo

- Executável: `AoK HD.exe`
- Versão: `5.8.INT`
- Plataforma: Windows x86 / PE32
- ImageBase estático: `0x00400000`
- Arquitetura: i386
- Timestamp PE registrado: 22/08/2018
- SHA-256 registrado: `CBD10D81B93601FFB26773D250B7478969DA92D70792A40AE423294202E14650`
- PDB referenciado: `AoK HD.pdb`

O binário apresenta C++ MSVC com RTTI, herança, métodos `__thiscall` e vtables.

## Convenção x86 importante

Em métodos de instância MSVC/x86:

```text
ECX = this
```

Esse padrão foi confirmado nas funções centrais investigadas.

---

## Classe World — identificação confirmada

RTTI:

```text
World
  ↓
BaseWorld
```

Vtable estática:

```text
0x009D329C
```

Foram observados 37 ponteiros de função.

Estado:

- `World`: ✅ RTTI
- `World : BaseWorld`: ✅ RTTI
- vtable `0x009D329C`: ✅
- 37 slots observados: ✅

---

## FUN_006889B0 — construção/inicialização de World

Trecho-chave:

```asm
006889D6  MOV ESI,ECX
006889DE  CALL FUN_00729360
006889F1  MOV dword ptr [ESI],009D329C
```

A função:

- recebe `this` em `ECX`;
- chama um inicializador/base antes;
- instala explicitamente a vtable de `World`;
- é chamada imediatamente após alocação de `0x2DC` bytes e `memset`.

Fluxo de criação observado:

```text
operator_new(0x2DC)
        ↓
memset(..., 0, 0x2DC)
        ↓
ECX = bloco
        ↓
FUN_006889B0
        ↓
World*
```

Estado:

- alocação `0x2DC`: ✅
- escrita da vtable de `World`: ✅
- papel de construtor/inicializador de `World`: ✅ estruturalmente
- overload/nome-fonte exato: 🟡

---

## FUN_00697800 — método virtual de World

Endereço:

```text
0x00697800
```

Na vtable:

```text
índice 33
offset = 33 * 4 = 0x84
```

Logo:

```text
World::vtable + 0x84 → FUN_00697800
```

Dentro dela:

```text
0x006979F7 → FUN_00735A00
```

Antes da chamada:

```asm
MOV ECX,[EBP + local_6c]
...
CALL FUN_00735A00
```

E no prólogo de `FUN_00697800`:

```asm
MOV [EBP + local_6c],ECX
```

Portanto as duas funções recebem o mesmo `World*`.

Estado: ✅.

---

## FUN_00735A00 — campos de World

No prólogo:

```asm
MOV [EBP + local_c0],ECX
```

Logo `local_c0 = this = World*`.

### World + 0x174 — mLocalPlayerIndex

Trecho:

```asm
MOV EAX,[EBP + local_c0]
MOV EAX,[EAX + 0x174]
MOVZX EAX,AX
...
PUSH "mLocalPlayerIndex.raw()"
```

Validação runtime:

```text
World = 0x18431CD8
World + 0x174 = 0x18431E4C
WORD[World+0x174] = 1
```

A partida foi executada com o usuário como Player 1.

Estado:

- offset `+0x174`: ✅
- associação a `mLocalPlayerIndex`: ✅
- consumo observado dos 16 bits baixos: ✅
- tipo C++ exato do wrapper `PlayerIndex`: 🟡

### World + 0x184 / +0x188 — mPlayers begin/end

A função usa os dois campos e calcula quantidade com stride 8.

Validação runtime:

```text
mPlayers.begin = 0x105010E0
mPlayers.end   = 0x105010F8
difference     = 0x18
stride         = 8
size           = 3
```

A partida tinha:

```text
1 humano + 1 IA + Gaia = 3 entradas
```

Estado:

- `World+0x184 = begin`: ✅
- `World+0x188 = end`: ✅
- stride 8: ✅
- coleção `mPlayers`: ✅ pela semântica/asserts + runtime
- `World+0x18C` como capacity/end-of-storage: 🟡 ainda não validado

---

## Owner/root e cadeia global até World

### FUN_0062E5D0

Prólogo:

```asm
0062E5FA  MOV [EBP + local_20],ECX
```

Portanto:

```text
local_20 = this = Owner*
```

No caminho de criação de `World`:

```asm
0062E671  MOV EAX,[EBP + local_20]
0062E674  ADD EAX,0xC8
...
0062E68B  MOV [EAX],ECX
```

Logo:

```text
Owner + 0xC8 → World*
```

O ponteiro antigo é destruído por chamada virtual através do slot 0, comportamento compatível com propriedade polimórfica.

### Origem do Owner

Em `FUN_00659750`:

```asm
0065979D  MOV ECX,[DAT_00AF2C58]
006597A3  PUSH 1
006597A5  PUSH EAX
006597A8  CALL FUN_0062E5D0
```

Portanto:

```text
[DAT_00AF2C58] = Owner*
```

Estado:

- global `DAT_00AF2C58` fornece o `Owner*` nesse caminho: ✅
- `Owner + 0xC8 → World*`: ✅
- classe/nome real de `Owner`: 🟡 ainda desconhecido

---

## Validação dinâmica da cadeia completa

Execução analisada:

```text
moduleBase = 0x00D90000
```

RVA do global:

```text
0x00AF2C58 - 0x00400000 = 0x006F2C58
```

RVA da vtable de `World`:

```text
0x009D329C - 0x00400000 = 0x005D329C
```

Valores observados:

```text
[moduleBase + 0x006F2C58] = 0x04C7A750   // Owner*
[0x04C7A750 + 0xC8]       = 0x18431CD8   // World*
[0x18431CD8]              = 0x0136329C   // vtable
moduleBase + 0x005D329C   = 0x0136329C   // esperado
```

A igualdade da vtable fornece validação independente da cadeia:

```text
module + 0x6F2C58
      ↓
    Owner*
      ↓ +0xC8
    World*
      ↓ +0
World vtable
```

Estado: ✅ validado no x32dbg.

---

## mPlayers — elementos de 8 bytes

Intervalo observado:

```text
0x105010E0 .. 0x105010F7
```

Três entradas:

```text
entry[0] @ 0x105010E0
  +0 = 0x18AA701C
  +4 = 0x18AA7010

entry[1] @ 0x105010E8
  +0 = 0x146EF234
  +4 = 0x146EF228

entry[2] @ 0x105010F0
  +0 = 0x18B1F09C
  +4 = 0x18B1F090
```

Em todas:

```text
entry.ptr = entry.control + 0x0C
```

No primeiro control block:

```text
0x18AA7010 +0x00 → 0x0136C88C
0x18AA7010 +0x04 → 1
0x18AA7010 +0x08 → 1
0x18AA7010 +0x0C → início do objeto (0x18AA701C)
```

Esse padrão é fortemente compatível com implementação MSVC x86 de `std::shared_ptr<T>` criada com objeto inline no control block/`make_shared`.

Modelo de trabalho:

```cpp
struct PlayerEntry {
    void* object;        // +0
    void* controlBlock;  // +4
}; // 8 bytes
```

Estado:

- dois ponteiros por entrada: ✅ observado
- relação objeto = control + 0x0C: ✅ nas três entradas observadas
- interpretação como `std::shared_ptr<T>`: 🟢 muito provável
- tipo template exato: 🟡

---

## RTTI do objeto apontado por mPlayers

Nos três objetos apontados pelas entradas foi observado o mesmo primeiro DWORD:

```text
runtime vtable = 0x0136C8A0
```

Com `moduleBase = 0x00D90000`:

```text
RVA           = 0x005DC8A0
static vtable = 0x009DC8A0
```

No Ghidra:

```text
0x009DC89C → 0x00A23F58  // RTTI Complete Object Locator
0x009DC8A0 → FUN_00758C90
0x009DC8A4 → FUN_00748A40
```

O locator contém:

```text
0x00A23F64 → 0x00ACBAF4  // TypeDescriptor
0x00A23F68 → 0x00A23F6C  // ClassHierarchyDescriptor
```

TypeDescriptor:

```text
0x00ACBAFC → ".?AVWorldPlayerGaia@@"
```

Logo a vtable `0x009DC8A0` resolve por RTTI para:

```text
WorldPlayerGaia
```

O hierarchy descriptor registra 3 classes.

Importante: **as três entradas runtime observadas apresentaram essa mesma vtable primária**. Portanto, não é seguro concluir que entry[1] e entry[2] sejam diretamente `WorldPlayerHumanOrCoop` e `WorldPlayerComputer`. A distinção de papel Gaia/humano/IA provavelmente está em outro campo, subobjeto ou relação ainda não mapeada.

RTTI adicional encontrado nas proximidades:

```text
0x00ACBB14 → ".?AVWorldPlayerHumanOrCoop@@"
0x00ACBB3C → ".?AVWorldPlayerScenarioEditorPhantom@@"
```

Essas classes existem no binário, mas ainda não foram ligadas diretamente às entradas runtime analisadas.

---

## Relações estruturais consolidadas

```text
DAT_00AF2C58
      │
      ▼
    Owner*
      │
      └── +0xC8 ─────► World (0x2DC bytes)
                         │
                         ├── vtable 0x009D329C
                         │      └── slot 33 / +0x84
                         │             └── FUN_00697800
                         │                    └── FUN_00735A00
                         │
                         ├── +0x174  mLocalPlayerIndex
                         ├── +0x184  mPlayers.begin
                         ├── +0x188  mPlayers.end
                         └── +0x18C  ? capacity/end-of-storage
                                     
mPlayers entry (8 bytes)
      │
      ├── +0x0 → object pointer
      └── +0x4 → control block
```

---

## ASLR / RVA

Nunca depender de endereço runtime absoluto.

```cpp
runtimeAddress = moduleBase + (staticAddress - 0x00400000);
```

Offsets/RVAs úteis:

```text
Owner global RVA = 0x006F2C58
World vtable RVA = 0x005D329C
```

---

## Próximo passo de maior retorno

A cadeia até `World*` já está validada. O novo gargalo é **resolver o jogador local e o papel de cada player entry**.

Prioridade:

1. investigar campos próximos ao início dos três objetos apontados;
2. localizar o discriminador que separa Gaia / humano / IA;
3. relacionar `mLocalPlayerIndex` à entrada correta;
4. identificar RTTI/vtables auxiliares de `WorldPlayerHumanOrCoop` e `WorldPlayerComputer`;
5. usar recursos visíveis (Food/Wood/Gold/Stone) para confirmar o `LocalPlayer*`.

---

## Regra de documentação

Para cada descoberta registrar:

```text
Endereço
Função
Assembly relevante
Pseudocódigo
Objeto this
Offset
Significado provável
Evidência
Nível de confiança
Próxima validação
```

Classificação:

- ✅ Confirmado
- 🟢 Muito provável
- 🟡 Hipótese
- 🔴 Descartado
