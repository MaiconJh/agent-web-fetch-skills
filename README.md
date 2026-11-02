# Agent Web-Fetch Skills

Governança centralizada e arquitetura de progressive disclosure para análise de repositórios GitHub via requisições web nativas de agentes.

## Visão Geral

Este repositório atua como o **Remote Hub** autoritativo para skills de agentes de IA (como Better DeepSeek e Claude). Em vez de carregar regras extensas ou baixar arquivos compactados de repositórios inteiros, os agentes consultam as diretrizes remotas deste hub e operam via endpoints cirúrgicos.

## Arquitetura de 3 Camadas

1. **Camada 1 — Estrutura:** Ingestão completa da árvore de diretórios em 1 única requisição via `git/trees?recursive=1`.
2. **Camada 2 — Conteúdo:** Leitura direta de arquivos específicos via `raw.githubusercontent.com` (sem limites de API e sem decodificação Base64).
3. **Camada 3 — Busca (Fallback):** Resolução de símbolos via Sourcegraph streaming API e searchcode.com quando a árvore não identifica o caminho.

## Estrutura do Repositório


```

agent-web-fetch-skills/
├── SKILL.md                    # Skill principal (Bootstrapper)
├── evals/
│   └── evals.json              # Casos de validação e gatilhos de execução
├── references/
│   ├── api-limits.md           # Quotas, formatos e limitações de cada API
│   ├── fallback-strategies.md  # Fluxos de retry, jitter e circuit breakers
│   └── error-taxonomy.md       # Tabela de diagnóstico de erros HTTP e recuperação
└── README.md

```

## Diretrizes de Uso

- **Sem download de arquivos compactados:** Proibido o uso de ferramentas que baixem repositórios inteiros.
- **Circuit Breaker:** Bloqueio temporário de endpoints após limites sucessivos de falha para evitar queima de tokens.
- **Branch Fallback:** Resolução automática em cascata: `main` -> `master` -> `default_branch`.

```
