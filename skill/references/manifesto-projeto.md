# Manifesto multi-agente do projeto — template

Copie para `.claude/multiagente.md` no repositório e preencha. Sem este arquivo preenchido,
não paralelize: você não sabe onde estão os pontos de colisão.

Mantenha-o curto. Ele é lido em toda delegação — cada linha inútil custa tokens em todas as
sessões futuras.

---

```markdown
# Manifesto multi-agente — <NOME DO PROJETO>

**Criticidade:** Crítico | Alto | Padrão
**Atualizado:** <data>

## Comandos de verificação

Todo agente que entrega implementação precisa colar a saída real destes comandos:

- Build: `<comando>`
- Lint/vet: `<comando>`
- Teste: `<comando>`
- Teste end-to-end (se houver): `<comando + pré-requisitos>`

## Zonas de contenção

Arquivos que toda mudança toca. **Exclusivos do orquestrador** — subagentes descrevem o que
precisa mudar, não editam:

| Arquivo | Por quê |
|---|---|
| `<caminho do wiring / injeção de dependência>` | Toda feature registra aqui |
| `<caminho do roteamento>` | Toda rota registra aqui |
| `<diretório de migrations>` | Numeração colide |
| `<docs de índice / changelog / log de decisões>` | Append por entrega |
| `<mocks e fixtures compartilhados>` | Dois agentes editando a mesma struct |

## Recursos numerados compartilhados

Reservar antes de qualquer fan-out. Comandos para descobrir o próximo livre:

```bash
<comando que mostra a última migration>
<comando que mostra o último ADR / registro de decisão>
<comando que mostra o último ID de tarefa>
```

> A fonte da verdade é o repositório, não o documento de planejamento.

## Zonas paralelizáveis

Dois agentes podem trabalhar ao mesmo tempo se cada um ficar inteiramente dentro de:

- `<módulos/pacotes/serviços independentes entre si>`
- `<arquivos de spec/teste distintos>`

## Áreas sensíveis — revisão independente obrigatória

Qualquer mudança que toque isto exige o portão de revisão, em invocação separada:

- `<autenticação / autorização>`
- `<criptografia, assinatura, validação de webhook>`
- `<qualquer código que mova dinheiro, altere saldo ou conceda acesso>`
- `<dado pessoal / regulado>`
- `<isolamento entre tenants/clientes>`

## Invariantes do domínio

Regras que a revisão precisa verificar explicitamente. Sejam concretas — invariante vago não
é verificável:

- `<ex.: nenhum valor monetário em ponto flutuante>`
- `<ex.: idempotência garantida por constraint de banco, não só por lógica>`
- `<ex.: toda query filtra por tenant_id>`

## Perfis de agente disponíveis

| Perfil | Arquivo | Modelo | Uso |
|---|---|---|---|
| builder | `.claude/agents/<nome>.md` | | |
| reviewer | `.claude/agents/<nome>.md` | | |
| planner | `.claude/agents/<nome>.md` | | |
| scribe | `.claude/agents/<nome>.md` | | |

## Convenções que todo agente precisa conhecer

Aponte o arquivo, não repita o conteúdo:

- Convenções gerais: `<CLAUDE.md ou equivalente>`
- Padrão de arquitetura: `<doc>`
- Processo de spec/planejamento: `<doc>`
```

---

## Como descobrir suas zonas de contenção

Se você não sabe quais arquivos toda mudança toca, o histórico do git responde:

```bash
# arquivos mais alterados nos últimos 100 commits
git log --format="" --name-only -n 100 | sort | uniq -c | sort -rn | head -20
```

Os do topo são candidatos a zona de contenção — especialmente se aparecerem em commits de
features não relacionadas entre si. Wiring, roteamento, changelog e arquivos de índice
costumam liderar.

Confirme olhando **por que** aparecem: um arquivo alterado sempre pelo mesmo motivo (append de
registro, registro de rota, numeração) é zona de contenção. Um arquivo grande que muda por
motivos variados é só um arquivo grande — pode ser paralelizável se as mudanças forem em regiões
distintas, mas trate com cuidado.
