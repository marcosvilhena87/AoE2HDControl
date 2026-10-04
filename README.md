# AoE2HDControl

Projeto de engenharia reversa e reconstrução de uma API de controle para **Age of Empires II HD (AoK HD.exe 5.8.INT, x86)**.

## Objetivo

Mapear progressivamente as estruturas internas do jogo para chegar a uma interface de alto nível semelhante à ideia original do AoE2Control:

```text
World
  ↓
Players
  ↓
Local Player
  ↓
Resources / Objects / Units
  ↓
Commands
```

## Estado atual

A espinha dorsal de acesso ao estado do jogo já foi identificada e **validada dinamicamente** com Ghidra + x32dbg.

Principais descobertas:

- executável alvo: `AoK HD.exe 5.8.INT`, PE32/x86;
- ImageBase estático: `0x00400000`;
- RTTI confirma `World : BaseWorld`;
- vtable estática de `World`: `0x009D329C`;
- 37 slots observados na vtable;
- `FUN_00697800` é o slot virtual 33 (`+0x84`);
- `FUN_00697800` chama `FUN_00735A00` em `0x006979F7`;
- ambas operam sobre o mesmo `World*`;
- `FUN_006889B0` instala a vtable de `World` e é usada após `operator_new(0x2DC)`;
- tamanho alocado observado de `World`: `0x2DC` bytes;
- `DAT_00AF2C58` contém um ponteiro para o objeto owner/root usado no caminho validado;
- `owner + 0xC8 → World*`;
- `World + 0x174 → mLocalPlayerIndex`;
- `World + 0x184/+0x188` são begin/end da coleção `mPlayers`;
- `mPlayers.size() = (end - begin) / 8`;
- em partida com 1 humano + 1 IA, a coleção tinha 3 entradas: Gaia + humano + IA;
- cada entrada de `mPlayers` tem 8 bytes e o layout observado é fortemente compatível com `std::shared_ptr<T>` MSVC x86;
- o objeto apontado pelas entradas começa com vtable cujo RTTI resolve para `WorldPlayerGaia`; curiosamente, as três entradas observadas apresentaram a mesma vtable primária, então a distinção Gaia/humano/IA ainda precisa ser localizada em outro campo/objeto.

## Cadeia runtime validada

Com ASLR, usar RVA:

```text
Owner global RVA = 0x006F2C58
World vtable RVA = 0x005D329C
```

Cadeia:

```text
[moduleBase + 0x006F2C58]
        ↓ deref
      Owner*
        ↓ +0xC8 / deref
      World*
        │
        ├── +0x000 → vtable = moduleBase + 0x005D329C
        ├── +0x174 → mLocalPlayerIndex
        ├── +0x184 → mPlayers.begin
        └── +0x188 → mPlayers.end
```

Exemplo validado em runtime:

```text
moduleBase              = 0x00D90000
[moduleBase+0x6F2C58]   = 0x04C7A750   // Owner*
[Owner+0xC8]            = 0x18431CD8   // World*
[World]                 = 0x0136329C   // World vtable runtime
WORD[World+0x174]       = 1            // jogador local
mPlayers.begin          = 0x105010E0
mPlayers.end            = 0x105010F8
(end-begin)/8           = 3
```

## Documentação

- [docs/reverse-engineering.md](docs/reverse-engineering.md) — consolidação técnica;
- [docs/functions.md](docs/functions.md) — funções mapeadas;
- [docs/structures.md](docs/structures.md) — estruturas provisórias;
- [docs/roadmap.md](docs/roadmap.md) — ordem de investigação.

## Próximo passo de maior retorno

A cadeia até `World*` já está fechada. O gargalo agora é resolver **o jogador local de forma semântica e estável**:

1. entender o layout exato das entradas de 8 bytes de `mPlayers`;
2. localizar onde Gaia/humano/IA são distinguidos;
3. relacionar `mLocalPlayerIndex` à entrada correta;
4. mapear recursos do jogador local (Food/Wood/Gold/Stone) como próxima validação forte.

## Regra de documentação

Toda descoberta é classificada como:

- ✅ **Confirmado** — provado pelo assembly, RTTI ou validação runtime;
- 🟢 **Muito provável** — evidência estrutural/semântica forte;
- 🟡 **Hipótese** — plausível, mas ainda precisa de validação;
- 🔴 **Descartado** — hipótese testada e incompatível com o binário.

O projeto evita hardcodes prematuros. A abordagem preferida é:

```text
RTTI / função
      ↓
objeto
      ↓
estrutura
      ↓
relações
      ↓
validação runtime
      ↓
assinatura/padrão estável
```

## Estrutura do repositório

```text
AoE2HDControl/
├─ README.md
└─ docs/
   ├─ reverse-engineering.md
   ├─ functions.md
   ├─ structures.md
   └─ roadmap.md
```

As pastas de código serão adicionadas quando as estruturas principais estiverem suficientemente validadas para uma primeira implementação segura de `GetWorld()` e `GetLocalPlayer()`.
