---
name: bks-multiagent-skill
description: >
  Orquestração multi-agente com governança para sistemas de alta criticidade. Define quais tarefas
  podem rodar em paralelo sem corromper estado compartilhado, quais exigem revisão independente
  obrigatória, e como briefar subagentes gastando o mínimo de tokens. Funciona em qualquer stack —
  cada projeto declara suas zonas de contenção e áreas sensíveis em um manifesto.
  Use ao delegar implementação, revisão ou documentação a subagentes, especialmente ao rodar 2+
  tarefas ao mesmo tempo. Trigger: "paralelo", "multi-agente", "delegar", "subagente", "fan-out",
  "vários agentes", "orquestrar", "/bks-multiagent-skill".
---

# Orquestração multi-agente com governança

Paralelismo entre agentes não é otimização gratuita. Ele troca tempo por **risco de corrupção de
estado compartilhado** e por **perda de rastreabilidade** de quem decidiu o quê. Em sistema de
baixa criticidade isso é aceitável na base do "depois arruma". Em sistema que move dinheiro,
guarda dado pessoal ou responde a auditoria externa, não é.

Esta skill governa **como** delegar. Ela não substitui o processo de planejamento do projeto —
o plano continua onde já estava (specs, issues, backlog). Aqui se decide: paraleliza ou não,
quem revisa, o que cada agente pode tocar, e quanto contexto ele recebe.

## Regra zero

**Sequencial é o padrão. Paralelo é a exceção que precisa ser justificada.**

Só há ganho real quando as tarefas tocam módulos diferentes e nenhuma precisa do resultado da
outra. Em qualquer dúvida, serialize. Retrabalho por conflito custa mais — em tokens e em risco —
do que a espera teria custado.

## Passo 1 — Carregar o manifesto do projeto

Antes de qualquer delegação, leia o manifesto do projeto (`.claude/multiagente.md` ou o caminho
que o projeto adotar). Ele declara o que é específico de cada repositório:

- **Zonas de contenção** — arquivos que toda mudança toca (ponto de serialização)
- **Recursos numerados compartilhados** — migrations, ADRs, IDs sequenciais
- **Áreas sensíveis** — o que exige revisão independente obrigatória
- **Nível de criticidade** — define a rigidez dos portões
- **Perfis de agente disponíveis** — quem faz o quê, com qual modelo

Sem manifesto, **crie um antes de paralelizar**. Template em `references/manifesto-projeto.md`.
Delegar sem saber onde estão as zonas de contenção é como fazer merge sem olhar o diff.

## Passo 2 — Classificar a criticidade

O rigor dos portões escala com o que está em jogo:

| Nível | Exemplos | Portões obrigatórios |
|---|---|---|
| **Crítico** | Pagamentos, saúde, identidade, infraestrutura, dado regulado | Revisão independente + evidência de teste + registro de decisão + segregação de funções |
| **Alto** | SaaS B2B com SLA, dado pessoal, integração com terceiros | Revisão independente em áreas sensíveis + evidência de teste |
| **Padrão** | Ferramenta interna, protótipo, conteúdo | Evidência de teste |

Um repositório pode ter níveis diferentes por área: o módulo de cobrança é crítico, o de
relatórios internos é padrão. O manifesto declara isso.

## Perfis de agente

Quatro perfis cobrem a maior parte do trabalho. Templates em `templates/agents/`.

| Perfil | Modelo sugerido | Faz | Nunca faz |
|---|---|---|---|
| `builder` | Alto (Opus) | Implementação, decisão de arquitetura | Commit; aprovar o próprio trabalho |
| `reviewer` | Alto (Opus) | Revisão de segurança/conformidade — só lê | Editar código |
| `planner` | Médio (Sonnet) | Specs, ADRs, quebra de tarefa | Código de produção |
| `scribe` | Baixo (Haiku) | Sincronizar docs de trabalho já verificado | Decidir o que foi feito |

Ordem natural dentro de **uma** entrega: `planner` → `builder` → `reviewer` → `scribe`.
São dependentes por natureza — nunca paralelize entre si na mesma entrega.

## Passo 3 — Teste de independência

Antes de disparar 2+ agentes, responda. Um "não" em qualquer uma = sequencial.

1. **Módulos distintos?** Cada tarefa fica inteiramente dentro de um módulo/pacote/serviço diferente?
2. **Zona de contenção intocada?** Nenhuma precisa editar, por conta própria, arquivo que toda
   mudança toca (wiring, roteamento, migrations, docs de índice, changelog)?
3. **Sem dependência de resultado?** A tarefa B compila e roda sem saber o que A decidiu?

### Armadilhas que parecem independentes e não são

- **Duas features que criam migration.** Ambas escolhem o próximo número livre e colidem.
  Resolve-se com reserva prévia, não com sorte.
- **Duas tarefas da mesma feature** (entidade + serviço que usa a entidade). Dependência de tipo:
  paralelizar quebra a compilação nas duas.
- **Dois agentes adicionando método ao mesmo mock/fixture compartilhado** de um pacote de teste.
- **Duas mudanças que alteram o mesmo contrato de API** — a segunda invalida a primeira.

## Passo 4 — Reservar recursos compartilhados

Numeração sequencial é recurso compartilhado. Reserve **antes** de disparar e informe no briefing
de cada agente: "você usa a migration 015 e o ADR 024; o outro agente usa 016 e 025".

Se um agente não precisa de migration, diga explicitamente que não precisa — senão ele inventa uma.

> **A fonte da verdade é o repositório, não o documento de planejamento.** Specs e tickets
> envelhecem: sugerem números que já foram usados. Sempre confira o estado real antes de reservar.

## Passo 5 — Briefar

Subagente nasce **frio**: zero contexto da sessão. Todo token que ele gasta redescobrindo o que
você já sabe é desperdício. O briefing é o principal instrumento de economia — use o template
completo da seção "Template de briefing" abaixo.

## Passo 6 — Integrar em série

Agentes entregam dentro dos próprios módulos. **O orquestrador** aplica, um de cada vez:

1. Wiring / registro de rota / injeção de dependência
2. Migration reservada
3. Suíte completa de teste
4. Só então integra a próxima entrega

Rodar a suíte após **cada** integração (não só no fim) é o que torna óbvio qual entrega quebrou o
quê. Investigar uma quebra depois de integrar tudo custa mais que as execuções extras.

> **Não use ciclos de `git stash push/pop` para isolar a verificação de cada
> commit.** Já causou perda silenciosa de arquivos em produção desta skill — sem
> erro, sem conflito reportado. Verifique contra a árvore de trabalho completa, ou
> use `git worktree` para isolamento de verdade. Detalhes e como recuperar se
> acontecer mesmo assim: `references/protocolo-paralelo.md`, seção "Verificação
> isolada por commit".

## Portões de governança

### 1. Segregação de funções
Quem implementa não aprova. A revisão precisa ser uma **invocação separada** — nunca peça ao
agente que escreveu para "revisar a própria segurança". Ele carrega o viés de quem já decidiu.
Em domínio crítico isso deixa de ser boa prática e vira requisito de conformidade
(ex.: PCI DSS 6.4.2, SOX, ISO 27001 A.8.31).

### 2. Revisão independente obrigatória
Dispare o `reviewer` antes de considerar pronta qualquer mudança que toque área declarada como
sensível no manifesto. Fora dessas áreas, revisão é opcional — não gaste modelo caro revisando
mudança de documentação.

### 3. Proibições absolutas para subagentes
- **Nunca commitar ou publicar.** Commit, push, deploy e release são decisão humana.
- **Nunca aplicar migration** fora de ambiente local descartável.
- **Nunca desabilitar, pular ou afrouxar teste** para fazer a suíte passar.
- **Nunca tocar em segredo ou credencial**, nem "só para testar".
- **Nunca inventar evidência.** Se não rodou, não afirma que passou.

### 4. Evidência antes de alegação
Toda entrega de implementação vem com saída **real** dos comandos de build, lint e teste, mais o
que foi verificado de ponta a ponta. Alegação sem output é trabalho não concluído — mande refazer.

### 5. Trilha de auditoria
Decisão que altere regra de negócio, segurança ou conformidade **vira registro no repositório**
(ADR, log de decisões, o que o projeto usar) com justificativa e impacto — mesmo quando tomada por
um subagente. Decisão que só existe no relatório do agente é decisão perdida.

## Orçamento de tokens

1. **Não delegue tarefa menor que ~3 chamadas de ferramenta.** O overhead de spawn supera o ganho.
2. **Se você não consegue listar os arquivos que o agente vai tocar, não delegue ainda.** Você
   ainda não entendeu a tarefa — e ele vai gastar tokens redescobrindo o que você já sabe.
3. **Aponte seções, não arquivos inteiros.** Documento de contexto grande consome orçamento à toa;
   diga "seção X do arquivo Y".
4. **Modelo proporcional à tarefa.** Sync mecânico de doc não é trabalho de modelo de raciocínio.
5. **Continue agente vivo** (mensagem para o agente existente) em vez de spawnar outro frio na
   mesma linha de trabalho — o contexto já está pago.
6. **Peça relatório comprimido.** Sem repetir o que já está na spec; só o que mudou e a evidência.
7. **Fan-out só com independência real.** Agentes que colidem geram retrabalho — o dobro de tokens
   do caminho sequencial.
8. **Limite prático: 2 a 3 agentes simultâneos.** Acima disso, o custo de integrar e rastrear
   conflito come o ganho de paralelismo.

## Template de briefing

```
ESCOPO: <uma frase: o que entregar>

ARQUIVOS QUE VOCÊ PODE TOCAR (allowlist — nada fora daqui):
- <caminho/modulo/...>
- <caminho/teste/...>

NÃO TOQUE (zona de contenção — o orquestrador aplica):
- <wiring / roteamento / migrations / docs de índice / changelog>

CONTEXTO MÍNIMO PARA LER (não leia mais que isto):
- <arquivo de convenções do projeto>
- <spec/ticket, seções específicas>
- <arquivo análogo já pronto, como referência de padrão>

RESERVAS JÁ FEITAS: <migration NNN, ADR NNN, ID NNN — ou "nenhuma necessária">

ENTREGÁVEL:
- código + testes passando (cole a saída real de build/lint/teste)
- lista do que precisa ser wireado na zona de contenção (descreva, não aplique)
- decisões que fugiram da spec (viram registro de decisão; quem grava é o orquestrador)

RELATÓRIO: comprimido. Sem repetir o que já está na spec.
```

## Sinais de que você paralelizou demais

- Mais tempo integrando do que os agentes gastaram implementando
- Dois relatórios descrevendo mudança no mesmo arquivo
- Suíte quebrando por motivo que nenhum dos agentes reconhece
- Você relendo o mesmo arquivo várias vezes para entender o estado atual

Qualquer um destes: volte ao sequencial na próxima rodada.

## Quando NÃO usar esta skill

- Tarefa única e pequena: faça direto, sem delegar.
- Investigação exploratória onde você ainda não sabe o que procura.
- Qualquer coisa que envolva commit, push, deploy ou ambiente compartilhado — decisão humana.

---

## Referências

- `references/manifesto-projeto.md` — template do manifesto (preencher por projeto)
- `references/checklist-revisao-critica.md` — checklist do portão de revisão, com perfis por domínio
- `references/protocolo-paralelo.md` — protocolo operacional detalhado de fan-out e recuperação de conflito
- `templates/agents/` — definições prontas dos quatro perfis
