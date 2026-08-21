# Roteamento híbrido de provedor (Claude + terceiro via API Anthropic-compatível)

Ler quando quiser rodar perfis mecânicos (`builder`, `scribe`) num provedor mais barato que a
Anthropic, mantendo `planner` e `reviewer` — os perfis não-repetíveis por natureza — em Claude de
verdade. **Opcional.** Sem esta seção, a skill funciona exatamente como antes, 100% em Claude.

Validado em 2026-08-20 contra a API Anthropic-compatível da DeepSeek (`deepseek-v4-pro` /
`deepseek-v4-flash`), mas a técnica é genérica: qualquer provedor que exponha um endpoint no
formato Anthropic Messages serve.

---

## O limite que isso não cruza

> `ANTHROPIC_BASE_URL` muda **para onde** a requisição vai, não **quem responde**. — doc oficial
> do Claude Code (`/docs/en/model-config`)

Dentro de **uma única sessão/processo** do Claude Code, `ANTHROPIC_BASE_URL` é global. Não existe
"este subagente vai pro provedor X, a sessão principal fica no Y" nativamente — o campo `model` no
frontmatter de um subagente escolhe **qual nome de modelo** é enviado, não **para qual host** a
requisição vai. Misturar provedores de verdade dentro da mesma sessão exige um LLM gateway
(LiteLLM, Portkey etc.) fazendo o roteamento por nome de modelo — infraestrutura própria, que a
Anthropic explicitamente não endossa nem audita para modelos não-Claude.

**O padrão desta seção evita essa infraestrutura inteira**: em vez de rotear dentro da sessão, o
orquestrador (a sessão principal, em Claude de verdade, com todas as ferramentas e MCP) dispara um
**processo `claude -p` separado**, com `ANTHROPIC_BASE_URL`/`ANTHROPIC_API_KEY` do terceiro
setados **só naquele comando**. É delegação por subprocesso, não roteamento nativo — mais simples
de auditar, sem proxy pra manter no ar.

## Quais perfis são candidatos

| Perfil | Rotear para terceiro? | Por quê |
|---|---|---|
| `builder` | Sim, se a tarefa não depende de MCP | Trabalho mecânico guiado por spec já pronta do `planner`; rede de segurança própria (TDD + verificação E2E real) absorve a diferença de qualidade |
| `scribe` | Sim | Sincronizar doc de trabalho já verificado é o caso mais barato de todos |
| `planner` | Não, por padrão | Decisão de arquitetura te custa caro se sair errada; mantenha em Claude a menos que a tarefa seja mecânica o bastante para justificar |
| `reviewer` | **Nunca** | É o perfil un-repetível (Portão 2 do `SKILL.md`) — o custo de um achado escapando supera qualquer economia de token |

Isso é uma dimensão **ortogonal** à tabela de modelo sugerido do `SKILL.md`: modelo escolhe o
tier (Haiku/Sonnet/Opus); esta seção escolhe o provedor. As duas se combinam.

## Limitações do lado do terceiro (verificadas contra a doc oficial da DeepSeek, 2026-08-20)

Não suportado pelo endpoint Anthropic-compatível da DeepSeek:

- **MCP** — qualquer subagente cuja tarefa dependa de um MCP server (browser automation,
  dashboards, etc.) não pode ir por este caminho.
- Imagens e documentos anexados.
- `redacted_thinking`, `budget_tokens` (thinking com orçamento).
- `cache_control` (prompt caching) — **é ignorado**. Contexto grande repetido cobra preço cheio
  em toda chamada; isso muda a conta de custo em favor da Anthropic para briefings longos.
- `top_k`, `disable_parallel_tool_use`.

Confirme o estado atual antes de depender de qualquer um destes — provedores mudam suporte com o
tempo; a doc oficial do provedor é a fonte da verdade, não este arquivo.

## Aviso de custo: `total_cost_usd` do `claude -p --output-format json` mente aqui

Quando `ANTHROPIC_BASE_URL` aponta pra um terceiro, o Claude Code **não sabe** que a requisição
foi remapeada do outro lado. O campo `total_cost_usd` do output JSON é calculado localmente contra
a tabela de preço da Anthropic para o nome de modelo que **você enviou** (ex.: `claude-opus-5`),
não o que o terceiro efetivamente cobrou (`deepseek-v4-pro`). Não use esse campo como gasto real
neste modo — confira o dashboard de billing do provedor.

## Economia real medida — DeepSeek v4-pro vs. estimativa Anthropic, mesmo volume de token

Não é benchmark nem promessa de preço futuro — é o que aconteceu nesta sessão, com os dois lados
medidos: o total de token do Claude Code (`--output-format json`, dois runs de teste tier `opus`) e
o billing real do painel da DeepSeek pro mesmo período. Os dois batem exatamente:

| | Claude Code (estimativa Anthropic p/ `claude-opus-5`) | Painel DeepSeek (real, `deepseek-v4-pro`) |
|---|---|---|
| Tokens | 257.203 (57.682 + 199.521, dois runs) | 257.203 (bate exato) |
| Requisições | 2 runs, 10 turnos | 8 API requests |
| Custo | **$0,4841** (`$0,166066` + `$0,318073`, soma dos dois `total_cost_usd`) | **$0,05** |

**≈ 89,7% mais barato (≈ 9,7×) no mesmo volume de token, tier Opus/`deepseek-v4-pro`.** O número da
coluna DeepSeek é o que o usuário confirmou no próprio painel de billing, não uma estimativa —
é o dado mais confiável que existe pra esse lado da comparação, porque nem o Claude Code nem esta
skill têm visibilidade do preço real do terceiro (ver aviso acima). O da coluna Anthropic é a
estimativa que o Claude Code teria mostrado se essas mesmas chamadas tivessem ido pro `claude-opus-5`
de verdade — é o ponto de comparação disponível, não o preço de tabela oficial confirmado por fatura.

Isso é para trabalho tier `opus` (`builder` em tarefa mais pesada, ou o `scribe` deste exemplo
rodando com `--model opus`). Perfis rodando em `deepseek-v4-flash` (`sonnet`/`haiku`) têm sua
própria relação de preço, não medida ainda — não extrapole este número pra lá sem medir.

### Como reproduzir esta medição no seu próprio uso

O número acima é specífico desta sessão — modelo, volume de token e o que o provedor cobrou por
ele mudam com o tempo. Pra medir o seu:

1. Rode uma tarefa real (não um teste de 3 tokens — o efeito do cache de prompt só aparece em
   volume real) pelo padrão de subprocesso descrito neste arquivo, com `--output-format json`.
2. Some `total_cost_usd` de cada chamada — é a estimativa Anthropic (não o custo real), mas dá o
   volume de token equivalente pra comparar.
3. Some os tokens (`input_tokens + cache_read_input_tokens + cache_creation_input_tokens +
   output_tokens`) de cada chamada.
4. Confira o painel de billing do provedor pro mesmo período — o total de token de lá deveria bater
   (ou ficar muito perto) do que você somou no passo 3. Se não bater, o provedor pode contar token
   de um jeito diferente (ex.: não separar cache read/write) — investigue antes de comparar custo.
5. `redução = 1 - (custo_real_provedor / soma_total_cost_usd_anthropic)`.

## Fontes

Este arquivo se apoia em três tipos de fonte. Provedores mudam mapeamento, limitação e preço com o
tempo — trate isto como um resumo datado, não a referência normativa; confira a fonte oficial antes
de depender de qualquer afirmação técnica específica.

**Documentação oficial da DeepSeek** (mapeamento de modelo, formato de erro, limitações do endpoint
Anthropic-compatível):
- [`api-docs.deepseek.com/guides/anthropic_api`](https://api-docs.deepseek.com/guides/anthropic_api/) — endpoint `https://api.deepseek.com/anthropic`, mapeamento `claude-opus-*` → `deepseek-v4-pro` / `claude-sonnet-*`, `claude-haiku-*` → `deepseek-v4-flash`, campos não suportados (MCP, imagem/documento, `cache_control`, `redacted_thinking`, `budget_tokens`, `top_k`, `disable_parallel_tool_use`).
- [`api-docs.deepseek.com/news/news260813`](https://api-docs.deepseek.com/news/news260813/) — GA do DeepSeek-V4-Pro (2026-08-13), 1M de contexto default em todos os serviços oficiais.
- [`api-docs.deepseek.com/news/news260424`](https://api-docs.deepseek.com/news/news260424/) — preview do DeepSeek-V4 (2026-04-24): V4-Pro (1,6T total / 49B ativos) e V4-Flash (284B total / 13B ativos), arquitetura MoE.

**Documentação oficial da Anthropic / Claude Code** (por que o roteamento é por subprocesso, não
nativo):
- [`code.claude.com/docs/en/model-config`](https://code.claude.com/docs/en/model-config) — `ANTHROPIC_BASE_URL` muda pra onde a requisição vai, não quem responde; aliases de modelo (`opus`/`sonnet`/`haiku`) resolvem client-side antes do envio.
- [`code.claude.com/docs/en/llm-gateway`](https://code.claude.com/docs/en/llm-gateway) — a Anthropic não endossa, mantém nem audita gateway de terceiro pra modelos não-Claude; roteamento nativo por modelo dentro da sessão exige essa infraestrutura.

**Medição direta** (não é documentação de terceiro — é dado desta skill, com metodologia
reproduzível na seção anterior):
- Validação de rota (chave válida / chave inválida, formato de erro do provedor): 2026-08-20, ver "Como isso foi validado" acima.
- Economia de custo (257.203 tokens, $0,4841 estimado vs. $0,05 real, ≈89,7%/≈9,7×): 2026-08-20, tier Opus/`deepseek-v4-pro`, confirmado no painel de billing da DeepSeek pelo usuário do projeto onde esta skill foi validada. Não é benchmark de terceiro nem número de marketing do provedor — é o que aconteceu numa sessão real, com os dois lados (contagem local do Claude Code e billing do provedor) medidos e batendo exato em tokens.

Achou um mapeamento, limitação ou número desatualizado? É exatamente o tipo de contribuição que
`CONTRIBUTING.md` pede — abra um PR atualizando a fonte junto com a data.

## Setup: nunca deixe a chave passar pelo contexto da conversa

A chave do terceiro **nunca** deve aparecer digitada no chat com o orquestrador nem em um comando
que o orquestrador ecoa de volta pra conversa. Padrão:

1. O humano gera a chave no painel do provedor e salva num arquivo **fora do repositório**
   (scratchpad da sessão, ou qualquer caminho fora de qualquer diretório versionado), rodando o
   comando ele mesmo no próprio terminal — nunca colando a chave numa mensagem pro agente:

   ```powershell
   Set-Content -Path "$env:TEMP\provider.key" -Value "SUA_CHAVE" -NoNewline -Encoding utf8
   ```

2. O orquestrador lê o arquivo e seta a variável **dentro do mesmo comando** que invoca o
   `claude -p`, e desfaz logo em seguida — tudo num único processo, pra a chave nunca precisar
   sobreviver entre chamadas de ferramenta nem aparecer em texto que o agente gera:

   ```powershell
   $env:ANTHROPIC_API_KEY = (Get-Content "$env:TEMP\provider.key" -Raw).Trim()
   $env:ANTHROPIC_BASE_URL = "https://api.deepseek.com/anthropic"
   Set-Location <diretorio-da-tarefa>
   claude -p "<briefing do template padrão>" --model opus --output-format json
   Remove-Item Env:\ANTHROPIC_API_KEY
   Remove-Item Env:\ANTHROPIC_BASE_URL
   ```

   `--model opus` aqui não liga o subprocesso a um Opus real — é o nome que a DeepSeek mapeia
   para `deepseek-v4-pro` do lado dela (`claude-sonnet`/`claude-haiku` mapeiam para
   `deepseek-v4-flash`). Confirme o mapeamento vigente na doc do provedor antes de assumir.

3. Modo não-interativo (`-p`) normalmente trava esperando aprovação de escrita de arquivo. Sem um
   humano pra aprovar o prompt de permissão de um subprocesso, a alternativa menos ampla é
   `--permission-mode acceptEdits` (aceita edição/escrita no diretório da tarefa, sem abrir mão do
   resto das restrições) em vez de `--dangerously-skip-permissions`. Trate a allowlist de arquivos
   do briefing (Passo 5 do `SKILL.md`) como o limite real de blast radius, já que o modelo por trás
   não é o Claude que você audita normalmente.

4. Apague o arquivo da chave assim que o teste/lote de tarefas terminar.

## Isole o subagente do pipeline completo do projeto — senão ele reimplementa a orquestração sozinho

**O erro mais caro que este padrão pode cometer, e o briefing padrão de "Template de briefing" do
`SKILL.md` não previne sozinho.** Quase todo projeto que usa esta skill tem um `CLAUDE.md`/
`WORKFLOW.md` que instrui **qualquer sessão do Claude Code** a, como primeira ação, ler o pipeline
completo do projeto — spec → builder → testes → revisão de segurança → commit. Isso é correto e
desejável numa sessão normal. Mas o subprocesso `claude -p` roteado para o terceiro **não sabe que
é só um papel dentro de uma orquestração maior** — ele abre no mesmo repositório, lê os mesmos
arquivos, e não tem como diferenciar "sou a sessão principal" de "sou um builder delegado". Sem
instrução em contrário, ele segue o `WORKFLOW.md` do jeito que está escrito: tenta rodar o pipeline
inteiro sozinho.

Isso já aconteceu em produção desta skill: um subagente delegado (só deveria implementar um fix)
leu o `WORKFLOW.md` do projeto, decidiu que precisava fechar o ciclo completo, e **invocou o
subagente `favo-security-reviewer` por conta própria** — que então rodou na mesma infraestrutura do
terceiro (porque `ANTHROPIC_BASE_URL` é global pro processo inteiro, ver "O limite que isso não
cruza" acima), não em Claude real. O relatório voltou dizendo "revisão de segurança aprovada" — uma
aprovação que não vale nada, porque o gate mais importante desta skill (Portão 2, revisor sempre
independente) foi silenciosamente contornado pelo próprio subagente, sem má intenção, só seguindo a
documentação do repositório ao pé da letra. Descoberto porque o relatório citava achados específicos
de uma "revisão" que o orquestrador nunca disparou — se não fosse por isso, teria passado
despercebido.

**O fix é contextualizar explicitamente, todo briefing, antes de qualquer instrução de tarefa.**
Adicione este preâmbulo (adapte os nomes de subagente do seu projeto):

```
CONTEXTO DE ORQUESTRACAO - LEIA ISTO PRIMEIRO:
Voce esta rodando como UM UNICO PAPEL (builder) dentro de uma orquestracao multi-agente maior.
A sessao principal (orquestrador) cuida de planejamento, revisao de seguranca e commit/push -
isso NAO e trabalho seu.

Este repositorio tem um CLAUDE.md/WORKFLOW.md que instrui QUALQUER sessao do Claude Code a
seguir um pipeline completo. IGNORE ESSA INSTRUCAO PARA ESTA TAREFA - voce e SO o passo
builder, ja dentro do pipeline.

REGRAS ABSOLUTAS:
1. NUNCA invoque outro subagente (Task/Agent tool) - especialmente nunca o de revisao de
   seguranca do projeto. Qualquer subagente que voce disparar rodaria neste MESMO provedor,
   quebrando a segregacao de funcoes que a revisao existe pra garantir.
2. NUNCA rode git commit/push/checkout -b/merge (leitura tipo status/diff/log esta OK).
3. NAO leia nem siga o WORKFLOW.md/CLAUDE.md do projeto - essas instrucoes sao pro
   orquestrador, nao pra voce. Siga SO o briefing abaixo.
4. Ao terminar implementacao + verificacao, PARE e relate. Nao tente integrar nem decidir se
   esta "pronto pra producao".
```

Coloque este bloco **antes** de qualquer separador (`---`) ou seção do briefing normal — algumas
implementações de shell/CLI podem truncar strings muito longas de forma inesperada (aconteceu uma
vez nesta skill, ver nota de robustez logo abaixo); se o preâmbulo vier primeiro, mesmo um corte
parcial ainda preserva a regra mais importante.

**Nota de robustez — não confie em string longa via argumento de linha de comando.** Na mesma
ocorrência acima, um briefing de ~7.700 caracteres passado como `claude -p $briefing` (variável
PowerShell interpolada no comando) chegou **truncado pela metade** no subagente — sem erro, sem
aviso, só sumiu o resto do texto. A causa exata não foi isolada (não é limite de tamanho do
Windows — 7.700 caracteres está bem abaixo de qualquer limite conhecido de `CreateProcess`), mas o
padrão mais robusto, testado com sucesso, é: salvar o briefing em arquivo e instruir o subagente a
**ler o arquivo com sua própria ferramenta de leitura**, em vez de depender que o texto sobreviva
inteiro à passagem de argumento:

```powershell
# Frágil - pode truncar sem aviso:
claude -p $briefingLongo --model opus ...

# Mais robusto - o proprio subagente le o arquivo completo:
claude -p "Leia o arquivo C:\caminho\briefing.txt e siga as instrucoes dele integralmente. Comece agora." --model opus ...
```

## Não espere ver isso no agents-observe

Testado em 2026-08-20: um `claude -p` (modo headless) rodando na mesma pasta de um projeto já
registrado no [agents-observe](https://github.com/simple10/agents-observe) **não** cria uma nova
sessão no dashboard — nem apontando pra Anthropic de verdade, nem pro terceiro. Isolado com
controle (mesmo comando, sem `ANTHROPIC_BASE_URL`, resultado igual): a causa é o modo `-p`, não o
roteamento pra terceiro. O hook de `SessionStart` que registra sessão parece não disparar em modo
headless. A saída de auditoria confiável deste padrão continua sendo o `--output-format json` do
próprio subprocesso (`session_id`, `total_cost_usd` — lembrando do aviso acima —, `result`), não o
dashboard.

## Como isso foi validado (não é só teoria)

Prova de que a rota realmente sai pela infraestrutura do terceiro, e não cai de volta
silenciosamente pro login normal da Anthropic — dois testes, mesmo `ANTHROPIC_BASE_URL`:

- **Chave válida** → tarefa dummy concluída com sucesso, fora de qualquer repositório real.
- **Chave inválida** (mesmo base_url) → erro **no formato de erro do próprio terceiro**
  (`"API Error: 401 Authentication Fails, Your api key: ****-xyz is invalid"` — esse texto é da
  DeepSeek, não da Anthropic). Se a rota tivesse ignorado o `ANTHROPIC_BASE_URL` e caído no login
  OAuth normal, a chave inválida não teria produzido erro nenhum.

Rode os dois lados (chave boa / chave ruim) antes de confiar num roteamento novo — sucesso sozinho
não descarta "caiu de volta pro provedor de sempre por engano".

## Antes de rotear qualquer área sensível (Portão 2 do SKILL.md)

Rotear uma tarefa para um provedor terceiro faz código, prompt e saída de ferramenta trafegarem
pela infraestrutura **daquele provedor**, fora do perímetro da Anthropic. Em domínio crítico
(pagamento, dado regulado, PCI, saúde, identidade — ver classificação de criticidade no
`SKILL.md`):

- **Nunca** roteie uma tarefa que toque área declarada como sensível no manifesto do projeto
  (`.claude/multiagente.md`) sem essa decisão estar explícita e por escrito no manifesto — não é
  algo pra decidir ad-hoc no meio de uma sessão.
- Trate como uma extensão do Portão 5 (trilha de auditoria): registrar **que** uma tarefa foi
  roteada para terceiro, e por quê, é decisão que vira registro no repositório — igual a qualquer
  outra decisão de arquitetura/segurança.
- Quando em dúvida se a área é sensível, não é. Fica em Claude.
