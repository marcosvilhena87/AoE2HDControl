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

A classe central **`World` já foi identificada por RTTI**.

Principais descobertas:

- executável alvo: `AoK HD.exe 5.8.INT`, PE32/x86;
- ImageBase: `0x00400000`;
- RTTI confirma `World : BaseWorld`;
- vtable de `World`: `0x009D329C`;
- 37 slots observados na vtable;
- `FUN_00697800` é o slot virtual 33 (`+0x84`);
- `FUN_00697800` chama `FUN_00735A00` em `0x006979F7`;
- ambas operam sobre o mesmo `World*`;
- `World + 0x174` está fortemente associado a `mLocalPlayerIndex`;
- `World + 0x184/+0x188` delimitam uma coleção associada a jogadores;
- os elementos dessa coleção têm stride de 8 bytes;
- uma instância de `World` é alocada com `0x2DC` bytes;
- `FUN_006889B0` é forte candidato ao construtor/inicializador de `World`;
- o ponteiro ativo é armazenado em `owner + 0xC8`;
- a identidade/origem desse `owner` é o próximo alvo prioritário.

Modelo atual:

```text
UnknownOwner
    │
    └── +0xC8 ─────► World (0x2DC bytes)
                       │
                       ├── vtable 0x009D329C
                       │      └── slot 33 → FUN_00697800
                       │                       └── FUN_00735A00
                       ├── +0x174  mLocalPlayerIndex
                       └── +0x184/+0x188  coleção de PlayerEntry (stride 8)
```

Veja:

- [docs/reverse-engineering.md](docs/reverse-engineering.md) — consolidação técnica;
- [docs/functions.md](docs/functions.md) — funções mapeadas;
- [docs/structures.md](docs/structures.md) — estruturas provisórias;
- [docs/roadmap.md](docs/roadmap.md) — ordem de investigação.

## Próximo passo de maior retorno

Na função que contém a criação/substituição de `World` e a região `0x0062E65A`, identificar **a primeira atribuição que define `local_20` / `owner`**.

Queremos fechar a cadeia:

```text
global/root ?
      ↓
    owner
      ↓ +0xC8
    World*
```

Isso pode fornecer a primeira implementação robusta de:

```cpp
World* GetWorld();
```

## Regra de documentação

Toda descoberta é classificada como:

- ✅ **Confirmado** — provado pelo assembly, RTTI ou evidências independentes;
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

As pastas de código serão adicionadas quando a cadeia de acesso a `World` e as estruturas principais estiverem suficientemente validadas.
