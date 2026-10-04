# AoE2HDControl

Projeto de engenharia reversa e reconstrução de uma API de controle para **Age of Empires II HD (AoK HD.exe 5.8.INT, x86)**.

## Estado atual

A cadeia central até `World`, o vetor `mPlayers` e a ponte entre configuração/lobby e `WorldPlayer` foram validados com Ghidra, RTTI e x32dbg.

Principais descobertas confirmadas:

- ImageBase estático: `0x00400000`;
- RTTI confirma `World : BaseWorld`;
- vtable estática de `World`: `0x009D329C`;
- `FUN_00697800` é o slot virtual 33 (`+0x84`);
- `FUN_006889B0` instala a vtable de `World`;
- tamanho alocado observado de `World`: `0x2DC` bytes;
- `DAT_00AF2C58` fornece o `Owner*` usado na cadeia validada;
- `Owner + 0xC8 → World*`;
- `World + 0x174 → mLocalPlayerIndex`;
- `World + 0x184/+0x188/+0x18C` formam um layout compatível com `std::vector` x86;
- `mPlayers.size() = (end - begin) / 8`;
- cada entry de `mPlayers` tem 8 bytes e é fortemente compatível com `std::shared_ptr<WorldPlayer>`;
- RTTI confirma `WorldPlayerGaia`, `WorldPlayerHumanOrCoop` e `WorldPlayerComputer`;
- `SlotRecord` tem stride `0x28`;
- `SlotRecord+0x18` seleciona um `ResolvedPlayerConfig`;
- `ResolvedPlayerConfig` tem stride `0x68`;
- `ResolvedPlayerConfig+0x50 = worldPlayerIndex`;
- `ResolvedPlayerConfig+0x54 = humanity`;
- observado: `humanity=2` para humano, `humanity=4` para computador e `humanity=1` em configs sem `WorldPlayer`;
- `FUN_00598700` resolve `SlotRecord → World.mPlayers[worldPlayerIndex]`;
- `FUN_00598880` aceita `humanity ∈ {2,3,4}`;
- Gaia ocupa `World.mPlayers[0]`, mas não há evidência de um `SlotRecord` normal correspondente no lobby.

## Cadeia runtime validada

~~~text
Owner global RVA = 0x006F2C58
World vtable RVA = 0x005D329C

[moduleBase + 0x006F2C58]
        ↓
      Owner*
        ↓ +0xC8
      World*
        ├── +0x000 → vtable = moduleBase + 0x005D329C
        ├── +0x174 → mLocalPlayerIndex
        ├── +0x184 → mPlayers.begin
        ├── +0x188 → mPlayers.end
        └── +0x18C → mPlayers.end_of_storage
~~~

Em uma execução com `moduleBase = 0x00410000`, os endereços runtime relevantes incluíram:

~~~text
Owner global = 0x00B02C58
World vtable = 0x009E329C

FUN_00598700 runtime = 0x005A8700
FUN_00598750 runtime = 0x005A8750
FUN_00598310 runtime = 0x005A8310
FUN_00598380 runtime = 0x005A8380
FUN_00598880 runtime = 0x005A8880
~~~

## World.mPlayers: partida com 8 jogadores

Em uma partida com 1 humano + 7 IAs:

~~~text
World*                 = 0x18839030
mPlayers.begin         = 0x1865C968
mPlayers.end           = 0x1865C9B0
mPlayers.end_of_storage= 0x1865C9B0
size                   = 9
~~~

Logo:

~~~text
mPlayers[0] = Gaia
mPlayers[1] = HumanOrCoop
mPlayers[2] = Computer
mPlayers[3] = Computer
mPlayers[4] = Computer
mPlayers[5] = Computer
mPlayers[6] = Computer
mPlayers[7] = Computer
mPlayers[8] = Computer
~~~

Vtables runtime confirmadas:

~~~text
Gaia        = 0x009EC8A0
HumanOrCoop = 0x009ECAE4
Computer    = 0x009EC618
~~~

Nas 9 entries observadas, `object = controlBlock + 0x0C`, reforçando fortemente o modelo `std::shared_ptr`.

## Configuração/lobby → WorldPlayer

A cadeia confirmada é:

~~~text
SlotRecord
  +0x18 → configIndex
        ↓
ResolveSlotConfig
        ↓
ResolvedPlayerConfig (stride 0x68)
  +0x50 → worldPlayerIndex
  +0x54 → humanity
        ↓
World.mPlayers[worldPlayerIndex]
~~~

Exemplos confirmados em runtime:

~~~text
SlotRecord 0x00B01AF8
  configIndex      = 0
  worldPlayerIndex = 1
  humanity         = 2
  → WorldPlayerHumanOrCoop

SlotRecord 0x00B01B20
  configIndex      = 1
  worldPlayerIndex = 2
  humanity         = 4
  → WorldPlayerComputer
~~~

Nos configs seguintes da amostra:

~~~text
configIndex 2..7
worldPlayerIndex = -1
humanity = 1
~~~

## Campo player+0x08

Valores repetidamente observados:

~~~text
Gaia        = 2
HumanOrCoop = 1
Computer    = 3
~~~

Isso confirma que o campo **não é a coluna “Jog.” 1–8 do lobby**. A semântica exata ainda permanece aberta; tratá-lo apenas como discriminador/tipo observado.

## Ciclo de vida

O jogo pode recriar `World`, `mPlayers` e os objetos de player durante mudanças de estado. Não armazenar ponteiros de player indefinidamente.

Regra operacional:

~~~text
resolver Owner → World → mPlayers novamente
após criação/troca de partida ou mudanças relevantes de setup
~~~

Uma vtable válida, isoladamente, não garante que o `World` esteja gameplay-ready.

## Próximo passo de maior retorno

Localizar o campo responsável pela coluna **“Jog.” 1–8 / número-cor atribuída no lobby**, separando-o definitivamente de:

~~~text
configIndex
worldPlayerIndex
player+0x08
mLocalPlayerIndex
~~~

Depois disso: readiness estável e recursos `Food/Wood/Gold/Stone`.

## Documentação

- `docs/reverse-engineering.md`
- `docs/functions.md`
- `docs/structures.md`
- `docs/roadmap.md`
