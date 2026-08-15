# Changelog

Formato baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/).
Versionamento [SemVer](https://semver.org/lang/pt-BR/).

## [1.3.0] - 2026-08-15

Correção vinda de um diagnóstico errado que sobreviveu cinco sessões em produção da skill
(favo-pay) — o perfil `reviewer` simplesmente não registrava, e a hipótese aceita durante dias
("o harness bloqueia subagente pinado num modelo acima do da sessão") era falsa.

### Adicionado

- **`skill/SKILL.md`**, nova subseção "Confirme que o agente registrou" em *Perfis de agente* —
  frontmatter inválido faz o harness descartar a definição **em silêncio**, sem erro nem aviso.
  A subseção manda listar os tipos disponíveis e conferir o nome antes de contar com o agente,
  documenta a causa concreta já observada (`description:` sem aspas contendo `: ` em arquivo
  CRLF) e dá o procedimento de isolamento para quando o sintoma reaparecer.

### Alterado

- **`skill/templates/agents/*.md`** (os quatro perfis) — valor de `description:` agora vem
  **entre aspas**. Os templates são distribuídos em CRLF e adaptados majoritariamente no
  Windows; sem aspas, qualquer `:` que o usuário escreva ao adaptar a description derruba o
  agente inteiro sem sinal nenhum.

### Por que isso importa

O modo de falha é pior que o próprio bug: o agente some sem mensagem, e o portão de revisão
independente — o controle mais importante que esta skill impõe (segregação de funções, Portão 1)
— deixa de existir sem que ninguém perceba. Nas cinco ocorrências reais, o trabalho seguiu com
um agente genérico recebendo a persona colada à mão; funcionou, mas por diligência do operador,
não por garantia do processo. Um portão que pode desaparecer em silêncio não é um portão, e
nenhuma quantidade de rigor nas outras etapas compensa isso.

O diagnóstico errado é parte da lição: o sintoma (só o perfil em Opus falhava, os em
Sonnet/Haiku registravam) apontava com força para restrição de modelo, e a correlação era
perfeita — porque só o `reviewer` tinha `: ` na description. Correlação com amostra de quatro
arquivos não é causa. O procedimento de isolamento agora está na skill justamente para que a
próxima ocorrência custe minutos em vez de sessões.

[1.3.0]: https://github.com/DiegoAmorimDev/bks-multiagent-skill/releases/tag/v1.3.0

## [1.2.0] - 2026-08-13

Ajuste vindo de custo real observado em produção da skill (favo-pay) — o perfil `builder`
processa muitas features guiadas por uma TASK já bem especificada (spec vem do `planner`,
correção vem de TDD + verificação E2E real), um padrão de trabalho mais mecânico do que o
que justifica um modelo de raciocínio no topo da linha em todo commit.

### Alterado

- **`skill/templates/agents/builder.md`** e **`skill/SKILL.md`** (tabela de perfis) — modelo
  sugerido para `builder` passa de Opus para **Sonnet**. `reviewer` continua em Opus: é o único
  perfil onde o custo de um achado escapando (bug financeiro/segurança indo para produção)
  supera o custo do modelo — a proporcionalidade descrita no Passo 4 ("modelo proporcional à
  tarefa") já apontava nessa direção, só não tinha sido aplicada ao próprio `builder`.
- **`README.md`** — tabela de perfis sincronizada com a mesma mudança.

### Por que isso importa

A tabela original recomendava Opus para `builder` e `reviewer` igualmente, tratando os dois
como "trabalho de arquitetura" — mas só `reviewer` é un-repetível por natureza (a revisão só
acontece uma vez por mudança sensível, e um bypass ali não tem segunda chance). `builder`
roda uma vez por feature/task, muitas vezes seguindo uma instrução já detalhada pelo
`planner`, com uma rede de segurança própria (TDD + teste de ponta a ponta contra o sistema
real, exigidos no Passo 6). Não é um domínio a menos zelo — é reconhecer que a skill já tinha
os dois controles (spec detalhada + verificação real) que tornam Sonnet suficiente aqui, sem
abrir mão do padrão de qualidade.

[1.2.0]: https://github.com/DiegoAmorimDev/bks-multiagent-skill/releases/tag/v1.2.0

## [1.1.0] - 2026-08-12

Correção vinda de incidente real no primeiro uso em produção da skill (gateway de
pagamento favo-pay) — exatamente o tipo de contribuição que `CONTRIBUTING.md` pede.

### Adicionado

- **`skill/references/protocolo-paralelo.md`** — seção "Verificação isolada por
  commit — cuidado com `git stash`", documentando um incidente real de perda
  silenciosa de arquivos causado por ciclos repetidos de `git stash push
  --keep-index` / `pop` durante a organização de commits de milestone. Inclui
  alternativas seguras (verificação contra a árvore completa, `git worktree`) e
  como recuperar via `git fsck --unreachable` se acontecer mesmo assim.
- **`skill/SKILL.md`** — aviso equivalente, mais curto, no Passo 6 (Integrar em
  série), apontando para a seção detalhada.

### Por que isso importa

A skill já exigia evidência de build/teste a cada commit ("Passo 6 — Integrar em
série"), mas não dizia *como* verificar cada commit com segurança. Na ausência
dessa orientação, a técnica óbvia (stage o commit N, stash o resto para testar em
isolamento) causou perda de ~120 linhas de trabalho sem nenhum erro reportado — só
percebida porque um diff posterior voltou vazio quando não devia. A skill agora
fecha essa lacuna explicitamente.

[1.1.0]: https://github.com/DiegoAmorimDev/bks-multiagent-skill/releases/tag/v1.1.0

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
