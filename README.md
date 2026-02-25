# AI Skills

> [Read in English](#english) | [Leia em Português](#português)

---

## English

Repository to group and organize AI skills used across projects.

### Structure

```
skills/
  my-skill/
    SKILL.md          # Agent instructions (frontmatter + content)
    metadata.json     # Metadata (version, description, references)
    README.md         # Human documentation
    references/       # Supporting documentation (optional)
    rules/            # Individual rules (optional)
    scripts/          # Helper scripts (optional)
```

### Available Skills

- **[laravel-domain-architecture](skills/laravel-domain-architecture/)** — Domain-oriented architecture for Laravel based on "Laravel Beyond CRUD" (Spatie)

### Installation

```bash
# Install all skills
npx skills add victoralbino/agent-skills

# Install a specific skill
npx skills add victoralbino/agent-skills@laravel-domain-architecture

# List available skills without installing
npx skills add victoralbino/agent-skills --list
```

---

## Português

Repositório para agrupar e organizar skills de IA utilizadas em projetos.

### Estrutura

```
skills/
  minha-skill/
    SKILL.md          # Instruções para o agente (frontmatter + conteúdo)
    metadata.json     # Metadata (versão, descrição, referências)
    README.md         # Documentação para humanos
    references/       # Documentação de apoio (opcional)
    rules/            # Regras individuais (opcional)
    scripts/          # Scripts auxiliares (opcional)
```

### Skills Disponíveis

- **[laravel-domain-architecture](skills/laravel-domain-architecture/)** — Arquitetura orientada a domínios para Laravel baseada em "Laravel Beyond CRUD" (Spatie)

### Instalação

```bash
# Instalar todas as skills
npx skills add victoralbino/agent-skills

# Instalar uma skill específica
npx skills add victoralbino/agent-skills@laravel-domain-architecture

# Listar skills disponíveis sem instalar
npx skills add victoralbino/agent-skills --list
```
