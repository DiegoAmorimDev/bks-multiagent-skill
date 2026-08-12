# Changelog

Formato baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/).
Versionamento [SemVer](https://semver.org/lang/pt-BR/).

## [1.0.0] - 2026-08-12

Primeira versão pública. Extraída de uso real em um gateway de pagamento e generalizada.

### Adicionado

- **`skill/SKILL.md`** — orquestração com regra zero (sequencial por padrão), classificação de
  criticidade em três níveis, teste de independência, reserva de recursos numerados, portões de
  governança e orçamento de tokens
- **`skill/references/manifesto-projeto.md`** — template de manifesto por projeto, com método para
  descobrir zonas de contenção pelo histórico do git
- **`skill/references/checklist-revisao-critica.md`** — checklist do portão de revisão: núcleo
  universal + perfis para financeiro, dado regulado, cartão/PCI e verificação de assinatura
- **`skill/references/protocolo-paralelo.md`** — protocolo de fan-out, integração serializada,
  recuperação de conflito e modo degradado
- **`skill/templates/agents/`** — quatro perfis (`builder`, `reviewer`, `planner`, `scribe`) com
  modelo sugerido e proibições explícitas
- **`docs/contextos-de-uso.md`** — oito contextos de uso, árvore de decisão e antipadrões
- **`docs/instalacao.md`** — instalação, preenchimento do manifesto e checklist de verificação
- **`docs/exemplos/`** — dois casos completos: gateway de pagamento (Go, hexagonal) e SaaS
  multi-tenant (Node/Prisma)

### Decisões de design

- **A skill não tem plano próprio.** Ela governa *como* delegar; o *que* construir permanece no
  processo do projeto. Isso é o que permite conviver com metodologias de planejamento existentes,
  em vez de competir com elas.
- **Manifesto obrigatório.** A alternativa — inferir zonas de contenção automaticamente — foi
  descartada: erro de inferência aqui custa corrupção de estado, não apenas retrabalho.
- **Segregação de funções é regra, não sugestão.** Em domínio crítico é requisito de conformidade.

[1.0.0]: https://github.com/DiegoAmorimDev/bks-multiagent-skill/releases/tag/v1.0.0
