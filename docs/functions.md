# Funções mapeadas

Última atualização: 2026-10-04.

## Convenção de endereços

Os endereços principais abaixo são os endereços estáticos do binário com ImageBase `0x00400000`.

Na execução analisada em 04/10/2026, `moduleBase = 0x00410000`, portanto os endereços runtime estavam deslocados em `+0x10000`.

## FUN_006889B0

Endereço estático: `0x006889B0`.

Papel: construtor/inicializador estrutural de `World`.

~~~asm
006889D6  MOV ESI,ECX
006889DE  CALL FUN_00729360
006889F1  MOV dword ptr [ESI],009D329C
~~~

Estado: ✅.

## FUN_00697800

- Endereço: `0x00697800`
- Classe: `World`
- Vtable: slot 33 / `+0x84`
- Chama `FUN_00735A00`
- Mesmo `World*` em ECX

Estado: ✅.

## FUN_00735A00

- Endereço: `0x00735A00`
- `this = World*`
- `+0x174 → mLocalPlayerIndex`
- `+0x184 → mPlayers.begin`
- `+0x188 → mPlayers.end`
- serializa dados relacionados a slots, incluindo `worldPlayerIndex`, `humanity` e nome.

Estado: ✅ para os campos acima.

## FUN_0062E5D0

- Endereço: `0x0062E5D0`
- `this = Owner*`
- cria/instala `World*` em `Owner+0xC8`.

Estado: ✅ para a relação estrutural.

## FUN_00659750

~~~asm
0065979D  MOV ECX,[DAT_00AF2C58]
006597A8  CALL FUN_0062E5D0
~~~

Conclusão:

~~~text
[DAT_00AF2C58] = Owner*
~~~

Estado: ✅.

## FUN_00583980 — GetSlotRecord28

Endereço estático: `0x00583980`.

Assembly:

~~~asm
MOV EAX,[arg]
LEA EAX,[EAX+EAX*4]
LEA EAX,[EAX*8+DAT_00AF1AD0]
RET 4
~~~

Retorna:

~~~text
DAT_00AF1AD0 + index * 0x28
~~~

Nome `GetSlotRecord28` é provisório; o stride `0x28` é confirmado.

Estado: ✅ estrutural.

## FUN_00598310 — ResolveSlotConfig

Endereço estático: `0x00598310`.
Runtime observado: `0x005A8310`.

Assembly runtime:

~~~asm
005A8310  PUSH ESI
005A8311  MOV ESI,[ECX+18]
005A8314  MOV ECX,[00B01AF4]
005A831A  CALL 00593640
005A831F  IMUL ECX,ESI,68
005A8323  MOV EAX,[EAX]
005A8325  ADD ECX,B0
005A832B  ADD EAX,ECX
005A832D  RET
~~~

Interpretação:

~~~cpp
ResolvedPlayerConfig* ResolveSlotConfig(SlotRecord* slot) {
    const int i = slot->configIndex; // +0x18
    return base + 0xB0 + i * 0x68;
}
~~~

Estado: ✅ para a relação `SlotRecord+0x18 → config com stride 0x68`.

## FUN_00598380 — GetHumanity

Endereço estático: `0x00598380`.
Runtime observado: `0x005A8380`.

~~~asm
005A8380  CALL 005A8310
005A8385  MOV EAX,[EAX+54]
005A8388  RET
~~~

Logo:

~~~text
ResolvedPlayerConfig + 0x54 = humanity
~~~

Valores observados:

~~~text
2 → humano
4 → computador
1 → configs sem WorldPlayer na amostra
~~~

`3` ainda não foi observado diretamente.

Estado: ✅ para o offset; 🟡 para enumeração completa.

## FUN_00598750 — GetWorldPlayerIndex

Endereço estático: `0x00598750`.
Runtime observado: `0x005A8750`.

~~~asm
005A8750  PUSH EBP
005A8751  MOV EBP,ESP
005A8753  CALL 005A8310
005A8758  MOV ECX,[EAX+50]
005A875B  MOV EAX,[EBP+8]
005A875E  MOV [EAX],ECX
005A8760  POP EBP
005A8761  RET 4
~~~

Logo:

~~~text
ResolvedPlayerConfig + 0x50 = worldPlayerIndex
~~~

Exemplos:

~~~text
SlotRecord 0x00B01AF8 → 1
SlotRecord 0x00B01B20 → 2
~~~

Estado: ✅.

## FUN_00598880 — predicate de humanity {2,3,4}

Endereço estático: `0x00598880`.
Runtime observado: `0x005A8880`.

A função:

1. chama `FUN_00598380`;
2. compara o resultado contra `2`, `3` e `4`;
3. retorna true se houver match.

Equivalente:

~~~cpp
bool IsResolvablePlayerSlot(SlotRecord* slot) {
    int h = GetHumanity(slot);
    return h == 2 || h == 3 || h == 4;
}
~~~

Nome semântico provisório.

Estado: ✅ para a lógica.

## FUN_00598700 — ResolveWorldPlayerFromSlot

Endereço estático: `0x00598700`.
Runtime observado: `0x005A8700`.

Trecho runtime:

~~~asm
005A8706  MOV EDI,ECX
005A8708  CALL 005A8880
005A870D  TEST AL,AL
005A870F  JE ...
005A8711  MOV EAX,[00B02C58]
005A8716  MOV ESI,[EAX+C8]
...
005A8723  MOV ECX,EDI
005A8726  CALL 005A8750
005A872B  MOV EAX,[EBP-4]
005A872E  CMP [00ABB914],EAX
...
005A8736  MOV ECX,[ESI+184]
005A873C  MOV EAX,[ECX+EAX*8]
~~~

Interpretação consolidada:

~~~cpp
WorldPlayer* ResolveWorldPlayerFromSlot(SlotRecord* slot) {
    if (!IsResolvablePlayerSlot(slot))
        return nullptr;

    World* world = Owner->world;

    int index;
    GetWorldPlayerIndex(slot, &index);

    if (index == invalidSentinel)
        return nullptr;

    return world->mPlayers[index].get();
}
~~~

Validação dinâmica:

~~~text
0x00B01AF8 → worldPlayerIndex 1 → WorldPlayerHumanOrCoop
0x00B01B20 → worldPlayerIndex 2 → WorldPlayerComputer
~~~

Estado: ✅.

## FUN_00593640

Na execução runtime foi chamada em `0x00593640`.

Seu retorno é usado por `FUN_00598310`; o comportamento observado é compatível com fornecer o endereço do ponteiro/base interno localizado a `+0xD54` do objeto de configuração global carregado via `[00B01AF4]`.

Estado: 🟡 para o nome/semântica exata; ✅ para o papel na cadeia observada.

## WorldPlayerGaia

Vtable estática: `0x009DC8A0`.

Runtime com base `0x00410000`: `0x009EC8A0`.

Estado: ✅.

## WorldPlayerHumanOrCoop

Vtable estática: `0x009DCAE4`.

Runtime com base `0x00410000`: `0x009ECAE4`.

Estado: ✅.

## WorldPlayerComputer

Vtable estática: `0x009DC618`.

Runtime com base `0x00410000`: `0x009EC618`.

Estado: ✅.

## FUN_00729360

Chamada no início de `FUN_006889B0` antes da escrita da vtable de `World`.

Hipótese: inicializador/construtor de `BaseWorld`.

Confiança: 🟡.

## Nota de lifecycle

Uma nova instância de `World` pode possuir vtable correta enquanto ponteiros derivados de uma instância anterior já estão obsoletos.

A futura API deve:

~~~text
1. resolver Owner
2. resolver World atual
3. validar readiness
4. resolver mPlayers atual
5. só então resolver WorldPlayer
~~~

Evitar cache longo de `WorldPlayer*`.
