# Engenharia reversa — estado consolidado

Última atualização: 2026-10-04.

## Alvo

- Executável: `AoK HD.exe`
- Versão: `5.8.INT`
- Plataforma: Windows x86 / PE32
- ImageBase estático: `0x00400000`
- Arquitetura: i386
- Timestamp PE registrado: 22/08/2018
- SHA-256: `CBD10D81B93601FFB26773D250B7478969DA92D70792A40AE423294202E14650`

## Relocação usada no x32dbg

Na execução analisada em 04/10/2026:

~~~text
moduleBase runtime = 0x00410000
deslocamento       = +0x00010000
~~~

Exemplos:

~~~text
static FUN_00598700 → runtime 0x005A8700
static FUN_00598750 → runtime 0x005A8750
static DAT_00AF2C58 → runtime 0x00B02C58
static DAT_00AF1AD0 → runtime 0x00B01AD0
~~~

Sempre converter endereço estático para runtime pelo RVA da execução atual.

## World

RTTI confirma:

~~~text
World
  ↓
BaseWorld
~~~

Vtable estática: `0x009D329C`.

Tamanho alocado observado: `0x2DC` bytes.

`FUN_006889B0` recebe `this` em ECX e instala explicitamente a vtable:

~~~asm
006889D6  MOV ESI,ECX
006889DE  CALL FUN_00729360
006889F1  MOV dword ptr [ESI],009D329C
~~~

## Cadeia global

~~~text
[DAT_00AF2C58] = Owner*
Owner + 0xC8   = World*
~~~

RVA do global: `0x006F2C58`.

Na execução com base `0x00410000`:

~~~text
Owner global runtime = 0x00B02C58
~~~

## World + 0x174

Associado diretamente à string `mLocalPlayerIndex.raw()`.

Estado: ✅ `mLocalPlayerIndex`.

Importante: esse índice interno não deve ser confundido automaticamente com a coluna visual **“Jog.”** do lobby.

## World + 0x184/+0x188/+0x18C

Interpretação confirmada:

~~~text
+0x184 → mPlayers.begin
+0x188 → mPlayers.end
+0x18C → mPlayers.end_of_storage
stride = 8
~~~

Logo:

~~~text
size = (end - begin) / 8
~~~

### Partida com 8 jogadores

Runtime observado:

~~~text
World*                  = 0x18839030
World+0x184             = 0x1865C968
World+0x188             = 0x1865C9B0
World+0x18C             = 0x1865C9B0
size                    = 9
capacity                = 9
~~~

A presença de 9 entries com 8 jogadores reais é explicada por:

~~~text
mPlayers[0] = Gaia
mPlayers[1] = Jogador real 1
...
mPlayers[8] = Jogador real 8
~~~

## Layout das entries de mPlayers

Cada entry tem 8 bytes:

~~~cpp
struct PlayerEntry {
    WorldPlayer* object;  // +0x0
    void* controlBlock;   // +0x4
};
~~~

Na partida de 8 jogadores:

~~~text
index  object      controlBlock
0      0x0F588734  0x0F588728
1      0x1469485C  0x14694850
2      0x184A501C  0x184A5010
3      0x151FA41C  0x151FA410
4      0x13B4941C  0x13B49410
5      0x185E901C  0x185E9010
6      0x138E564C  0x138E5640
7      0x143F701C  0x143F7010
8      0x18C7584C  0x18C75840
~~~

Em todas:

~~~text
object = controlBlock + 0x0C
~~~

RTTI também contém `_Ref_count_obj<...>` para os tipos concretos, reforçando fortemente:

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

### Vtables estáticas

~~~text
WorldPlayerGaia        = 0x009DC8A0
WorldPlayerHumanOrCoop = 0x009DCAE4
WorldPlayerComputer    = 0x009DC618
~~~

Com base runtime `0x00410000`:

~~~text
WorldPlayerGaia        = 0x009EC8A0
WorldPlayerHumanOrCoop = 0x009ECAE4
WorldPlayerComputer    = 0x009EC618
~~~

### Confirmação da partida com 8 jogadores

Primeiros DWORDs:

~~~text
mPlayers[0] @ 0x0F588734 → 0x009EC8A0 → WorldPlayerGaia
mPlayers[1] @ 0x1469485C → 0x009ECAE4 → WorldPlayerHumanOrCoop
mPlayers[2] @ 0x184A501C → 0x009EC618 → WorldPlayerComputer
mPlayers[3] @ 0x151FA41C → 0x009EC618 → WorldPlayerComputer
mPlayers[4] @ 0x13B4941C → 0x009EC618 → WorldPlayerComputer
mPlayers[5] @ 0x185E901C → 0x009EC618 → WorldPlayerComputer
mPlayers[6] @ 0x138E564C → 0x009EC618 → WorldPlayerComputer
mPlayers[7] @ 0x143F701C → 0x009EC618 → WorldPlayerComputer
mPlayers[8] @ 0x18C7584C → 0x009EC618 → WorldPlayerComputer
~~~

Estado: ✅.

## Prefixo comum dos player objects

Nas amostras:

~~~text
[player+0x04] = World*
~~~

Valores observados em `player+0x08`:

~~~text
Gaia        = 2
HumanOrCoop = 1
Computer    = 3
~~~

Em múltiplas mudanças da coluna visual **“Jog.”**, o Human continuou com `+0x08 = 1`.

Conclusão:

~~~text
player+0x08 != coluna “Jog.” / número-cor do lobby
~~~

A semântica exata do campo ainda é aberta.

## SlotRecord

A tabela usada pelos fluxos de configuração tem stride `0x28`.

Exemplos runtime:

~~~text
SlotRecord jogador 1 = 0x00B01AF8
SlotRecord jogador 2 = 0x00B01B20
próximo              = 0x00B01B48
...
~~~

O endereço `0x00B01AD0` não deve ser tratado como “SlotRecord da Gaia”. O dump mostrou que `0x00B01AF4` contém um ponteiro global usado pelas rotinas auxiliares, e os registros normais começam em `0x00B01AF8`.

### SlotRecord + 0x18

Confirmado como índice usado para resolver o config correspondente:

~~~text
0x00B01AF8 + 0x18 = 0
0x00B01B20 + 0x18 = 1
0x00B01B48 + 0x18 = 2
...
~~~

Nome provisório:

~~~text
configIndex
~~~

Não confundir com `worldPlayerIndex`.

## FUN_00598310 / runtime 0x005A8310

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
    int i = slot->configIndex;          // +0x18
    auto base = *GetConfigBase(...);    // via global em 0x00B01AF4
    return base + 0xB0 + i * 0x68;
}
~~~

Stride de `ResolvedPlayerConfig`: `0x68`.

## FUN_00598750 / runtime 0x005A8750

Assembly runtime:

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

Estado: ✅.

## FUN_00598380 / runtime 0x005A8380

~~~asm
005A8380  CALL 005A8310
005A8385  MOV EAX,[EAX+54]
005A8388  RET
~~~

Logo:

~~~text
ResolvedPlayerConfig + 0x54 = humanity
~~~

Estado: ✅.

## FUN_00598880 / runtime 0x005A8880

Essa função chama `FUN_00598380` e testa o resultado contra:

~~~text
{2, 3, 4}
~~~

Interpretação:

~~~cpp
bool IsResolvablePlayerSlot(SlotRecord* slot) {
    int humanity = GetHumanity(slot);
    return humanity == 2 || humanity == 3 || humanity == 4;
}
~~~

O nome é provisório; a lógica do conjunto é confirmada.

## FUN_00598700 / runtime 0x005A8700

Fluxo confirmado:

~~~asm
005A8706  MOV EDI,ECX
005A8708  CALL 005A8880
...
005A8711  MOV EAX,[00B02C58]
005A8716  MOV ESI,[EAX+C8]
...
005A8723  MOV ECX,EDI
005A8726  CALL 005A8750
005A872B  MOV EAX,[EBP-4]
...
005A8736  MOV ECX,[ESI+184]
005A873C  MOV EAX,[ECX+EAX*8]
~~~

Interpretação:

~~~cpp
WorldPlayer* ResolveWorldPlayerFromSlot(SlotRecord* slot) {
    if (!IsResolvablePlayerSlot(slot))
        return nullptr;

    World* world = Owner->world;

    int i = GetWorldPlayerIndex(slot);
    if (i == INVALID)
        return nullptr;

    return world->mPlayers[i].get();
}
~~~

### Validação dinâmica

Humano:

~~~text
SlotRecord*       = 0x00B01AF8
configIndex       = 0
worldPlayerIndex  = 1
→ mPlayers[1]     = WorldPlayerHumanOrCoop
~~~

IA:

~~~text
SlotRecord*       = 0x00B01B20
configIndex       = 1
worldPlayerIndex  = 2
→ mPlayers[2]     = WorldPlayerComputer
~~~

Estado: ✅.

## ResolvedPlayerConfig: amostra de 8 configs

Base resolvida na amostra:

~~~text
base = 0x18637148
~~~

Campos observados:

| configIndex | worldPlayerIndex | humanity | interpretação observada |
|---:|---:|---:|---|
| 0 | 1 | 2 | humano |
| 1 | 2 | 4 | computador |
| 2 | -1 | 1 | sem WorldPlayer |
| 3 | -1 | 1 | sem WorldPlayer |
| 4 | -1 | 1 | sem WorldPlayer |
| 5 | -1 | 1 | sem WorldPlayer |
| 6 | -1 | 1 | sem WorldPlayer |
| 7 | -1 | 1 | sem WorldPlayer |

`humanity=3` ainda não foi observado diretamente e não deve ser nomeado.

## Gaia

Confirmado:

~~~text
World.mPlayers[0] → WorldPlayerGaia
~~~

Não há evidência de que Gaia tenha um `SlotRecord` normal na tabela dos jogadores configuráveis do lobby.

Isso é coerente com o fato de Gaia não ser um jogador configurável normal; ela ocupa uma entry interna reservada em `World.mPlayers[0]`.

## Ciclo de vida

`World*`, `mPlayers` e objetos de player podem ser recriados.

Exemplo: um `World*` que antes era válido deixou de ter um `mPlayers.begin` plausível após mudança de estado, enquanto uma nova execução de `FUN_00598700` revelou um novo `World*`.

Conclusão operacional:

~~~text
não cachear WorldPlayer* indefinidamente
resolver a cadeia novamente após mudança de partida/setup
~~~

## Próximo passo de maior retorno

Localizar o campo responsável pela coluna visual:

~~~text
“Jog.” = 1..8 / número-cor atribuída
~~~

Esse campo já foi separado de:

~~~text
World.mPlayers index
mLocalPlayerIndex
SlotRecord.configIndex
ResolvedPlayerConfig.worldPlayerIndex
player+0x08
~~~

Depois: consolidar readiness e iniciar recursos `Food/Wood/Gold/Stone`.
