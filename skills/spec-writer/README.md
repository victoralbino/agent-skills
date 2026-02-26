# Spec Writer

> [Read in English](#english) | [Leia em Portugues](#portugues)

---

## English

Activity planning skill that conducts a deep technical interview and writes a complete spec file.

### How it works

1. **Input** — provide a file path or a direct description of the activity
2. **Interview** — the agent explores your codebase and asks deep, non-obvious questions about flows, edge cases, data model, architecture, security, and tests
3. **Spec** — writes a complete, unambiguous specification file ready to implement

### What makes it different

- Explores the codebase before asking — never asks what it can already infer
- Always includes a recommendation for each option
- Keeps asking until zero ambiguities remain
- Adapts the spec format to the activity — only includes relevant sections
- Matches the user's language

### Structure

```
skills/spec-writer/
├── SKILL.md        # Agent instructions
├── metadata.json   # Skill metadata
└── README.md       # This file
```

### Installation

```bash
# Install this skill only
npx skills add victoralbino/agent-skills@spec-writer

# Or install all skills from the repo
npx skills add victoralbino/agent-skills
```

### Usage

```
/spec-writer docs/plan/02-password-reset.md
/spec-writer push notification system
```

---

## Portugues

Skill de planejamento de atividades que conduz uma entrevista tecnica profunda e escreve uma spec completa.

### Como funciona

1. **Input** — forneca um caminho de arquivo ou uma descricao direta da atividade
2. **Entrevista** — o agente explora o codebase e faz perguntas profundas e nao-obvias sobre fluxos, edge cases, modelo de dados, arquitetura, seguranca e testes
3. **Spec** — escreve uma especificacao completa e sem ambiguidades, pronta para implementar

### O que a diferencia

- Explora o codebase antes de perguntar — nunca pergunta o que ja consegue inferir
- Sempre inclui uma recomendacao para cada opcao
- Continua perguntando ate que nao reste nenhuma ambiguidade
- Adapta o formato da spec a atividade — inclui apenas as secoes relevantes
- Usa o idioma do usuario

### Estrutura

```
skills/spec-writer/
├── SKILL.md        # Instrucoes para o agente
├── metadata.json   # Metadata da skill
└── README.md       # Este arquivo
```

### Instalacao

```bash
# Instalar apenas esta skill
npx skills add victoralbino/agent-skills@spec-writer

# Ou instalar todas as skills do repo
npx skills add victoralbino/agent-skills
```

### Uso

```
/spec-writer docs/plan/02-password-reset.md
/spec-writer sistema de notificacoes push
```
