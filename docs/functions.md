# Funções mapeadas

Última atualização: 2026-10-03.

## FUN_006889B0

Endereço: 0x006889B0.

Papel: construtor/inicializador estrutural de World.

~~~asm
006889D6  MOV ESI,ECX
006889DE  CALL FUN_00729360
006889F1  MOV dword ptr [ESI],009D329C
~~~

Estado: ✅.

## FUN_00697800

- Endereço: 0x00697800
- Classe: World
- Vtable: slot 33 / +0x84
- Chama FUN_00735A00 em 0x006979F7
- Mesmo World* em ECX

Estado: ✅.

## FUN_00735A00

- Endereço: 0x00735A00
- this = World*
- +0x174 → mLocalPlayerIndex
- +0x184 → mPlayers.begin
- +0x188 → mPlayers.end

Estado: ✅.

## FUN_0062E5D0

- Endereço: 0x0062E5D0
- this = Owner*

~~~asm
0062E5FA  MOV [EBP + local_20],ECX
~~~

Cria World e instala o ponteiro em Owner+0xC8.

Estado: ✅ para a relação estrutural.

## FUN_00659750

~~~asm
0065979D  MOV ECX,[DAT_00AF2C58]
006597A8  CALL FUN_0062E5D0
~~~

Conclusão: [DAT_00AF2C58] = Owner*.

Estado: ✅.

## WorldPlayerGaia

Vtable estática: 0x009DC8A0.

~~~text
vtable[-1]     = 0x00A23F58
TypeDescriptor = 0x00ACBAF4
RTTI name      = WorldPlayerGaia
slot 0         = FUN_00758C90
~~~

Runtime observado: 0x18AA701C → vtable 0x0136C8A0.

Estado: ✅.

## WorldPlayerHumanOrCoop

Vtable estática: 0x009DCAE4.

~~~text
vtable[-1]     = 0x00A23FA8
TypeDescriptor = 0x00ACBB14
RTTI name      = WorldPlayerHumanOrCoop
~~~

Runtime observado: 0x146EF234 → vtable 0x0136CAE4.

Estado: ✅.

## WorldPlayerComputer

Vtable estática: 0x009DC618.

~~~text
vtable[-1]     = 0x00A23B44
TypeDescriptor = 0x00ACB750
RTTI name      = WorldPlayerComputer
slot 0         = FUN_00755A40
~~~

Runtime observado: 0x18B1F09C → vtable 0x0136C618.

Estado: ✅.

## FUN_00729360

Chamada no início de FUN_006889B0 antes da escrita da vtable de World.

Hipótese: inicializador/construtor de BaseWorld.

Confiança: 🟡.

## Nota de lifecycle

Uma nova instância de World pode já possuir a vtable correta antes de mLocalPlayerIndex e mPlayers estarem prontos para leitura de gameplay.

A futura API deve separar:

~~~text
World pointer válido
~~~

de:

~~~text
World gameplay-ready
~~~
