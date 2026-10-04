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

O binário apresenta fortes evidências de C++ MSVC com RTTI, herança, métodos `__thiscall` e vtables.

## Convenção x86 importante

Em métodos de instância MSVC/x86:

```text
ECX = this
```

Esse padrão foi observado nas funções centrais investigadas.

---

## Classe World — identificação confirmada

A análise de RTTI identifica a classe:

```text
World
  ↓ herda de
BaseWorld
```

A vtable de `World` começa em:

```text
0x009D329C
```

Foram observados 37 ponteiros de função nessa tabela.

Estado:

- classe `World`: ✅ confirmado por RTTI;
- herança `World : BaseWorld`: ✅ confirmada por RTTI;
- início da vtable de `World`: ✅ `0x009D329C`;
- total observado: ✅ 37 slots.

Isso substitui a antiga hipótese `UnknownWorldLike`.

---

## FUN_00697800 — método virtual de World

Endereço:

```text
0x00697800
```

`FUN_00697800` aparece na vtable de `World` no:

```text
índice 33
offset da vtable = 33 * 4 = 0x84
```

Logo:

```text
World::vtable + 0x84 → FUN_00697800
```

Estado: ✅ método virtual de `World`.

Dentro de `FUN_00697800` há chamada para:

```text
0x006979F7 → FUN_00735A00
```

A análise do fluxo indica que `FUN_00697800` e `FUN_00735A00` operam sobre o mesmo objeto `World`.

---

## FUN_00735A00 — método relacionado a estado/jogadores

Endereço:

```text
0x00735A00
```

Recebe `this` em `ECX`, compatível com método C++.

Campos relevantes:

```text
World + 0x174
World + 0x184
World + 0x188
```

### World + 0x174

A semântica observada está associada a:

```text
mLocalPlayerIndex.raw()
```

Modelo atual:

```cpp
// World + 0x174
PlayerIndex mLocalPlayerIndex;
```

Estado: 🟢 muito provável.

### World + 0x184 / +0x188

Os dois campos delimitam uma coleção contínua:

```cpp
begin = *(void**)(world + 0x184);
end   = *(void**)(world + 0x188);
count = (end - begin) / 8;
```

Stride observado:

```text
8 bytes
```

Interpretação atual:

```cpp
// World + 0x184
PlayerEntry* mPlayersBegin;

// World + 0x188
PlayerEntry* mPlayersEnd;
```

Estado:

- delimitadores de coleção: ✅ confirmado;
- coleção relacionada a jogadores / `mPlayers`: 🟢 muito provável;
- formato exato de cada elemento de 8 bytes: 🟡 ainda aberto.

### World + 0x18C

Ainda não validado.

Se a estrutura for semelhante a um `std::vector` MSVC clássico, `+0x18C` pode representar `capacity/end-of-storage`, mas isso permanece hipótese.

Estado: 🟡.

---

## Tamanho e construção de World

Foi localizado um caminho de criação de `World` na função que contém a região `0x0062E65A`.

Fluxo observado:

```text
operator_new(0x2DC)
        ↓
memset(...)
        ↓
FUN_006889B0
        ↓
World construído
```

Portanto, o tamanho alocado observado para a instância é:

```text
sizeof(World) observado = 0x2DC bytes = 732 bytes
```

`FUN_006889B0` é o forte candidato a construtor/inicializador principal de `World`.

Estado:

- alocação de `0x2DC`: ✅ confirmada;
- `FUN_006889B0` recebe o bloco recém-alocado para inicialização: ✅ confirmado;
- nome semântico `World::World`: 🟢 muito provável, pendente de caracterização completa do prólogo/RTTI/vtable write.

---

## Owner de World — campo +0xC8

Na rotina de criação/substituição foi observado um objeto aqui chamado provisoriamente de `owner` / `local_20`.

O novo `World*` é armazenado em:

```text
owner + 0xC8
```

Modelo:

```cpp
struct UnknownOwner {
    // ...
    World* world; // +0xC8
};
```

Ao substituir o ponteiro anterior, o objeto antigo é destruído por chamada virtual usando o primeiro slot de sua vtable, compatível com deleting destructor/destruição polimórfica.

Estado:

- `[owner + 0xC8] = World*`: ✅ confirmado no caminho observado;
- identidade/classe do `owner`: 🟡 desconhecida;
- destruição do World antigo via slot virtual 0: 🟢 fortemente sustentada pelo fluxo.

Essa cadeia é hoje uma das pistas de maior valor para obter a instância global/ativa de `World`.

---

## Relação estrutural atual

```text
UnknownOwner
    │
    └── +0xC8 ─────► World (0x2DC bytes)
                       │
                       ├── vtable 0x009D329C
                       │      └── slot 33 / +0x84
                       │             └── FUN_00697800
                       │                    └── 0x006979F7
                       │                           └── FUN_00735A00
                       │
                       ├── +0x174  mLocalPlayerIndex
                       ├── +0x184  mPlayers begin
                       ├── +0x188  mPlayers end
                       └── +0x18C  ? capacity/end-of-storage
```

Essa é a espinha dorsal atualmente conhecida.

---

## ASLR / RVA

Os endereços do Ghidra usam:

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

Não depender de endereços absolutos na futura instrumentação.

---

## Pistas semânticas adicionais

Foram observados conceitos/nomenclaturas compatíveis com a arquitetura original:

```text
World
BaseWorld
WorldPlayer
RGE_Command
TRIBE_Command
PathingSystem
```

Também surgiram referências relacionadas a fontes como:

```text
world.cpp
gamecommand.cpp
path.cpp
move_obj.cpp
```

Estratégia útil:

```text
RTTI / strings / asserts
          ↓
       funções
          ↓
       classes
          ↓
      estruturas
          ↓
 relações entre objetos
```

---

## Pendências conhecidas

### FUN_00659750

A função já foi aberta no Ghidra e apresenta variáveis locais como `local_8`, `local_10` e `local_14`, mas ainda não existe evidência suficiente para atribuir semântica confiável.

Regra: não nomear nem conectar essa função ao modelo principal até existir evidência estrutural.

### PlayerEntry

Stride confirmado em 8 bytes, composição ainda desconhecida.

Não assumir prematuramente que seja apenas `Player*`.

---

## Próximo passo de maior retorno

Seguir a cadeia do `owner`.

Na função que contém a criação/substituição de `World` e a região `0x0062E65A`:

> identificar a primeira escrita/atribuição que define `local_20` (o `owner`).

Objetivo:

```text
origem do owner
      ↓
owner + 0xC8
      ↓
World*
```

Se `owner` vier de global, singleton ou estrutura raiz estável, isso pode produzir a primeira cadeia runtime robusta para:

```cpp
World* GetWorld();
```

Depois disso:

1. validar `World+0x174` em runtime;
2. decodificar os elementos de `World+0x184/+0x188`;
3. resolver `Player*` do jogador local;
4. usar recursos como Food/Wood/Gold/Stone como validação.

---

## Regra de documentação

Para cada descoberta registrar:

```text
Endereço
Função
Assembly relevante
Pseudocódigo
Objeto this
Offset
Significado provável
Evidência
Nível de confiança
Próxima validação
```

Classificação:

- ✅ Confirmado
- 🟢 Muito provável
- 🟡 Hipótese
- 🔴 Descartado
