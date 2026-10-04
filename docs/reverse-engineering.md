# Engenharia reversa — estado consolidado

Última atualização: 2026-10-03.

## Alvo

- Executável: AoK HD.exe
- Versão: 5.8.INT
- Plataforma: Windows x86 / PE32
- ImageBase estático: 0x00400000
- Arquitetura: i386
- Timestamp PE registrado: 22/08/2018
- SHA-256: CBD10D81B93601FFB26773D250B7478969DA92D70792A40AE423294202E14650

## World

RTTI confirma:

~~~text
World
  ↓
BaseWorld
~~~

Vtable estática: 0x009D329C.

Tamanho alocado observado: 0x2DC bytes.

FUN_006889B0 recebe this em ECX e instala explicitamente a vtable:

~~~asm
006889D6  MOV ESI,ECX
006889DE  CALL FUN_00729360
006889F1  MOV dword ptr [ESI],009D329C
~~~

## FUN_00697800 / FUN_00735A00

FUN_00697800 está no slot virtual 33 de World (+0x84) e chama FUN_00735A00 em 0x006979F7.

O fluxo de ECX confirma que ambas operam sobre o mesmo World*.

### World + 0x174

Associado diretamente à string mLocalPlayerIndex.raw() e validado em runtime:

~~~text
World = 0x18431CD8
WORD[World+0x174] = 1
~~~

Estado: ✅ mLocalPlayerIndex.

### World + 0x184/+0x188/+0x18C

Runtime:

~~~text
World+0x184 = 0x105010E0
World+0x188 = 0x105010F8
World+0x18C = 0x105010F8
stride      = 8
size        = 3
capacity    = 3
~~~

Interpretação:

~~~text
+0x184 → begin
+0x188 → end
+0x18C → end_of_storage
~~~

O terceiro ponteiro é estruturalmente compatível com o layout std::vector MSVC x86.

## Owner/root e cadeia global

FUN_0062E5D0 salva ECX em local_20:

~~~asm
0062E5FA  MOV [EBP + local_20],ECX
~~~

e armazena o novo World em Owner+0xC8:

~~~asm
0062E671  MOV EAX,[EBP + local_20]
0062E674  ADD EAX,0xC8
...
0062E68B  MOV [EAX],ECX
~~~

FUN_00659750 revela a origem global:

~~~asm
0065979D  MOV ECX,[DAT_00AF2C58]
006597A8  CALL FUN_0062E5D0
~~~

Logo:

~~~text
[DAT_00AF2C58] = Owner*
Owner + 0xC8   = World*
~~~

RVA do global: 0x006F2C58.

## Validação runtime da cadeia

~~~text
moduleBase = 0x00D90000

[moduleBase+0x006F2C58] = 0x04C7A750
[0x04C7A750+0xC8]       = 0x18431CD8
[0x18431CD8]            = 0x0136329C
moduleBase+0x005D329C   = 0x0136329C
~~~

Estado: ✅.

## Layout das entries de mPlayers

~~~text
entry[0] = { 0x18AA701C, 0x18AA7010 }
entry[1] = { 0x146EF234, 0x146EF228 }
entry[2] = { 0x18B1F09C, 0x18B1F090 }
~~~

Nas três:

~~~text
object = controlBlock + 0x0C
~~~

Modelo de trabalho:

~~~cpp
struct PlayerEntry {
    WorldPlayer* object;
    void* controlBlock;
}; // 8 bytes
~~~

RTTI STL encontrado:

~~~text
_Ref_count_obj<WorldPlayerGaia>
_Ref_count_obj<WorldPlayerHumanOrCoop>
_Ref_count_obj<WorldPlayerComputer>
~~~

Isso reforça fortemente:

~~~text
mPlayers ≈ std::vector<std::shared_ptr<WorldPlayer>>
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

### Gaia

~~~text
object runtime   = 0x18AA701C
vtable runtime   = 0x0136C8A0
vtable static    = 0x009DC8A0
vtable[-1]       = 0x00A23F58
TypeDescriptor   = 0x00ACBAF4
RTTI name        = WorldPlayerGaia
~~~

Estado: ✅.

### HumanOrCoop

~~~text
object runtime   = 0x146EF234
vtable runtime   = 0x0136CAE4
vtable static    = 0x009DCAE4
vtable[-1]       = 0x00A23FA8
TypeDescriptor   = 0x00ACBB14
RTTI name        = WorldPlayerHumanOrCoop
~~~

Estado: ✅.

### Computer

~~~text
object runtime   = 0x18B1F09C
vtable runtime   = 0x0136C618
vtable static    = 0x009DC618
vtable[-1]       = 0x00A23B44
TypeDescriptor   = 0x00ACB750
RTTI name        = WorldPlayerComputer
~~~

Estado: ✅.

Assim, na partida observada:

~~~text
mPlayers[0] → WorldPlayerGaia
mPlayers[1] → WorldPlayerHumanOrCoop
mPlayers[2] → WorldPlayerComputer
~~~

## Prefixo comum dos player objects

Nas amostras:

~~~text
[player+0x04] = World*
~~~

Em player+0x08 foram observados:

~~~text
Gaia        = 2
HumanOrCoop = 1
Computer    = 3
~~~

Esse campo continua 🟡: candidato a id/índice/discriminador, ainda sem segunda validação independente.

## Ciclo de vida ao criar nova partida

Ao mudar para uma nova partida:

~~~text
Owner* = 0x04C7A750
~~~

permaneceu estável na amostra, mas Owner+0xC8 passou a apontar para:

~~~text
World* = 0x102F4108
[World] = 0x0136329C
~~~

A nova instância já tinha a vtable correta, porém World+0x174 ainda não refletia o slot local esperado naquele momento.

Conclusão:

~~~text
World* válido por vtable != World pronto para gameplay
~~~

O futuro adapter deve possuir uma verificação explícita de readiness.

## Próximo passo de maior retorno

Com a segunda partida totalmente carregada:

1. reler Owner+0xC8;
2. validar a vtable de World;
3. ler World+0x174;
4. resolver mPlayers[localIndex];
5. confirmar WorldPlayerHumanOrCoop;
6. comparar player+0x08 com o índice local.

Depois: Food/Wood/Gold/Stone.
