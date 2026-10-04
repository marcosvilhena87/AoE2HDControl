# Engenharia reversa — estado consolidado

Última atualização: 2026-10-03.

## Alvo

- Executável: `AoK HD.exe`
- Versão: `5.8.INT`
- Plataforma: Windows x86 / PE32
- ImageBase: `0x00400000`
- Arquitetura: i386
- Tamanho registrado: 7.521.280 bytes
- Timestamp PE registrado: 22/08/2018
- SHA-256 registrado: `CBD10D81B93601FFB26773D250B7478969DA92D70792A40AE423294202E14650`
- PDB referenciado: `AoK HD.pdb`

O binário apresenta fortes evidências de código C++ com métodos de instância, tabelas de ponteiros e RTTI/strings úteis para reconstrução semântica.

## Convenção importante

No MSVC/x86, métodos de instância normalmente usam `__thiscall`, portanto:

```text
ECX = this
```

Esse padrão já foi observado nas funções centrais investigadas.

## FUN_00735A00

Endereço:

```text
0x00735A00
```

A função salva o objeto recebido em `ECX` em uma variável local, o que sustenta a interpretação de método C++.

Campos relevantes acessados:

```text
this + 0x174
this + 0x184
this + 0x188
```

### +0x174

Esse campo aparece associado semanticamente à string/expressão:

```text
mLocalPlayerIndex.raw()
```

Hipótese atual:

```cpp
// +0x174
PlayerIndex mLocalPlayerIndex;
```

Estado: 🟢 muito provável.

### +0x184 e +0x188

Os dois valores são tratados como delimitadores de uma coleção contínua:

```cpp
begin = *(void**)(this + 0x184);
end   = *(void**)(this + 0x188);
count = (end - begin) / 8;
```

O stride observado é de 8 bytes.

Interpretação provisória:

```cpp
// +0x184
PlayerEntry* mPlayersBegin;

// +0x188
PlayerEntry* mPlayersEnd;
```

Estado:

- delimitadores de coleção: ✅ confirmado;
- associação com `mPlayers`: 🟢 muito provável;
- tipo exato de `PlayerEntry`: 🟡 hipótese.

## FUN_00697800

Endereço:

```text
0x00697800
```

Também salva `ECX` como objeto local, compatível com método C++.

Há uma chamada para `FUN_00735A00` na região:

```text
0x006979F7
```

Fluxo:

```text
FUN_00697800
    ↓
0x006979F7
    ↓
FUN_00735A00
```

O principal ponto ainda não resolvido é determinar exatamente qual valor está em `ECX` no call site.

## XREFs e tabela de funções

`FUN_00697800` é referenciada em:

```text
0x009D3320
```

Esse endereço pertence a uma região com vários ponteiros consecutivos para funções:

```text
0x009D32F0
...
0x009D3320 → FUN_00697800
...
0x009D332C
```

A região é compatível com uma vtable, mas isso ainda não está confirmado.

Alternativas possíveis:

- vtable;
- dispatch table;
- callback table;
- interface table;
- array estático de handlers.

Estado: 🟡 hipótese.

## Hipótese de estrutura atual

```cpp
struct UnknownWorldLike
{
    void* vtable; // ainda não confirmado

    // ...

    // +0x174
    PlayerIndex mLocalPlayerIndex;

    // ...

    // +0x184
    PlayerEntry* mPlayersBegin;

    // +0x188
    PlayerEntry* mPlayersEnd;

    // +0x18C possivelmente capacity/end-of-storage
};
```

O campo `+0x18C` ainda precisa ser inspecionado. Caso a coleção seja um `std::vector` clássico de MSVC, o trio esperado seria begin/end/capacity, mas isso é somente hipótese.

## ASLR / RVA

Os endereços do Ghidra são baseados em:

```text
ImageBase = 0x00400000
```

Exemplo:

```text
FUN_00735A00 RVA
= 0x00735A00 - 0x00400000
= 0x00335A00
```

Em runtime:

```cpp
runtimeAddress = moduleBase + RVA;
```

Não devemos depender de endereços absolutos quando começarmos a instrumentação runtime.

## Evidências de arquitetura interna

Foram observados nomes e conceitos compatíveis com a arquitetura original do jogo, incluindo referências relacionadas a:

```text
World
BaseWorld
WorldPlayer
RGE_Command
TRIBE_Command
PathingSystem
```

Também apareceram nomes relacionados a arquivos-fonte como:

```text
world.cpp
gamecommand.cpp
path.cpp
move_obj.cpp
```

Essas pistas reforçam a estratégia:

```text
strings
  ↓
asserts/debug strings
  ↓
funções
  ↓
classes
  ↓
estruturas
```

## Pergunta central atual

> Qual é exatamente a classe representada pelo `this` usado em `FUN_00735A00` e `FUN_00697800`?

Responder isso provavelmente destrava o mapeamento de World/GameState, Player e recursos.

## Próximas validações

1. Mapear completamente `0x009D3280–0x009D3330`.
2. Determinar o início real da possível tabela de funções.
3. Procurar XREFs para o início da tabela.
4. Encontrar código que grave o endereço da tabela em `[this]` para localizar o provável construtor.
5. Reconstruir o contexto de `0x006979F7` e determinar `ECX`.
6. Buscar todos os usos de `+0x174`, `+0x184` e `+0x188`.
7. Inspecionar `+0x18C`.
8. Descobrir a composição exata dos elementos de 8 bytes de `mPlayers`.

## Regra operacional

Para cada descoberta registrar:

```text
Endereço
Função
Assembly relevante
Pseudocódigo
Offset
Significado provável
Evidência
Nível de confiança
Próxima validação
```
