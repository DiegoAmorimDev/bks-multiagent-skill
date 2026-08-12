# Exemplo completo — gateway de pagamento (criticidade: Crítico)

Caso real anonimizado: gateway com split de receita, repasses agendados e integração fiscal.
Arquitetura hexagonal, módulos por Bounded Context, PostgreSQL, migrations numeradas.

---

## Manifesto (trecho preenchido)

```markdown
# Manifesto multi-agente — gateway-pagamentos

**Criticidade:** Crítico

## Comandos de verificação
- Build: `go build ./...`
- Lint: `go vet ./...`
- Teste: `go test ./... -count=1`
- E2E: API com stubs (`USE_STUB=true go run ./cmd/api/`) + PostgreSQL local

## Zonas de contenção
| Arquivo | Por quê |
|---|---|
| `cmd/api/main.go` | Todo wiring de dependência passa aqui |
| `internal/api/router.go` | Toda rota registra aqui |
| `db/migrations/NNN_*.sql` | Numeração colide |
| `docs/CONTEXTO.md`, `docs/API.md`, `docs/GUIA_TESTE.md` | Append por feature |
| `docs/DECISOES.md` | Numeração DNN colide |
| mocks compartilhados por pacote de teste | Dois agentes editando a mesma struct |

## Recursos numerados
```bash
ls db/migrations/ | tail -1
grep -o "^### D[0-9]*" docs/DECISOES.md | sort -V | tail -1
ls spec/tasks/ | tail -1
```

## Zonas paralelizáveis
- `internal/<contexto>/domain|ports|usecase|adapters` de contextos **diferentes**
- arquivos de spec distintos

## Áreas sensíveis
- autenticação e API keys (`internal/identidade/`, `internal/api/middleware/`)
- validação HMAC de webhook (`/webhooks/*`)
- caminho de cartão (barreira PCI na borda do handler)
- qualquer código que mova dinheiro: split, repasse, estorno, retenção fiscal
- isolamento entre clientes (qualquer query com `cliente_id`)

## Invariantes
- Nenhum valor monetário em `float64` — sempre `decimal.Decimal`
- Centavos no banco, decimal no domínio, string formatada na API
- Idempotência por constraint UNIQUE, nunca só por checagem em código
- Soma das parcelas fecha com o bruto; sobra de arredondamento no último recebedor
- Estorno nunca reverte repasse já liquidado
```

---

## Rodada 1 — duas features independentes

**Tarefas:** F-A (extrato financeiro, novo módulo de consulta) e F-B (dashboard operacional, outro
módulo de consulta).

### Teste de independência

| Pergunta | Resposta |
|---|---|
| Módulos distintos? | Sim — `internal/relatorio/` e `internal/operacao/` |
| Zona de contenção intocada? | **Não** — ambas expõem rota nova e F-B precisa de migration |
| Sem dependência de resultado? | Sim |

Uma resposta "não" → não é fan-out livre. **Solução:** paraleliza a implementação, serializa a
integração. Os agentes entregam o código dos módulos e *descrevem* a rota; o orquestrador registra.

### Reserva

```
agente A (extrato)   → nenhuma migration necessária
agente B (dashboard) → migration 013, decisão D22
```

Confirmado no repositório: última migration `012`, última decisão `D21`. A spec da F-B dizia
"migration 009" — desatualizada, ignorada.

### Briefing do agente B (resumido)

```
ESCOPO: dashboard operacional — sumário por status, próximos 7 dias, itens falhos

PODE TOCAR:
- internal/operacao/**
- db/migrations/013_adiciona_motivo_falha.sql

NÃO TOQUE:
- cmd/api/main.go, internal/api/router.go, docs/*.md, docs/DECISOES.md

CONTEXTO MÍNIMO:
- CLAUDE.md (convenções de dinheiro e erro)
- spec/features/FEAT-dashboard.md §2 e §3
- internal/relatorio/query/extrato_query.go (padrão de query a seguir)

RESERVAS: migration 013, decisão D22

ENTREGÁVEL: código + testes (cole saída de build/vet/test) + descrição do wiring e da rota

RELATÓRIO: comprimido.
```

### Integração

1. Entrega A: wiring → suíte completa → verde
2. Entrega B: wiring → aplica migration 013 → suíte completa → verde
3. Portão de revisão: **A e B tocam consulta com `cliente_id`** → área sensível (isolamento) →
   `reviewer` em invocação separada, para cada uma

### Resultado da revisão

O `reviewer` apontou em B:

```
internal/operacao/query/dashboard_query.go:88
Problema: query de agregação não filtra por cliente_id
Exploração: qualquer cliente autenticado vê o somatório de todos os clientes
Severidade: crítica
```

Correção feita pelo `builder` em invocação separada. Registrada em `DECISOES.md`.

> Este é exatamente o tipo de falha que a segregação de funções existe para pegar: o agente que
> escreveu a query "sabia" que o filtro estava lá, porque ele o adicionou nas outras três queries
> do mesmo arquivo.

---

## Rodada 2 — o que deu errado quando ignoramos o protocolo

**Tentativa:** três agentes em paralelo, sem reserva de numeração.

**Resultado:** dois escolheram `migration 014`. O terceiro editou `main.go` "só para testar se
compilava" — fora da allowlist.

**Custo:** desfazer duas integrações, reaplicar uma manualmente, e uma sessão inteira de
investigação porque a suíte quebrou por motivo que nenhum dos três reconhecia.

**Lição incorporada ao protocolo:** reserva **antes** do disparo, allowlist explícita no briefing,
e suíte completa após **cada** integração — não só no fim.

---

## Números da rodada bem executada

| Métrica | Resultado |
|---|---|
| Agentes simultâneos | 2 |
| Conflitos de arquivo | 0 |
| Suíte verde no primeiro run após cada integração | Sim |
| Achados críticos no portão de revisão | 1 (isolamento entre clientes) |
| Retrabalho por briefing incompleto | 0 |
