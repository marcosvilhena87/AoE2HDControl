# Roadmap

Última atualização: 2026-10-03.

## Marco 1 — cadeia estável até World

Status: ✅ concluído para o build analisado.

Obtido:

```text
RTTI: World : BaseWorld
vtable: 0x009D329C
tamanho observado: 0x2DC bytes
FUN_006889B0 instala a vtable de World
DAT_00AF2C58 → Owner*
Owner + 0xC8 → World*
```

RVA operacional:

```text
Owner global RVA = 0x006F2C58
World vtable RVA = 0x005D329C
```

Validação runtime:

```text
[moduleBase+0x6F2C58] → Owner*
[Owner+0xC8]          → World*
[World]               → moduleBase+0x5D329C
```

Meta conceitual atingida:

```cpp
World* GetWorld();
```

Ainda falta tornar a resolução resiliente a outros builds/patches por assinatura/padrão, em vez de depender apenas de RVA.

---

## Marco 2 — jogador local

Status: 🟢 avançado, ainda não fechado semanticamente.

Confirmado:

```text
World + 0x174 → mLocalPlayerIndex
World + 0x184 → mPlayers.begin
World + 0x188 → mPlayers.end
stride = 8 bytes
```

Runtime:

```text
1 humano + 1 IA + Gaia → 3 entries
```

Cada entry possui dois ponteiros e o padrão observado é compatível com `std::shared_ptr<T>`.

RTTI da vtable primária dos objetos apontados:

```text
0x009DC8A0 → WorldPlayerGaia
```

Surpresa importante: as três entradas observadas usam essa mesma vtable primária. Portanto a distinção Gaia/humano/IA ainda está em aberto.

### Próxima tarefa concreta — maior retorno

Descobrir **onde o papel de cada jogador é discriminado**.

Candidatos:

1. campo `player+0x08` (valores observados 2/1/3);
2. outro subobjeto/vtable dentro de cada player;
3. relação externa/controller específica;
4. RTTI/vtables de `WorldPlayerHumanOrCoop` e `WorldPlayerComputer`.

Meta:

```cpp
auto localIndex = world->mLocalPlayerIndex;
auto localPlayer = ResolveLocalPlayer(world, localIndex);
```

---

## Marco 3 — recursos do jogador

Status: próximo grande marco funcional.

Prioridade:

```text
Food
Wood
Gold
Stone
Population
```

Uso principal: validação semântica de `LocalPlayer*`.

Estratégia:

```text
player candidate
      ↓
campos/containers candidatos
      ↓
comparar com valores visíveis na UI
      ↓
alterar recurso no jogo por meios normais
      ↓
observar qual campo acompanha a mudança
```

Primeiro objetivo: leitura somente.

---

## Marco 4 — objetos/unidades

Mapear:

- object id;
- owner;
- type;
- position;
- HP;
- coleção de objetos do jogador/world.

Meta conceitual:

```cpp
GetObjectById()
GetObjectsByType()
GetObjectsByTypes()
GetOwningPlayer()
GetTownCenters()
```

---

## Marco 5 — comandos

Investigar:

```text
RGE_Command
TRIBE_Command
```

Objetivos futuros:

- move;
- attack;
- build;
- gather;
- train;
- research;
- target object.

Somente após leitura de estado e identificação de jogadores/objetos estarem sólidas.

---

## Marco 6 — robustez por versão

Depois que os primeiros acessos funcionais existirem:

- substituir endereços absolutos por RVA;
- preferir signature scanning para raízes/funções críticas;
- validar RTTI/vtable antes de usar um ponteiro;
- adicionar version/hash gate;
- falhar de forma segura em builds desconhecidos.

---

## Ordem de investigação recomendada

```text
1. localizar discriminador Gaia / humano / IA
2. resolver LocalPlayer* de forma confiável
3. mapear Food/Wood/Gold/Stone
4. confirmar Player layout
5. mapear objetos/unidades
6. mapear comandos
7. transformar RVAs críticos em signatures
8. implementar HDAdapter
```

A cadeia até `World*` deixou de ser o gargalo. O foco agora deve permanecer em `mPlayers` e na resolução do jogador local.
