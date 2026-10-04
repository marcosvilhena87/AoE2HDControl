# Roadmap

Última atualização: 2026-10-04.

## Marco 1 — cadeia estável até World

Status: ✅ concluído para o build analisado.

~~~text
DAT_00AF2C58 → Owner*
Owner + 0xC8 → World*
World vtable = 0x009D329C
World size observado = 0x2DC
~~~

RVA operacional:

~~~text
Owner global RVA = 0x006F2C58
World vtable RVA = 0x005D329C
~~~

## Marco 2 — World.mPlayers e tipos concretos

Status: ✅ concluído estruturalmente.

Confirmado:

~~~text
World + 0x184 → mPlayers.begin
World + 0x188 → mPlayers.end
World + 0x18C → mPlayers.end_of_storage
stride = 8
~~~

Partida com 8 jogadores:

~~~text
size = 9

mPlayers[0] → WorldPlayerGaia
mPlayers[1] → WorldPlayerHumanOrCoop
mPlayers[2] → WorldPlayerComputer
...
mPlayers[8] → WorldPlayerComputer
~~~

Todos os tipos foram validados por vtable runtime.

O padrão `object = controlBlock + 0x0C` e o RTTI `_Ref_count_obj` reforçam fortemente `vector<shared_ptr<WorldPlayer>>`.

## Marco 3 — SlotRecord → WorldPlayer

Status: ✅ cadeia principal fechada.

Confirmado:

~~~text
SlotRecord stride = 0x28
SlotRecord+0x18 = configIndex

ResolveSlotConfig:
  base + 0xB0 + configIndex * 0x68

ResolvedPlayerConfig+0x50 = worldPlayerIndex
ResolvedPlayerConfig+0x54 = humanity

FUN_00598700:
  SlotRecord → worldPlayerIndex → World.mPlayers[index]
~~~

Validação runtime:

~~~text
config 0 → worldPlayerIndex 1 → humanity 2 → HumanOrCoop
config 1 → worldPlayerIndex 2 → humanity 4 → Computer
configs 2..7 → worldPlayerIndex -1 → humanity 1 na amostra
~~~

`humanity=3` continua não identificado.

## Marco 4 — localizar “Jog.” 1–8 / número-cor

Status: 🔴 próximo alvo de maior retorno.

Já sabemos que a coluna visual **“Jog.”** não é:

~~~text
player+0x08
SlotRecord.configIndex
ResolvedPlayerConfig.worldPlayerIndex
World.mLocalPlayerIndex
~~~

Objetivo:

~~~text
identificar o campo que representa a atribuição visual 1..8
e mapear leitura/escrita dessa configuração
~~~

Método recomendado:

1. manter a mesma linha/pessoa no lobby;
2. alterar apenas “Jog.” entre dois valores;
3. comparar estruturas de configuração relevantes;
4. usar breakpoint de escrita quando um candidato aparecer;
5. confirmar que a alteração acompanha 1↔2↔...↔8 sem mudar `configIndex` ou classe do player.

## Marco 5 — readiness / lifecycle

Status: 🟡 requisito conhecido.

É necessário distinguir:

~~~text
World pointer válido
~~~

de:

~~~text
World completamente inicializado para gameplay
~~~

Além disso, `World`, `mPlayers` e `WorldPlayer*` podem ser recriados entre estados.

Meta:

~~~cpp
bool IsWorldReady(const World*);
~~~

e política de não cachear ponteiros derivados por longo prazo.

## Marco 6 — jogador local

Status: 🟢 estruturalmente forte.

Confirmado:

~~~text
World + 0x174 → mLocalPlayerIndex
~~~

Em estados gameplay-ready, o índice resolve para `WorldPlayerHumanOrCoop`.

Meta:

~~~cpp
WorldPlayer* GetLocalPlayer(World*);
~~~

Separar explicitamente esse índice da coluna visual “Jog.” do lobby.

## Marco 7 — recursos do jogador

Depois de fechar o campo “Jog.” e readiness:

~~~text
Food
Wood
Gold
Stone
Population
~~~

Primeiro objetivo: leitura somente.

## Marco 8 — objetos/unidades

Mapear:

~~~text
object id
owner
type
position
HP
coleções
~~~

## Marco 9 — comandos

Investigar `RGE_Command` e `TRIBE_Command` somente após a leitura de estado/jogadores/objetos estar sólida.

## Marco 10 — robustez por versão

- signatures para raízes/funções críticas;
- version/hash gate;
- validação RTTI/vtable;
- falha segura em build desconhecido.

## Referência conceitual — AoE2Control / Definitive Edition

Status: 🟢 incorporada como guia semântico.

O AoE2Control é usado apenas para responder:

~~~text
quais conceitos uma API madura de controle precisa expor?
~~~

Exemplos de alvos inspirados por essa referência:

~~~text
player id / color
resources / population
object id / owner / type / position / HP
map tiles / terrain / elevation / passability
pathfinding / placement
commands
IPC / external agent
~~~

Nenhum offset, stride, vtable ou layout do AoE2:DE deve ser usado como evidência para o HD.

Detalhes: `docs/aoe2control-reference.md`.

## Ordem atual recomendada

~~~text
1. localizar campo “Jog.” 1–8 / número-cor
2. consolidar IsWorldReady()
3. fechar GetLocalPlayer() como API
4. mapear Food/Wood/Gold/Stone
5. mapear objetos/unidades
6. mapear comandos
7. trocar RVAs críticos por signatures
8. implementar HDAdapter
~~~
