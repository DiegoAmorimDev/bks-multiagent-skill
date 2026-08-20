# bks-multiagent-skill

### Orquestração híbrida Claude + DeepSeek

**É esse o foco da skill:** dividir o trabalho de um time de agentes entre Claude (Anthropic) e
DeepSeek (provedor terceiro compatível com a API Anthropic) por perfil — sem gateway, sem misturar
provedor dentro da mesma sessão, sem nunca deixar a chave do terceiro passar pelo contexto da
conversa. A governança de paralelismo por trás (zonas de contenção, segregação de funções, portão
de revisão) é o que torna esse roteamento seguro em sistemas onde errar custa caro: pagamentos,
saúde, identidade, dado regulado, infraestrutura.

> **Medido em produção, não estimado:** mesmo volume de token (257.203), tier Opus/`deepseek-v4-pro`
> — estimativa Anthropic $0,4841 vs. billing real da DeepSeek $0,05. **≈89,7% mais barato (≈9,7×).**
> Detalhe e como reproduzir: [`skill/references/roteamento-hibrido-provedores.md`](skill/references/roteamento-hibrido-provedores.md).

---

## Claude vs DeepSeek — qual provedor por perfil

| Perfil | Provedor | Modelo | Custo medido (tier Opus, ver acima) | Regra |
|---|---|---|---|---|
| `reviewer` | **Claude, sempre** | Opus | — | **Nunca roteia** — portão un-repetível (Portão 2); o custo de um achado escapando supera qualquer economia |
| `planner` | Claude, por padrão | Sonnet | — | Decisão de arquitetura errada custa caro; roteia só se a tarefa for mecânica o bastante |
| `builder` | Claude **ou** DeepSeek v4-pro | Sonnet / `deepseek-v4-pro` | ≈9,7× mais barato no DeepSeek | Roteia se a spec já vem pronta do `planner` e a tarefa não depende de MCP |
| `scribe` | DeepSeek v4-pro/flash (recomendado) | Haiku / `deepseek-v4-flash` | ≈9,7× mais barato no DeepSeek | O caso mais barato de todos — sincronizar doc de trabalho já verificado |

Como funciona sem gateway, limitações confirmadas do provedor (sem MCP, sem cache de prompt), setup
que nunca deixa a chave passar pelo contexto da conversa, e como medir o *seu* custo real (não
extrapole o número acima sem medir no seu próprio uso):
[`skill/references/roteamento-hibrido-provedores.md`](skill/references/roteamento-hibrido-provedores.md).

---

## O problema que ela resolve

Ao delegar trabalho a vários agentes ao mesmo tempo, quatro coisas quebram na prática:

| Problema | Como aparece | O que a skill faz |
|---|---|---|
| **Colisão em estado compartilhado** | Dois agentes editam o arquivo de wiring; dois escolhem o mesmo número de migration | Mapa de zonas de contenção + protocolo de reserva de recursos numerados |
| **Perda de governança** | Quem escreveu o código também aprovou; decisão crítica ficou só no relatório do agente | Segregação de funções obrigatória + trilha de auditoria no repositório |
| **Desperdício de tokens** | Agente nasce frio e gasta metade do orçamento explorando o repositório | Briefing com allowlist e contexto mínimo + modelo proporcional à tarefa |
| **Custo de provedor desproporcional** | Trabalho mecânico (`builder`/`scribe`) rodando no mesmo modelo caro que a revisão de segurança | Roteamento opcional Claude + terceiro (DeepSeek), por subprocesso, sem gateway, sem tocar nos perfis onde qualidade pesa mais que preço |

---

## Instalação

```bash
# skill de projeto (só neste repositório)
mkdir -p .claude/skills/bks-multiagent-skill
cp -r skill/* .claude/skills/bks-multiagent-skill/

# ou skill de usuário (todos os seus projetos)
mkdir -p ~/.claude/skills/bks-multiagent-skill
cp -r skill/* ~/.claude/skills/bks-multiagent-skill/
```

Depois, o passo que **não** é opcional:

```bash
cp skill/references/manifesto-projeto.md .claude/multiagente.md
# preencha o manifesto com as zonas de contenção do SEU repositório
```

Sem manifesto preenchido a skill não sabe onde estão seus pontos de colisão — e paralelizar vira
aposta. Detalhes em [`docs/instalacao.md`](docs/instalacao.md).

Opcionalmente, instale os perfis de agente:

```bash
mkdir -p .claude/agents
cp skill/templates/agents/*.md .claude/agents/
# ajuste nome, modelo e caminhos de cada um para o seu projeto
```

---

## Uso rápido

A skill é invocada automaticamente quando você pede paralelismo ou delegação, ou manualmente:

```
/bks-multiagent-skill
```

Fluxo que ela impõe:

```
manifesto → criticidade → teste de independência → reserva de recursos
    → briefing → execução → integração serializada → portão de revisão → registro
```

Em uma frase: **sequencial é o padrão; paralelo precisa ser justificado.**

---

## Os quatro perfis

| Perfil | Modelo sugerido | Faz | Nunca faz |
|---|---|---|---|
| `builder` | Sonnet | Implementação, decisão de arquitetura | Commit; aprovar o próprio trabalho |
| `reviewer` | Opus | Revisão de segurança/conformidade — só lê | Editar código |
| `planner` | Sonnet | Specs, ADRs, quebra de tarefa | Código de produção |
| `scribe` | Haiku | Sincronizar docs de trabalho já verificado | Decidir o que foi feito |

Modelo (esta tabela) e provedor (tabela "Claude vs DeepSeek" no topo) são eixos independentes que
se combinam — esta tabela não muda quando você roteia `builder`/`scribe` pra DeepSeek.

Ordem dentro de uma entrega: `planner` → `builder` → `reviewer` → `scribe`. Nunca em paralelo entre
si — são dependentes por natureza.

---

## Estrutura do repositório

```
skill/
├── SKILL.md                          # a skill (é o que o agente carrega)
├── references/
│   ├── manifesto-projeto.md          # template a preencher por projeto
│   ├── checklist-revisao-critica.md  # checklist do portão, com perfis por domínio
│   ├── protocolo-paralelo.md         # fan-out, integração e recuperação de conflito
│   └── roteamento-hibrido-provedores.md  # builder/scribe em provedor terceiro, sem gateway (opcional)
└── templates/agents/                 # os quatro perfis, prontos para ajustar

docs/
├── instalacao.md                     # instalação e preenchimento do manifesto
├── contextos-de-uso.md               # quando usar, quando não usar — por contexto
└── exemplos/                         # casos completos, ponta a ponta
```

---

## Princípios

1. **Sequencial por padrão.** Retrabalho por conflito custa mais que espera.
2. **Quem implementa não aprova.** Segregação de funções não é formalidade em domínio crítico —
   é requisito de conformidade (PCI DSS 6.4.2, SOX, ISO 27001 A.8.31).
3. **Evidência antes de alegação.** Sem saída real de build/teste, o trabalho não está concluído.
4. **Zona de contenção é do orquestrador.** Subagente descreve o que precisa mudar; quem aplica
   é quem tem visão do todo.
5. **A fonte da verdade é o repositório**, não o documento de planejamento. Specs envelhecem.
6. **Se você não sabe quais arquivos o agente vai tocar, não delegue ainda.**
7. **Modelo e provedor são eixos independentes.** Rebaixar tier é risco de qualidade; trocar
   provedor é risco de dado saindo do perímetro da Anthropic — as duas trocas exigem julgamentos
   diferentes, nunca a mesma regra de bolso.

---

## Limitações conhecidas

- **Não é um sistema de plano.** A skill governa *como* delegar; o *que* construir continua no seu
  processo (specs, issues, backlog). Se você usa uma metodologia de planejamento, ela permanece.
- **Não paraleliza automaticamente.** A decisão de fan-out é do orquestrador, com base no manifesto.
- **O manifesto exige manutenção.** Zona de contenção nova que não for declarada vira conflito.
- **Não substitui revisão humana** em mudança de alto impacto. O portão de revisão automatizado
  reduz o volume que chega no humano; não elimina o humano.

---

## Evolução

Ver [`CHANGELOG.md`](CHANGELOG.md) e [`CONTRIBUTING.md`](CONTRIBUTING.md).

Melhorias vindas de uso real são as mais valiosas — especialmente novas zonas de contenção
descobertas na prática e invariantes de domínio que a revisão deveria ter pego e não pegou.

---

## Licença

MIT — ver [`LICENSE`](LICENSE).
