# Laravel Domain Architecture

> [Leia em Português](#português) | [Read in English](#english)

---

## English

Pragmatic domain-oriented architecture for Laravel, based on "Laravel Beyond CRUD" (Spatie).
Works with pure PHP and native Laravel features — no mandatory external packages.

### Structure

```
skills/laravel-domain-architecture/
├── SKILL.md                                # Agent instructions
├── metadata.json                           # Skill metadata
├── README.md                               # This file
└── references/
    ├── domain-building-blocks.md           # Data Objects, Actions, Models, Enums, QueryBuilders
    └── application-layer.md                # Controllers, ViewModels, HTTP Queries, Jobs, API Versioning
```

### When to Use

Laravel projects larger than average (50+ models, multiple devs, multi-year lifespan).
For simple CRUD, stick with Laravel defaults.

### Key Concepts

- **Data Objects** — Typed DTOs with pure PHP 8.3+
- **Actions** — Classes with `execute()` method for business logic
- **Lean Models** — Only relationships, casts, and query builders
- **Enums with behavior** — Replace the state pattern in most cases
- **ViewModels** — Response composition (Blade and API)
- **Domain/App separation** — Code by business meaning, not by technical type
- **API Versioning** — Only at the application layer; domain stays shared

### Installation

```bash
# Install this skill only
npx skills add victoralbino/agent-skills@laravel-domain-architecture

# Or install all skills from the repo
npx skills add victoralbino/agent-skills
```

---

## Português

Arquitetura pragmática orientada a domínios para Laravel, baseada em "Laravel Beyond CRUD" (Spatie).
Funciona com PHP puro e recursos nativos do Laravel — sem pacotes externos obrigatórios.

### Estrutura

```
skills/laravel-domain-architecture/
├── SKILL.md                                # Instruções para o agente
├── metadata.json                           # Metadata da skill
├── README.md                               # Este arquivo
└── references/
    ├── domain-building-blocks.md           # Data Objects, Actions, Models, Enums, QueryBuilders
    └── application-layer.md                # Controllers, ViewModels, HTTP Queries, Jobs, Versionamento de API
```

### Quando Usar

Projetos Laravel maiores que a média (50+ models, múltiplos devs, vida útil de anos).
Para CRUD simples, mantenha os padrões do Laravel.

### Conceitos Principais

- **Data Objects** — DTOs tipados com PHP 8.3+ puro
- **Actions** — Classes com método `execute()` para lógica de negócio
- **Models enxutos** — Apenas relacionamentos, casts e query builders
- **Enums com comportamento** — Substituem o state pattern na maioria dos casos
- **ViewModels** — Composição de respostas (Blade e API)
- **Separação Domain/App** — Código por significado de negócio, não por tipo técnico
- **Versionamento de API** — Apenas na camada de aplicação; domínio é compartilhado

### Instalação

```bash
# Instalar apenas esta skill
npx skills add victoralbino/agent-skills@laravel-domain-architecture

# Ou instalar todas as skills do repo
npx skills add victoralbino/agent-skills
```
