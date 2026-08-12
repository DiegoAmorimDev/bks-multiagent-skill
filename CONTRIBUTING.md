# Como contribuir

Esta skill evolui a partir de **uso real**. Relato de algo que deu errado na prática vale mais que
sugestão teórica.

---

## O que é mais valioso

| Tipo de contribuição | Por que importa |
|---|---|
| **Zona de contenção descoberta na prática** | Cada stack tem as suas; a lista atual cobre o que já vimos |
| **Invariante que a revisão deveria ter pego e não pegou** | Falha do checklist é a falha mais cara |
| **Antipadrão observado** | Alguém vai repetir; documentar evita |
| **Perfil de domínio novo no checklist** | Hoje há financeiro, dado regulado, PCI e assinatura |
| **Exemplo em stack não coberta** | Rails, Django, .NET, Java — cada uma tem armadilhas próprias |

## O que provavelmente será recusado

- Automação que **infere** zonas de contenção sem confirmação humana — erro aqui corrompe estado
- Aumento do limite de agentes simultâneos sem dado que sustente
- Afrouxamento da segregação de funções "para ir mais rápido"
- Plano/backlog próprio dentro da skill — ela deliberadamente não tem um

---

## Fluxo de branches

```
main       ← estável, versionada com tag
  ↑
develop    ← integração
  ↑
feat/*     ← trabalho
fix/*
docs/*
```

1. Crie a branch a partir de `develop`
2. Commits no padrão [Conventional Commits](https://www.conventionalcommits.org/pt-br/)
3. PR para `develop`
4. `develop` → `main` em release, com tag e entrada no `CHANGELOG.md`

### Padrão de commit

```
feat(skill): adiciona perfil de revisão para dados de saúde
fix(docs): corrige comando de descoberta de zonas de contenção
docs(exemplos): adiciona caso Rails com ActiveRecord
```

Escopos usados: `skill`, `docs`, `templates`, `exemplos`.

---

## Critérios de qualidade do texto

A skill é lida por um agente em **toda** delegação. Cada linha custa tokens em todas as sessões
futuras de quem usa.

- **Corte o que não muda decisão.** Se a frase não altera o que o agente faz, ela não paga o custo.
- **Concreto vence abstrato.** "Nenhum valor monetário em ponto flutuante" > "cuidado com dinheiro".
- **Diga o porquê quando for contraintuitivo.** Regra sem motivo é regra que alguém vai burlar na
  primeira pressa.
- **Material extenso vai para `references/`**, carregado sob demanda — não para o `SKILL.md`.

---

## Testar uma mudança antes de propor

Não há suíte automatizada — é documentação executável por agente. Valide assim:

1. Instale a versão modificada num projeto real
2. Rode uma delegação que exercite o trecho alterado
3. Confira: o agente fez o que a mudança pretendia? Gastou mais ou menos contexto?
4. Descreva esse teste no PR — inclusive se o resultado foi parcial

PR sem relato de uso real será revisado com mais rigor, não recusado de saída.
