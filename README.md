# AoE2HDControl

Projeto de engenharia reversa e reconstrução de uma API de controle para **Age of Empires II HD (AoK HD.exe 5.8.INT, x86)**.

## Objetivo

Mapear progressivamente as estruturas internas do jogo para chegar a uma interface de alto nível semelhante à ideia original do AoE2Control:

```text
World / GameState
    ↓
Players
    ↓
Local Player
    ↓
Resources / Objects / Units
    ↓
Commands
```

A prioridade atual é identificar com segurança a estrutura equivalente a **World/GameState**, a coleção de jogadores e o jogador local.

## Estado atual

Descobertas mais relevantes:

- executável alvo: `AoK HD.exe`, PE32/x86;
- ImageBase: `0x00400000`;
- `FUN_00735A00` recebe um objeto em `ECX`;
- `FUN_00697800` também aparenta ser método C++ e chama `FUN_00735A00`;
- call site conhecido: `0x006979F7`;
- `this + 0x174` está fortemente associado a `mLocalPlayerIndex`;
- `this + 0x184` e `this + 0x188` delimitam uma coleção;
- os elementos dessa coleção têm stride de 8 bytes;
- há uma possível tabela de funções na região `0x009D32F0–0x009D332C`;
- `0x009D3320` referencia `FUN_00697800`.

Veja a documentação detalhada em [docs/reverse-engineering.md](docs/reverse-engineering.md).

## Regra de documentação

Toda descoberta deve ser classificada como:

- ✅ **Confirmado** — provado pelo assembly ou por evidências independentes;
- 🟢 **Muito provável** — evidência estrutural/semântica forte;
- 🟡 **Hipótese** — plausível, mas ainda precisa de validação;
- 🔴 **Descartado** — hipótese testada e incompatível com o binário.

O projeto evita hardcodes prematuros. A abordagem preferida é:

```text
função → objeto → estrutura → relações → validação runtime → assinatura/padrão
```

## Estrutura do repositório

```text
AoE2HDControl/
├─ README.md
├─ docs/
│  ├─ reverse-engineering.md
│  ├─ functions.md
│  ├─ structures.md
│  └─ roadmap.md
├─ include/
└─ src/
```

As pastas de código serão preenchidas depois que as estruturas principais estiverem suficientemente validadas.
