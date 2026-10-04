# Roadmap

Última atualização: 2026-10-03.

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

## Marco 2 — jogador local

Status: 🟢 quase fechado.

Confirmado:

~~~text
World + 0x174 → mLocalPlayerIndex
World + 0x184 → mPlayers.begin
World + 0x188 → mPlayers.end
World + 0x18C → vector-like end_of_storage
stride = 8
~~~

Partida observada:

~~~text
mPlayers[0] → WorldPlayerGaia
mPlayers[1] → WorldPlayerHumanOrCoop
mPlayers[2] → WorldPlayerComputer
~~~

RTTI STL reforça o modelo vector<shared_ptr<WorldPlayer>>.

Na primeira partida totalmente carregada:

~~~text
mLocalPlayerIndex = 1
mPlayers[1]       = WorldPlayerHumanOrCoop
~~~

### Próxima tarefa concreta

Repetir com o humano em Player 2, depois do mapa estar completamente carregado:

~~~text
World+0x174 == novo índice local
mPlayers[index] == WorldPlayerHumanOrCoop
player+0x04 == World*
player+0x08 == ? índice/id
~~~

## Marco 3 — readiness / lifecycle

Novo requisito descoberto.

É necessário distinguir:

~~~text
World pointer válido
~~~

de:

~~~text
World completamente inicializado para gameplay
~~~

Uma nova instância observada já possuía a vtable correta enquanto mLocalPlayerIndex ainda não refletia o slot esperado.

Meta:

~~~cpp
bool IsWorldReady(const World*);
~~~

## Marco 4 — recursos do jogador

Depois de fechar GetLocalPlayer():

~~~text
Food
Wood
Gold
Stone
Population
~~~

Primeiro objetivo: leitura somente.

## Marco 5 — objetos/unidades

Mapear object id, owner, type, position, HP e coleções.

## Marco 6 — comandos

Investigar RGE_Command e TRIBE_Command somente após leitura de estado/jogadores/objetos estar sólida.

## Marco 7 — robustez por versão

- signatures para raízes/funções críticas;
- version/hash gate;
- validação RTTI/vtable;
- falha segura em build desconhecido.

## Ordem atual recomendada

~~~text
1. concluir teste Player 2 após carregamento completo
2. fechar GetLocalPlayer()
3. mapear readiness de World
4. mapear Food/Wood/Gold/Stone
5. mapear objetos/unidades
6. mapear comandos
7. trocar RVAs críticos por signatures
8. implementar HDAdapter
~~~
