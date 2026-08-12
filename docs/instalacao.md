# Instalação e configuração

---

## 1. Instalar a skill

**Skill de projeto** (vale só neste repositório, versionada junto com o código — recomendado para
time):

```bash
mkdir -p .claude/skills/bks-multiagent-skill
cp -r skill/* .claude/skills/bks-multiagent-skill/
```

**Skill de usuário** (vale em todos os seus projetos, não versionada):

```bash
mkdir -p ~/.claude/skills/bks-multiagent-skill
cp -r skill/* ~/.claude/skills/bks-multiagent-skill/
```

Confirme que o Claude Code enxergou a skill: ela deve aparecer na lista de skills disponíveis com
o nome `bks-multiagent-skill`.

---

## 2. Preencher o manifesto (obrigatório)

```bash
cp skill/references/manifesto-projeto.md .claude/multiagente.md
```

O manifesto é o que torna a skill específica do seu repositório. **Sem ele preenchido, não
paralelize** — a skill não tem como saber onde estão seus pontos de colisão.

### Como descobrir suas zonas de contenção

```bash
git log --format="" --name-only -n 100 | sort | uniq -c | sort -rn | head -20
```

Os arquivos do topo são candidatos. Confirme olhando **por quê** aparecem tanto:

| Padrão observado | É zona de contenção? |
|---|---|
| Sempre pelo mesmo motivo: append de registro, registro de rota, numeração | **Sim** |
| Arquivo grande que muda por motivos variados, em regiões distintas | Provavelmente não — mas trate com cuidado |
| Arquivo de configuração alterado a cada deploy | Sim, se agentes puderem tocá-lo |

Suspeitos frequentes por stack:

| Stack | Zonas de contenção típicas |
|---|---|
| Go / Java / C# | arquivo de wiring/DI, registro de rotas, diretório de migrations |
| Node / TypeScript | `index.ts` de barril, arquivo de rotas, `package.json` |
| Rails / Django | `routes.rb` / `urls.py`, diretório de migrations, initializers |
| Qualquer um | `CHANGELOG.md`, log de decisões/ADRs, docs de índice, mocks compartilhados |

### Preencher as áreas sensíveis

Liste o que exige revisão independente. Seja concreto — "código importante" não é acionável.
Exemplos por domínio:

- **Financeiro:** cálculo de valor, transferência, estorno, conciliação, retenção tributária
- **Saúde:** prontuário, prescrição, consentimento, anonimização
- **Identidade:** autenticação, sessão, recuperação de senha, MFA, permissões
- **Qualquer SaaS:** isolamento entre clientes, cotas, faturamento

### Declarar os invariantes

Regras verificáveis que a revisão precisa conferir. Invariante vago não serve. Compare:

| Ruim | Bom |
|---|---|
| "Cuidado com dinheiro" | "Nenhum valor monetário em ponto flutuante" |
| "Garantir idempotência" | "Idempotência por constraint UNIQUE no banco, não só por checagem em código" |
| "Respeitar multi-tenant" | "Toda query filtra por `tenant_id` vindo do token, nunca da URL" |

---

## 3. Instalar os perfis de agente (opcional, recomendado)

```bash
mkdir -p .claude/agents
cp skill/templates/agents/*.md .claude/agents/
```

Ajuste em cada arquivo:

1. **`name`** — prefixe com o nome do projeto (`meuprojeto-builder`) se você tiver vários repos
2. **`model`** — o modelo que você quer fixar para aquele perfil
3. **`description`** — quando delegar para ele; é isso que o orquestrador lê para decidir
4. **Corpo** — substitua os `<placeholders>` pelos caminhos reais das suas convenções

> Os perfis fixam **modelo por tipo de tarefa**. É o principal mecanismo de controle de custo:
> revisão de segurança em modelo forte, sincronização de documentação em modelo barato.

---

## 4. Integrar com o processo existente

A skill governa **como delegar**. Ela não substitui seu processo de planejamento.

Se o seu repositório tem um arquivo de convenções lido em toda sessão (`CLAUDE.md` ou equivalente),
adicione um bloco apontando para a skill e para os perfis — assim a orquestração é descoberta sem
precisar ser pedida:

```markdown
## Orquestração multi-agente

Delegação e paralelismo neste repositório seguem a skill `bks-multiagent-skill`.
Manifesto: `.claude/multiagente.md`. Perfis: `.claude/agents/`.

Ordem dentro de uma entrega: planner → builder → reviewer → scribe.
Zona de contenção é exclusiva do orquestrador — subagentes descrevem, não aplicam.
```

**Conflito com outras skills:** se o seu projeto já usa alguma skill de agentes paralelos que
escreve o próprio plano, decida qual manda. Dois sistemas de plano no mesmo repositório é fonte
garantida de confusão. Esta skill foi desenhada para **não** ter plano próprio justamente para
conviver com o processo que você já tem.

---

## 5. Verificar a configuração

Antes do primeiro fan-out:

- [ ] Skill aparece na lista de skills disponíveis
- [ ] `.claude/multiagente.md` existe e está preenchido (sem `<placeholders>` sobrando)
- [ ] Zonas de contenção conferidas contra o histórico do git
- [ ] Comandos de build/lint/teste do manifesto realmente rodam
- [ ] Comandos de descoberta de numeração retornam o valor certo
- [ ] Perfis de agente com modelo definido e caminhos reais
- [ ] Áreas sensíveis listadas de forma concreta
- [ ] Ao menos um invariante verificável declarado
