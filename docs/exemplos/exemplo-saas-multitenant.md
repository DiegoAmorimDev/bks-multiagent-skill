# Exemplo — SaaS B2B multi-tenant (criticidade: Alto)

Aplicação Node/TypeScript, Prisma, multi-tenant por coluna `tenantId`. Sem dinheiro em trânsito,
mas com dado de cliente empresarial e SLA contratual.

---

## Manifesto (trecho preenchido)

```markdown
# Manifesto multi-agente — plataforma-saas

**Criticidade:** Alto

## Comandos de verificação
- Build: `npm run build`
- Lint: `npm run lint`
- Teste: `npm test -- --run`
- E2E: `npm run test:e2e` (exige `docker compose up -d` antes)

## Zonas de contenção
| Arquivo | Por quê |
|---|---|
| `src/routes/index.ts` | Toda rota registra aqui |
| `src/container.ts` | Injeção de dependência |
| `prisma/migrations/` | Nome com timestamp — colisão rara, mas o schema.prisma é único |
| `prisma/schema.prisma` | **Arquivo único para todos os modelos** — sério ponto de contenção |
| `CHANGELOG.md`, `docs/adr/` | Append e numeração |
| `src/test/fixtures.ts` | Fixtures compartilhadas |

## Recursos numerados
```bash
ls prisma/migrations/ | tail -1
ls docs/adr/ | tail -1
```

## Zonas paralelizáveis
- `src/modules/<modulo>/` de módulos diferentes
- testes de módulos diferentes

## Áreas sensíveis
- `src/auth/**` — sessão, JWT, refresh token
- qualquer query sem `tenantId` no `where`
- `src/billing/**` — cotas e limites de plano
- exportação de dados (LGPD)

## Invariantes
- Toda query Prisma tem `tenantId` no `where` — vindo do contexto da sessão, nunca do body
- Nenhum endpoint retorna agregado cruzando tenants
- Exportação de dados registra quem exportou, o quê e quando
```

---

## O detalhe que muda tudo neste stack: `schema.prisma`

Em projetos Prisma (ou qualquer ORM com schema centralizado), **o arquivo de schema é uma zona de
contenção brutal**: dois agentes adicionando modelos diferentes editam o mesmo arquivo, e o
`prisma migrate` gera migrations divergentes.

**Consequência prática:** duas features que precisam de modelo novo **não paralelizam**, mesmo
estando em módulos diferentes.

**Contorno possível:** o orquestrador aplica todas as mudanças de schema **antes** do fan-out,
gera a migration única, e só então dispara os agentes para implementar a lógica sobre o schema já
pronto. Isso transforma um caso não-paralelizável em paralelizável, ao custo de uma etapa
sequencial no início.

```
1. orquestrador: adiciona modelos A e B ao schema.prisma, gera migration única
2. fan-out: agente A implementa lógica sobre modelo A; agente B sobre modelo B
3. integração serializada: rotas e container, um de cada vez
```

---

## Rodada — duas features com o contorno aplicado

**Tarefas:** exportação de relatórios (módulo `reports`) e webhooks de saída (módulo `webhooks`).

### Etapa sequencial (orquestrador)

```prisma
model ReportExport { id String @id @default(cuid()) tenantId String ... }
model WebhookDelivery { id String @id @default(cuid()) tenantId String ... }
```

Uma migration, aplicada. Agora o schema está estável para os dois agentes.

### Fan-out

| Agente | Allowlist | Reservas |
|---|---|---|
| A | `src/modules/reports/**`, `src/modules/reports/**/*.test.ts` | ADR 012 |
| B | `src/modules/webhooks/**`, testes do módulo | ADR 013 |

Ambos com denylist: `schema.prisma`, `routes/index.ts`, `container.ts`, `fixtures.ts`.

### Portão de revisão

Ambas as features tocam área sensível — **toda query nova precisa de `tenantId`**. Perfil de
revisão aplicado: núcleo + isolamento entre titulares + dado pessoal (LGPD, por causa da
exportação).

Achado típico neste contexto:

```
src/modules/reports/report.service.ts:64
Problema: findMany sem tenantId no where — o filtro está só na camada de rota
Exploração: qualquer chamada interna que reutilize o service vaza dados entre tenants
Severidade: crítica
```

**Por que o teste não pegou:** o teste passava `tenantId` corretamente pela rota. A falha só aparece
quando o service é chamado de outro lugar — exatamente o que aconteceria na próxima feature.
Defesa em profundidade: o filtro precisa estar no service, não só na borda.

---

## Adaptação do orçamento de tokens neste contexto

| Situação | Ajuste |
|---|---|
| Monorepo grande, `tsconfig` com muitos paths | Aponte o módulo exato; nunca "leia o src" |
| Testes lentos (e2e com docker) | Agente roda só o teste do módulo dele; suíte completa é do orquestrador na integração |
| Muitos arquivos de tipo gerado | Coloque na denylist — agente não deve editar código gerado |

---

## Quando **não** paralelizar neste stack

- Qualquer mudança em `schema.prisma` sem a etapa sequencial prévia
- Mudança em middleware de autenticação (afeta todas as rotas)
- Alteração de contrato de API consumido por outro módulo
- Upgrade de dependência maior
