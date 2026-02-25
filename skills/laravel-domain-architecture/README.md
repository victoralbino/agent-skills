# Laravel Domain Architecture

> [Leia em Portugues](#portugues) | [Read in English](#english)

---

## English

Pragmatic domain-oriented architecture for Laravel, based on "Laravel Beyond CRUD" (Spatie).
Works with pure PHP and native Laravel features — no mandatory external packages, no custom autoload.

### Philosophy

**Don't fight the framework.** Everything stays inside `app/` with standard Laravel namespaces.
The only addition is a `Domain/` folder for business logic:

```
app/
├── Domain/          # Business logic grouped by concept
│   ├── Invoice/
│   ├── Customer/
│   └── Shared/
├── Http/            # Standard Laravel (Controllers, Requests, Resources)
├── Jobs/
├── Providers/
└── Console/
```

### Structure

```
skills/laravel-domain-architecture/
├── SKILL.md                                # Agent instructions
├── metadata.json                           # Skill metadata
├── README.md                               # This file
└── references/
    ├── domain-building-blocks.md           # Data Objects, Actions, Models, Enums, QueryBuilders
    └── application-layer.md                # Controllers, Requests, Resources, HTTP Queries, Jobs, API Versioning
```

### When to Use

Laravel projects larger than average (50+ models, multiple devs, multi-year lifespan).
For simple CRUD, stick with Laravel defaults.

### Key Concepts

- **Domain folders** — Flat by default, subfolders only when needed (15+ files)
- **Actions** — Classes with `execute()` method for business logic
- **Data Objects** — Typed DTOs only when justified (multiple sources, reuse across contexts)
- **Lean Models** — Only relationships, casts, and query builders
- **Enums with behavior** — Replace the state pattern in most cases
- **Standard app layer** — Controllers, Requests, Resources in standard Laravel locations
- **Pragmatic cross-domain** — Models/enums: direct import. Side effects: events. Orchestration: direct call.
- **API Versioning** — Only when breaking changes happen; don't version preemptively

### Installation

```bash
# Install this skill only
npx skills add victoralbino/agent-skills@laravel-domain-architecture

# Or install all skills from the repo
npx skills add victoralbino/agent-skills
```

---

## Portugues

Arquitetura pragmatica orientada a dominios para Laravel, baseada em "Laravel Beyond CRUD" (Spatie).
Funciona com PHP puro e recursos nativos do Laravel — sem pacotes externos, sem autoload customizado.

### Filosofia

**Nao lute contra o framework.** Tudo fica dentro de `app/` com namespaces padrao do Laravel.
A unica adicao e uma pasta `Domain/` para a logica de negocio:

```
app/
├── Domain/          # Logica de negocio agrupada por conceito
│   ├── Invoice/
│   ├── Customer/
│   └── Shared/
├── Http/            # Laravel padrao (Controllers, Requests, Resources)
├── Jobs/
├── Providers/
└── Console/
```

### Estrutura

```
skills/laravel-domain-architecture/
├── SKILL.md                                # Instrucoes para o agente
├── metadata.json                           # Metadata da skill
├── README.md                               # Este arquivo
└── references/
    ├── domain-building-blocks.md           # Data Objects, Actions, Models, Enums, QueryBuilders
    └── application-layer.md                # Controllers, Requests, Resources, HTTP Queries, Jobs, Versionamento de API
```

### Quando Usar

Projetos Laravel maiores que a media (50+ models, multiplos devs, vida util de anos).
Para CRUD simples, mantenha os padroes do Laravel.

### Conceitos Principais

- **Pastas de dominio** — Flat por padrao, subpastas so quando necessario (15+ arquivos)
- **Actions** — Classes com metodo `execute()` para logica de negocio
- **Data Objects** — DTOs tipados so quando justifica (multiplas fontes, reuso entre contextos)
- **Models enxutos** — Apenas relacionamentos, casts e query builders
- **Enums com comportamento** — Substituem o state pattern na maioria dos casos
- **App layer padrao** — Controllers, Requests, Resources nos locais padrao do Laravel
- **Cross-domain pragmatico** — Models/enums: import direto. Efeitos colaterais: events. Orquestracao: chamada direta.
- **Versionamento de API** — So quando ha breaking changes; nao versione preventivamente

### Instalacao

```bash
# Instalar apenas esta skill
npx skills add victoralbino/agent-skills@laravel-domain-architecture

# Ou instalar todas as skills do repo
npx skills add victoralbino/agent-skills
```
