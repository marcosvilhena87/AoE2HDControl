# AoE2HDControl

Projeto de engenharia reversa e reconstrução de uma API de controle para **Age of Empires II HD (AoK HD.exe 5.8.INT, x86)**.

## Estado atual

A cadeia central já foi validada com Ghidra, RTTI e x32dbg.

Principais descobertas:

- ImageBase estático: 0x00400000;
- RTTI confirma World : BaseWorld;
- vtable estática de World: 0x009D329C;
- FUN_00697800 é o slot virtual 33 (+0x84);
- FUN_006889B0 instala a vtable de World;
- tamanho alocado observado de World: 0x2DC bytes;
- DAT_00AF2C58 fornece o Owner* usado na cadeia validada;
- Owner + 0xC8 → World*;
- World + 0x174 → mLocalPlayerIndex;
- World + 0x184/+0x188/+0x18C exibem layout compatível com std::vector x86: begin/end/end-of-storage;
- mPlayers.size() = (end - begin) / 8;
- cada entrada de mPlayers tem 8 bytes e é fortemente compatível com std::shared_ptr<WorldPlayer>;
- RTTI confirma as três classes concretas observadas: WorldPlayerGaia, WorldPlayerHumanOrCoop e WorldPlayerComputer;
- RTTI também confirma WorldPlayer e WorldPlayerBase;
- RTTI de _Ref_count_obj<...> para Gaia/Human/Computer reforça a interpretação de std::shared_ptr.

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

Exemplo validado em uma partida totalmente carregada:

~~~text
moduleBase              = 0x00D90000
[moduleBase+0x6F2C58]   = 0x04C7A750
[Owner+0xC8]            = 0x18431CD8
[World]                 = 0x0136329C
WORD[World+0x174]       = 1
mPlayers.begin          = 0x105010E0
mPlayers.end            = 0x105010F8
mPlayers.end_of_storage = 0x105010F8
size                    = 3
capacity                = 3
~~~

Entradas observadas:

~~~text
mPlayers[0].object = 0x18AA701C → WorldPlayerGaia
mPlayers[1].object = 0x146EF234 → WorldPlayerHumanOrCoop
mPlayers[2].object = 0x18B1F09C → WorldPlayerComputer
~~~

## Ciclo de vida

Ao criar uma nova partida, o Owner* permaneceu estável na amostra, mas Owner+0xC8 passou a apontar para uma nova instância de World:

~~~text
Owner*       = 0x04C7A750
novo World*  = 0x102F4108
[novo World] = 0x0136329C
~~~

Nesse estágio inicial, World+0x174 ainda não refletia o slot local esperado. Portanto, vtable válida não significa necessariamente World pronto para leitura de gameplay.

## Próximo passo de maior retorno

Repetir o teste com o humano em Player 2 depois de o mapa estar completamente carregado e confirmar:

~~~text
World+0x174
mPlayers[localIndex].object
player+0x04
player+0x08
RTTI/vtable da entry local
~~~

Se isso fechar, GetLocalPlayer() fica validado em runtime. Depois: Food/Wood/Gold/Stone.

## Documentação

- docs/reverse-engineering.md
- docs/functions.md
- docs/structures.md
- docs/roadmap.md
