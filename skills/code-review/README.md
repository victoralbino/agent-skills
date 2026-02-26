# Code Review

> [Read in English](#english) | [Leia em Português](#português)

---

## English

Code review skill that analyzes PRs or local changes and gives actionable, prioritized recommendations.

### How it works

1. **Gather context** — reads the PR diff or local changes, then reads all affected files
2. **Analyze** — reviews architecture, code quality, security, performance, and test coverage
3. **Categorize** — organizes findings into Critical, Important, and Minor
4. **Decide** — uses `AskUserQuestion` so you choose which fixes to apply
5. **Apply** — implements approved changes, runs tests, and formats the code

### Severity levels

| Severity | Meaning |
|----------|---------|
| **Critical** | Must fix — bugs, security vulnerabilities, data loss risk |
| **Important** | Should fix — architecture violations, missing validation, logic gaps |
| **Minor** | Nice to fix — code style, duplication, missing tests for edge cases |

### Structure

```
skills/code-review/
├── SKILL.md        # Agent instructions
├── metadata.json   # Skill metadata
└── README.md       # This file
```

### Installation

```bash
# Install this skill only
npx skills add victoralbino/agent-skills@code-review

# Or install all skills from the repo
npx skills add victoralbino/agent-skills
```

### Usage

```
/code-review          # Code Reviews open PR or local uncommitted changes
/code-review 42       # Code Reviews PR #42
```

---

## Português

Skill de code review que analisa PRs ou mudanças locais e entrega recomendações priorizadas e acionáveis.

### Como funciona

1. **Coleta contexto** — lê o diff do PR ou as mudanças locais, depois lê todos os arquivos afetados
2. **Analisa** — revisa arquitetura, qualidade de código, segurança, performance e cobertura de testes
3. **Categoriza** — organiza os achados em Critical, Important e Minor
4. **Decide** — usa `AskUserQuestion` para você escolher quais correções aplicar
5. **Aplica** — implementa as mudanças aprovadas, roda os testes e formata o código

### Níveis de severidade

| Severidade | Significado |
|------------|-------------|
| **Critical** | Deve corrigir — bugs, vulnerabilidades de segurança, risco de perda de dados |
| **Important** | Deveria corrigir — violações de arquitetura, validações faltando, gaps de lógica |
| **Minor** | Legal corrigir — estilo de código, duplicação, testes faltando para edge cases |

### Estrutura

```
skills/code-review/
├── SKILL.md        # Instruções para o agente
├── metadata.json   # Metadata da skill
└── README.md       # Este arquivo
```

### Instalação

```bash
# Instalar apenas esta skill
npx skills add victoralbino/agent-skills@code-review

# Ou instalar todas as skills do repo
npx skills add victoralbino/agent-skills
```

### Uso

```
/code-review          # Revisa o PR aberto ou mudanças locais não commitadas
/code-review 42       # Revisa o PR #42
```
