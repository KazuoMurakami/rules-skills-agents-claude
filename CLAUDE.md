# REDD Admin - Claude Code Configuration

Este arquivo serve como ponto de entrada para o Claude Code entender o projeto REDD.

## O que é o REDD?

O REDD é um **sistema de gestão empresarial** (ERP) para uma empresa de brindes promocionais. É um monorepo com:

- **redd-api**: Backend em AdonisJS v5 + TypeScript
- **redd-client**: Frontend em Next.js 13 + TypeScript + MUI
- **docs**: Documentação técnica e de features

## Arquitetura

### Backend (redd-api)

```
Request → Routes → Middleware → Controller → Validator → Service → Repository → Model → PostgreSQL
```

### Frontend (redd-client)

```
Page → Provider (Context) → Components → Hooks → API Resources (Axios) → Backend
```

## Convenções Críticas

1. **Idioma do código**: Inglês (variáveis, funções, classes)
2. **Idioma das mensagens**: Português BR (erros, toasts, labels)
3. **Soft delete**: Todas as entidades usam `deletedAt`
4. **TypeScript strict**: Sempre tipar tudo, evitar `any`

## Documentação Disponível

- `docs/technical/ARCHITECTURE.md` - Arquitetura geral
- `docs/technical/BACKEND.md` - Guia do backend
- `docs/technical/FRONTEND.md` - Guia do frontend
- `docs/features/` - Documentação de features específicas

## Rules e Agents

- `.claude/rules/` - Regras de contexto por camada
- `.claude/agents/` - Agents especializados
- `.claude/skills/` - Skills para geração de código

## Como usar

1. **Para criar código backend**: Use o agent `redd-backend-generator`
2. **Para criar código frontend**: Use o agent `redd-frontend-generator`
3. **Para analisar queries**: Use o agent `postgresql-query-analyzer`
4. **Para código limpo em PT-BR**: Use o agent `codigo-limpo-pt-br`
