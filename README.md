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

## Configurar o DeepSeek

Passo a passo pra ligar o lado DeepSeek do roteamento. Sem isso, a skill funciona 100% em Claude —
esta seção é só pra quem quer os perfis `builder`/`scribe` rodando em `deepseek-v4-pro`/`flash`.

### 1. Gerar a API key

Crie uma conta e uma API key em [platform.deepseek.com](https://platform.deepseek.com). Guarde a
chave — ela só aparece uma vez na criação.

### 2. Salvar a chave sem nunca digitá-la pro agente

**Nunca cole a chave numa mensagem pro Claude.** Duas opções, dependendo de quanto você vai usar:

**Uso ocasional — arquivo temporário fora do repositório**, rodado por você mesmo no terminal:

```powershell
Set-Content -Path "$env:TEMP\deepseek.key" -Value "SUA_CHAVE_AQUI" -NoNewline -Encoding utf8
```

O orquestrador lê esse arquivo só dentro do comando que invoca o subprocesso (passo 4) e apaga o
arquivo depois — a chave nunca precisa sobreviver entre mensagens nem aparecer no texto que o
agente gera.

**Uso recorrente — variável de ambiente permanente por usuário**, pra não repetir o passo acima
toda sessão:

```powershell
$key = Get-Content "$env:TEMP\deepseek.key" -Raw
[System.Environment]::SetEnvironmentVariable("DEEPSEEK_API_KEY", $key.Trim(), "User")
Remove-Item "$env:TEMP\deepseek.key" -Force
```

**Importante:** isso salva em `DEEPSEEK_API_KEY`, uma variável própria — **nunca** salve
permanentemente em `ANTHROPIC_API_KEY`/`ANTHROPIC_BASE_URL`. Se essas duas ficarem permanentes,
toda sessão normal do Claude Code (a sua, incluindo fora desta skill) passa a tentar autenticar com
a chave da DeepSeek contra a API real da Anthropic e quebra seu login normal — é exatamente o erro
que este padrão existe pra evitar (ver "O limite que isso não cruza" na referência completa).

Uma pegadinha do Windows: `SetEnvironmentVariable(..., "User")` grava no registro, mas processos já
abertos (inclusive a sessão atual do Claude Code) não veem a mudança até reiniciar. Pra ler o valor
sem depender de reiniciar nada, leia direto do registro em vez do ambiente do processo:

```powershell
$key = [System.Environment]::GetEnvironmentVariable("DEEPSEEK_API_KEY", "User")
```

### 3. Testar a rota antes de confiar nela

Dois testes, o mesmo `ANTHROPIC_BASE_URL`, validam que a rota sai de verdade pela DeepSeek e não
cai de volta no seu login normal da Anthropic:

```powershell
$key = [System.Environment]::GetEnvironmentVariable("DEEPSEEK_API_KEY", "User")
$env:ANTHROPIC_API_KEY = $key
$env:ANTHROPIC_BASE_URL = "https://api.deepseek.com/anthropic"
claude -p "diga apenas 'ok'" --model opus --permission-mode acceptEdits --output-format json
Remove-Item Env:\ANTHROPIC_API_KEY; Remove-Item Env:\ANTHROPIC_BASE_URL
```

Sucesso esperado: resposta normal, `"is_error":false` no JSON. Se você quiser confirmar que a rota
é real (não caiu no login antigo por engano), repita com uma chave errada — o erro deve vir no
formato da própria DeepSeek (`"API Error: 401 Authentication Fails..."`), não um erro genérico da
Anthropic. Detalhe da prova completa: [seção "Como isso foi validado"](skill/references/roteamento-hibrido-provedores.md#como-isso-foi-validado-não-é-só-teoria).

### 4. Rodar de verdade — delegar uma tarefa pro DeepSeek

O padrão de produção é um `claude -p` separado, com a allowlist de arquivos do briefing como limite
de blast radius:

```powershell
$key = [System.Environment]::GetEnvironmentVariable("DEEPSEEK_API_KEY", "User")
$env:ANTHROPIC_API_KEY = $key
$env:ANTHROPIC_BASE_URL = "https://api.deepseek.com/anthropic"
Set-Location <diretorio-da-tarefa>
claude -p "<briefing do template padrão — escopo, allowlist, contexto mínimo>" `
  --model opus --permission-mode acceptEdits --output-format json
Remove-Item Env:\ANTHROPIC_API_KEY; Remove-Item Env:\ANTHROPIC_BASE_URL
```

`--model opus` mapeia pra `deepseek-v4-pro` do lado da DeepSeek; troque por `--model sonnet` ou
`--model haiku` pra `deepseek-v4-flash` (perfil `scribe`, trabalho mais mecânico). Guia completo —
setup detalhado, limitações confirmadas do provedor, por que isso não aparece no `agents-observe`,
e como medir seu próprio custo —: [`skill/references/roteamento-hibrido-provedores.md`](skill/references/roteamento-hibrido-provedores.md).

### Erros comuns

| Sintoma | Causa provável | Ação |
|---|---|---|
| `401 Authentication Fails` | Chave errada ou expirada | Gere uma nova em platform.deepseek.com |
| Subprocesso trava esperando aprovação | Modo `-p` sem `--permission-mode` | Adicione `--permission-mode acceptEdits`, nunca `--dangerously-skip-permissions` sem necessidade |
| Tarefa falha silenciosamente ou ignora um MCP server | DeepSeek não suporta MCP | Não roteie essa tarefa — mantenha em Claude |
| Chave não aparece disponível numa sessão nova | `SetEnvironmentVariable` gravou no registro, processo atual não releu | Use `GetEnvironmentVariable(..., "User")` explícito, não `$env:` direto |
| `total_cost_usd` do JSON parece caro demais | É a estimativa Anthropic, não o custo real da DeepSeek | Confira o painel de billing do provedor |

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

| Perfil | Modelo Claude | Uso do DeepSeek v4-pro/flash | Faz | Nunca faz |
|---|---|---|---|---|
| `builder` | Sonnet | `deepseek-v4-pro` (via alias `opus`) quando a tarefa é mecânica o bastante e não depende de MCP — spec já vem pronta do `planner`, rede de segurança é o próprio TDD + verificação E2E real | Implementação, decisão de arquitetura | Commit; aprovar o próprio trabalho |
| `reviewer` | Opus | **Nenhum — nunca roteia**, para nenhum dos dois modelos DeepSeek. Portão un-repetível (Portão 2): o custo de um achado escapando supera qualquer economia de token, independente de o provedor ser confiável ou não com o dado | Revisão de segurança/conformidade — só lê | Editar código |
| `planner` | Sonnet | `deepseek-v4-flash` (via alias `sonnet`/`haiku`) só se a tarefa for mecânica o bastante — decisão de arquitetura errada custa caro, então o padrão é Claude | Specs, ADRs, quebra de tarefa | Código de produção |
| `scribe` | Haiku | `deepseek-v4-flash` — o caso recomendado por padrão; é o trabalho mais barato e mais mecânico dos quatro perfis, e onde a diferença de custo mais compensa | Sincronizar docs de trabalho já verificado | Decidir o que foi feito |

Modelo Claude e provedor DeepSeek são eixos independentes que se combinam, não uma substituição um
do outro — a coluna "Uso do DeepSeek" desta tabela é o mesmo roteamento já resumido na tabela
"Claude vs DeepSeek" acima, detalhado por perfil. `--model opus` no comando do subprocesso mapeia
para `deepseek-v4-pro` do lado da DeepSeek; `--model sonnet`/`--model haiku` mapeiam para
`deepseek-v4-flash` — ver a fonte oficial na seção "Fontes e citações" abaixo.

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

## Fontes e citações

O roteamento Claude + DeepSeek descrito neste README se apoia em três tipos de fonte — documentação
oficial dos dois lados, e medição direta feita nesta skill. Detalhe completo, incluindo como
reproduzir a medição de custo, em
[`skill/references/roteamento-hibrido-provedores.md`](skill/references/roteamento-hibrido-provedores.md#fontes).

- **DeepSeek — API compatível com Anthropic, mapeamento de modelo e limitações:**
  [`api-docs.deepseek.com/guides/anthropic_api`](https://api-docs.deepseek.com/guides/anthropic_api/)
- **DeepSeek — GA do V4-Pro (2026-08-13):**
  [`api-docs.deepseek.com/news/news260813`](https://api-docs.deepseek.com/news/news260813/)
- **DeepSeek — preview do V4 (2026-04-24), specs de V4-Pro (1.6T MoE) e V4-Flash (284B MoE):**
  [`api-docs.deepseek.com/news/news260424`](https://api-docs.deepseek.com/news/news260424/)
- **Anthropic Claude Code — `ANTHROPIC_BASE_URL`, aliases de modelo, por que roteamento nativo
  dentro da sessão exige gateway:**
  [`code.claude.com/docs/en/model-config`](https://code.claude.com/docs/en/model-config)
- **Anthropic Claude Code — LLM gateways, por que a Anthropic não endossa roteamento pra modelos
  não-Claude:** [`code.claude.com/docs/en/llm-gateway`](https://code.claude.com/docs/en/llm-gateway)
- **Métrica de economia (89,7%/9,7×):** medição direta, não documentação de terceiro — ver "Fontes"
  na referência completa para a metodologia e os números brutos.

Provedores mudam mapeamento, limitação e preço com o tempo. Antes de depender de qualquer
afirmação técnica específica deste README, confira a fonte oficial correspondente — este documento
é um resumo, não a referência normativa.

---

## Licença

MIT — ver [`LICENSE`](LICENSE).
