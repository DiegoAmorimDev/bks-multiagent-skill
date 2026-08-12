# Contextos de uso

Guia objetivo de quando a skill ajuda, quando atrapalha, e como ela muda de forma conforme o
contexto do projeto.

---

## Índice

- [Decisão rápida: delegar ou fazer direto?](#decisão-rápida-delegar-ou-fazer-direto)
- [Contexto 1 — Feature nova em módulo isolado](#contexto-1--feature-nova-em-módulo-isolado)
- [Contexto 2 — Duas features independentes ao mesmo tempo](#contexto-2--duas-features-independentes-ao-mesmo-tempo)
- [Contexto 3 — Correção em área sensível](#contexto-3--correção-em-área-sensível)
- [Contexto 4 — Refatoração ampla](#contexto-4--refatoração-ampla)
- [Contexto 5 — Investigação de bug](#contexto-5--investigação-de-bug)
- [Contexto 6 — Documentação atrasada](#contexto-6--documentação-atrasada)
- [Contexto 7 — Monólito legado sem fronteira clara](#contexto-7--monólito-legado-sem-fronteira-clara)
- [Contexto 8 — Time pequeno, orçamento apertado](#contexto-8--time-pequeno-orçamento-apertado)
- [Antipadrões](#antipadrões)

---

## Decisão rápida: delegar ou fazer direto?

```
A tarefa cabe em menos de ~3 chamadas de ferramenta?
   └─ sim → faça direto. Spawn custa mais que a tarefa.

Você consegue listar os arquivos que serão tocados?
   └─ não → você ainda não entendeu a tarefa. Explore primeiro, delegue depois.

Existem 2+ tarefas independentes esperando?
   ├─ não → delegação única, sequencial.
   └─ sim → aplique o teste de independência (3 perguntas do SKILL.md).
            Um "não" em qualquer uma → sequencial.
```

---

## Contexto 1 — Feature nova em módulo isolado

**Situação:** implementar uma feature que vive num módulo próprio, com spec já escrita.

**Forma:** delegação única, sem paralelismo.

```
planner (se não houver spec)  →  builder  →  reviewer (se área sensível)  →  scribe
```

**Onde economizar:** o `builder` recebe a lista exata de arquivos e os caminhos das convenções. Não
mande "leia a documentação do projeto" — mande "leia a seção X do arquivo Y".

**Zona de contenção:** o `builder` **descreve** o wiring necessário; você aplica. Isso vale mesmo
sem paralelismo — mantém o hábito e evita que o agente decida sozinho a ordem de registro de rotas.

---

## Contexto 2 — Duas features independentes ao mesmo tempo

**Situação:** duas features em módulos diferentes, sem dependência entre si.

**Forma:** fan-out de 2 agentes `builder`, integração serializada.

**Antes de disparar:**

1. Teste de independência (módulos distintos? contenção intocada? sem dependência de resultado?)
2. Reserva de numeração — cada agente recebe explicitamente o seu número de migration/ADR
3. Allowlist de arquivos em cada briefing

**Depois:**

1. Integra entrega A: wiring → migration → suíte completa
2. Só então integra entrega B: wiring → migration → suíte completa
3. Portão de revisão para o que tocou área sensível

**Erro clássico aqui:** achar que "módulos diferentes" basta. Se as duas criam migration, elas
colidem no número. Se as duas expõem rota, colidem no arquivo de roteamento. Módulo diferente
resolve o *código*, não os recursos compartilhados.

---

## Contexto 3 — Correção em área sensível

**Situação:** bug ou ajuste em autenticação, autorização, criptografia, ou caminho que move
recurso de valor.

**Forma:** sequencial, com portão obrigatório e segregação de funções.

```
builder (corrige)  →  reviewer (invocação SEPARADA)  →  registro de decisão
```

**Não negociável:** o agente que corrigiu **não** revisa. Ele carrega o viés de quem já decidiu que
a solução está certa. Em domínio regulado isso é requisito formal, não preferência.

**Registro:** correção de segurança sempre vira entrada no log de decisões — o quê, por quê, e qual
era a exploração possível. Quem for mexer nesse código daqui a seis meses precisa dessa informação.

---

## Contexto 4 — Refatoração ampla

**Situação:** renomear conceito, extrair módulo, trocar biblioteca — mudanças que atravessam o
repositório inteiro.

**Forma:** **não paralelize.** Refatoração ampla é a definição de "tudo é zona de contenção".

Delegação única, ou trabalho direto. Se o volume for grande demais para uma sessão, quebre em
etapas **sequenciais** com suíte verde entre cada uma — não em agentes simultâneos.

**Por quê:** dois agentes refatorando o mesmo conceito em arquivos diferentes produzem convenções
divergentes. O custo de reconciliar supera qualquer ganho de tempo.

---

## Contexto 5 — Investigação de bug

**Situação:** algo está quebrado e você não sabe por quê.

**Forma:** depende de quantas frentes independentes existem.

- **Uma causa suspeita** → trabalhe direto ou use um agente de exploração. Não fragmente: a
  investigação precisa de visão do todo.
- **Falhas claramente independentes** (módulos diferentes, causas diferentes) → um agente por
  frente, em paralelo. É o caso clássico de fan-out.

**Cuidado:** falhas que *parecem* independentes mas têm causa comum. Se três testes quebraram
depois do mesmo commit, provavelmente é **uma** causa — três agentes vão encontrar a mesma coisa
três vezes, e você paga três vezes.

---

## Contexto 6 — Documentação atrasada

**Situação:** o código andou e a documentação ficou para trás.

**Forma:** `scribe`, modelo barato, em paralelo com outro trabalho — documentação não conflita com
implementação em módulo diferente.

**Requisito:** o `scribe` só documenta o que **já foi verificado**. Passe a evidência junto. Se você
não sabe como algo foi testado, o `scribe` não pode escrever que foi — e você acabou de descobrir
uma lacuna de verificação, não de documentação.

---

## Contexto 7 — Monólito legado sem fronteira clara

**Situação:** repositório antigo, sem módulos bem definidos, tudo importa tudo.

**Forma:** paralelismo é perigoso aqui. Comece **sem** ele.

**O que fazer primeiro:** descubra as zonas de contenção pelo histórico:

```bash
git log --format="" --name-only -n 200 | sort | uniq -c | sort -rn | head -20
```

Se os 10 arquivos mais alterados aparecem em commits de features não relacionadas, você não tem
fronteiras — tem um monólito acoplado. Nesse caso:

1. Use delegação **única** com allowlist rígida (é o principal valor da skill aqui)
2. Cada entrega que criar fronteira nova vira zona paralelizável no manifesto
3. Paralelize só quando houver dois módulos com fronteira real

A skill continua útil sem paralelismo: allowlist, portão de revisão e orçamento de tokens valem
sozinhos.

---

## Contexto 8 — Time pequeno, orçamento apertado

**Situação:** custo de token importa mais que velocidade.

**Forma:** delegação seletiva, modelo proporcional.

**Onde o dinheiro vai embora:**

| Desperdício | Custo | Correção |
|---|---|---|
| Agente explorando repositório | Alto | Allowlist + caminhos exatos no briefing |
| Modelo caro em tarefa mecânica | Alto | `scribe` em modelo barato para documentação |
| Spawn frio repetido na mesma linha | Médio | Continue o agente existente em vez de criar outro |
| Fan-out que colide | Muito alto | Teste de independência honesto |
| Relatório verboso repetindo a spec | Baixo, mas constante | Peça relatório comprimido |

**Regra de bolso:** se a tarefa cabe em três chamadas de ferramenta, o spawn custa mais que fazer
direto.

---

## Antipadrões

| Antipadrão | Por que falha |
|---|---|
| **Paralelizar por padrão** | A maioria das tarefas tem dependência escondida; o retrabalho come o ganho |
| **Agente que revisa o próprio código** | Viés de confirmação; e em domínio regulado, violação de conformidade |
| **Briefing vago ("implemente a feature X")** | O agente explora, gasta orçamento e entrega fora do escopo |
| **Confiar em número sugerido pela spec** | Specs envelhecem; migration/ADR já pode ter sido usada |
| **Integrar tudo e rodar a suíte no fim** | Quebrou — mas qual das cinco entregas foi? |
| **Aceitar "os testes passaram" sem output** | Alegação não é evidência |
| **Deixar decisão do agente só no relatório** | Daqui a três meses ninguém sabe por que o código é assim |
| **5+ agentes simultâneos** | O tempo de integração e rastreio supera o ganho |
| **Delegar exploração vaga** | Você paga para o agente descobrir o que você deveria ter delimitado |
