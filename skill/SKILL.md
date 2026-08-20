---
name: bks-multiagent-skill
description: >
  Orquestração multi-agente Claude + DeepSeek: governança de paralelismo e roteamento de provedor
  (Claude + terceiro compatível com a API Anthropic, ex. DeepSeek v4-pro/flash) para sistemas de
  alta criticidade. Define quais tarefas podem rodar em paralelo sem corromper estado
  compartilhado, quais exigem revisão independente obrigatória, como briefar subagentes gastando
  o mínimo de tokens, e quais perfis (`builder`/`scribe`) podem rodar num provedor mais barato sem
  abrir mão de qualidade nos papéis que importam (`reviewer` nunca roteia). Redução de custo
  medida em produção: ≈89,7% (≈9,7×) no mesmo volume de token, tier Opus/deepseek-v4-pro — ver
  `references/roteamento-hibrido-provedores.md`. Funciona em qualquer stack — cada projeto declara
  suas zonas de contenção e áreas sensíveis em um manifesto. Use ao delegar implementação, revisão
  ou documentação a subagentes, especialmente ao rodar 2+ tarefas ao mesmo tempo ou ao otimizar
  custo de token entre Claude e um provedor terceiro. Trigger: "paralelo", "multi-agente",
  "delegar", "subagente", "fan-out", "vários agentes", "orquestrar", "deepseek", "deepseek v4",
  "claude + deepseek", "orquestração híbrida", "roteamento de provedor", "economia de token",
  "reduzir custo de agente", "cost reduction", "hybrid orchestration", "/bks-multiagent-skill".
---

# Orquestração multi-agente com governança — e economia de provedor (Claude + DeepSeek)

Paralelismo entre agentes não é otimização gratuita. Ele troca tempo por **risco de corrupção de
estado compartilhado** e por **perda de rastreabilidade** de quem decidiu o quê. Em sistema de
baixa criticidade isso é aceitável na base do "depois arruma". Em sistema que move dinheiro,
guarda dado pessoal ou responde a auditoria externa, não é.

Esta skill governa **como** delegar, em dois eixos independentes que se combinam:

1. **Modelo/tier** (Haiku/Sonnet/Opus) — proporcional à complexidade da tarefa.
2. **Provedor** (Anthropic ou um terceiro compatível com a API Anthropic, ex. DeepSeek) — para
   rodar os perfis mecânicos (`builder`/`scribe`) por uma fração do custo, sem tocar nos papéis
   onde qualidade e confiabilidade pesam mais que preço (`reviewer`, decisão de arquitetura).

Ela não substitui o processo de planejamento do projeto — o plano continua onde já estava (specs,
issues, backlog). Aqui se decide: paraleliza ou não, quem revisa, o que cada agente pode tocar,
quanto contexto ele recebe, e **em qual provedor cada perfil roda**.

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
| `builder` | Médio (Sonnet) | Implementação, decisão de arquitetura | Commit; aprovar o próprio trabalho |
| `reviewer` | Alto (Opus) | Revisão de segurança/conformidade — só lê | Editar código |
| `planner` | Médio (Sonnet) | Specs, ADRs, quebra de tarefa | Código de produção |
| `scribe` | Baixo (Haiku) | Sincronizar docs de trabalho já verificado | Decidir o que foi feito |

Ordem natural dentro de **uma** entrega: `planner` → `builder` → `reviewer` → `scribe`.
São dependentes por natureza — nunca paralelize entre si na mesma entrega.

### Confirme que o agente registrou

Criar o arquivo de definição não garante que o harness carregou o agente. **Frontmatter inválido é
descartado em silêncio**: sem erro, sem aviso — o agente só não aparece na lista de tipos
disponíveis, e a chamada falha com "agent type not found" no pior momento possível (em geral o
portão de revisão, já no fim da entrega, quando o custo de contornar é maior).

Depois de criar ou editar qualquer definição, **liste os tipos de agente disponíveis e confirme que
o nome está lá** antes de contar com ele. Trate ausência como bug de configuração, não como
indisponibilidade do modelo.

Causa já observada em produção desta skill: **`description:` sem aspas contendo `: `
(dois-pontos seguido de espaço) em arquivo com quebras de linha CRLF**. Em YAML, `: ` dentro de um
escalar simples é ambíguo; somado ao `\r` do CRLF, o parse falha e a definição inteira é
descartada. Arquivo criado no Windows nasce em CRLF por padrão — e uma description como
`Read-only: reporta achados` dispara exatamente isso. O sintoma engana: parece restrição de modelo
ou bug do harness, e leva a diagnóstico errado.

Por isso os templates em `templates/agents/` trazem `description:` **entre aspas**. Mantenha as
aspas ao adaptar, e prefira `—` ou vírgula a `:` dentro do texto.

Se mesmo assim não registrar, isole em vez de adivinhar: recrie a definição num arquivo novo,
mínimo e em LF. Se a cópia registra e o original não, a diferença está no arquivo — não no nome
do agente nem no modelo escolhido.

## Roteamento de provedor — Claude + terceiro (DeepSeek)

Modelo e provedor são eixos independentes. Modelo decide o tier (Haiku/Sonnet/Opus); provedor
decide **onde** a requisição é processada — Anthropic direto, ou um terceiro que expõe API
compatível com o formato Anthropic (`ANTHROPIC_BASE_URL`/`ANTHROPIC_API_KEY`). Os dois se
combinam por perfil:

| Perfil | Roteia para terceiro? | Por quê |
|---|---|---|
| `builder` | Sim, se a tarefa não depende de MCP | Mecânico o bastante quando a spec já vem pronta do `planner`; a rede de segurança é o próprio TDD + verificação E2E real |
| `scribe` | Sim | Sincronizar doc de trabalho já verificado é o caso mais barato de todos |
| `planner` | Não, por padrão | Decisão de arquitetura errada custa caro — mantenha em Claude a menos que a tarefa seja mecânica o bastante |
| `reviewer` | **Nunca** | Portão 2 (un-repetível): o custo de um achado escapando supera qualquer economia de token, independente de o provedor terceiro ser confiável ou não com o dado |

**Economia medida (não é benchmark, é o que aconteceu em produção, 2026-08-20):** mesmo volume de
token (257.203, os dois lados batem exato), tier Opus/`deepseek-v4-pro` — estimativa do Claude Code
pra `claude-opus-5` ($0,4841) vs. billing real confirmado no painel da DeepSeek ($0,05). **≈89,7%
mais barato, ≈9,7×.** Detalhe completo, com a ressalva de que isso não extrapola pra tier
Sonnet/Haiku sem medir: `references/roteamento-hibrido-provedores.md`.

**Como, sem gateway:** dentro de uma única sessão, `ANTHROPIC_BASE_URL` é global — muda para onde
a requisição vai, não quem responde. Não há "este subagente vai pro provedor X" nativo sem um LLM
gateway (infra própria, que a Anthropic não endossa para modelos não-Claude). O padrão que esta
skill usa evita essa infraestrutura: o orquestrador (sessão principal, Claude, com todas as
ferramentas) dispara um **processo `claude -p` separado**, com as duas variáveis setadas **só
naquele processo**, nunca globalmente:

```powershell
$env:ANTHROPIC_API_KEY = (chave do terceiro, lida de um arquivo fora do repo — nunca digitada no chat)
$env:ANTHROPIC_BASE_URL = "https://api.deepseek.com/anthropic"
claude -p "<briefing padrão, allowlist de um arquivo/módulo>" --model opus --permission-mode acceptEdits --output-format json
Remove-Item Env:\ANTHROPIC_API_KEY; Remove-Item Env:\ANTHROPIC_BASE_URL
```

`--model opus` aqui não liga a Opus de verdade — é o nome que a DeepSeek mapeia server-side para
`deepseek-v4-pro` (`claude-sonnet`/`claude-haiku` mapeiam para `deepseek-v4-flash`). Confirme o
mapeamento vigente na doc do provedor antes de assumir que continua igual.

**Antes de rotear área sensível do manifesto** (financeiro, PCI, dado regulado): por padrão esta
skill assume que não se roteia — trafegar código/prompt/saída de ferramenta por infraestrutura de
terceiro é uma decisão de risco que só o dono do projeto pode tomar, e que precisa virar registro
explícito no manifesto (Portão 5, trilha de auditoria), não uma escolha implícita no meio de uma
sessão. Quando o dono do projeto autoriza essa cobertura ampliada, registre a decisão do mesmo jeito
que qualquer outra mudança de regra de segurança/conformidade — com data e justificativa.

Guia completo (setup seguro da chave, limitações confirmadas do provedor — sem MCP, sem cache de
prompt, sem imagem/documento —, por que `total_cost_usd` do `--output-format json` não reflete o
que o terceiro cobrou de fato, e por que este padrão não aparece no agents-observe):
`references/roteamento-hibrido-provedores.md`.

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
- `references/roteamento-hibrido-provedores.md` — rotear `builder`/`scribe` pra provedor terceiro mais barato sem gateway (opcional)
- `templates/agents/` — definições prontas dos quatro perfis
