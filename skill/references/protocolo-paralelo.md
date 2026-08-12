# Protocolo operacional de fan-out paralelo

Ler quando for disparar 2+ agentes simultâneos. Para delegação única sequencial, o `SKILL.md` basta.

---

## 1. Teste de independência

Três perguntas. Um "não" = sequencial.

1. **Módulos distintos?**
2. **Zona de contenção intocada por ambos?**
3. **Sem dependência de resultado entre as tarefas?**

Se você precisou pensar mais de trinta segundos em alguma delas, a resposta prática é: sequencial.
Independência real é óbvia.

## 2. Reserva de recursos compartilhados

Rode os comandos de descoberta declarados no manifesto e distribua explicitamente:

```
agente A → migration 015, ADR 024
agente B → migration 016, ADR 025
```

Quem não precisa de recurso numerado recebe "nenhuma reserva necessária" — explicitamente. Agente
sem instrução inventa.

## 3. Disparo

Múltiplas chamadas de agente na **mesma** mensagem, em background. Cada uma com o template de
briefing preenchido, incluindo a allowlist de arquivos.

**Limite prático: 2 a 3 simultâneos.** Acima disso o custo de integrar e rastrear conflito supera
o ganho, e fica difícil saber qual agente causou qual quebra.

## 4. Integração serializada

Os agentes entregam dentro dos próprios módulos. O orquestrador aplica, **um de cada vez**, na
ordem de chegada:

1. Wiring / registro de rota / injeção de dependência
2. Migration reservada, no ambiente local
3. Build + lint + suíte completa
4. Só então a próxima entrega

Rodar a suíte após cada integração é o que torna óbvio qual entrega quebrou o quê. Investigar
depois de integrar tudo custa mais que as execuções extras.

### Verificação isolada por commit — cuidado com `git stash`

Ao organizar um monte de código já pronto (não commitado) em vários commits de
milestone, a tentação é: stage os arquivos do commit N, `git stash push --keep-index`
o resto para testar só o diff de N em isolamento, commit, `git stash pop`, repete
para N+1.

**Isso já causou perda de dado real.** Num ciclo com várias repetições desse
padrão — stage, stash, pop-para-inspecionar, stage de mais um arquivo, stash de
novo, unstage para o próximo passo — dois arquivos centrais (`main.go` e
`router.go`, ~120 linhas de trabalho da sessão) voltaram silenciosamente ao estado
do commit anterior à sessão. **Sem erro, sem conflito, `git stash pop` reportando
sucesso.** Só foi percebido porque um `git diff HEAD -- main.go` posterior voltou
vazio quando não devia.

**Não use esse padrão.** Alternativas seguras:

- **Verificação contra a árvore de trabalho completa**, em vez de isolada por
  commit — build/test são baratos de rodar de novo; a prova fica menos "isolada"
  por commit, mas o risco de perda cai a zero.
- **`git worktree add`** para isolamento de verdade, quando o commit realmente
  precisa ser testado sozinho — não compartilha o stash/index da árvore principal,
  então não tem esse modo de falha.

Se um stash sumir mesmo assim: `git fsck --unreachable --no-reflog | grep commit`
lista commits órfãos ainda alcançáveis (git não coleta lixo imediatamente), e
`git show <sha>:<caminho> > <caminho>` recupera um arquivo específico de dentro
dele.

## 5. Portão de revisão

Para cada entrega que tocou área sensível, dispare o `reviewer` em invocação separada, com o
escopo daquela entrega. Nunca peça revisão ao agente que escreveu o código.

Pode rodar em paralelo com a integração da entrega seguinte — revisão é read-only, não conflita.

## 6. Consolidação

Depois de tudo integrado e revisado:

1. Verificação de ponta a ponta, no padrão que o projeto exige (unitário raramente basta)
2. Registrar decisões que os agentes reportaram como desvio de spec
3. Atualizar specs/tickets para o estado real
4. Documentação — pode ser delegada ao `scribe`, passando o que foi feito **e como foi verificado**
5. Relatório: o que cada agente entregou, evidência, o que ficou pendente

## Recuperação de conflito

Dois agentes editaram o mesmo arquivo apesar do protocolo:

1. **Não mescle automaticamente** código de área crítica — leia as duas versões
2. Identifique qual violou a allowlist: foi erro de briefing (você) ou de execução (o agente)?
3. Mantenha a entrega que respeitou o escopo; reaplique a outra manualmente por cima
4. Rode a suíte completa antes de seguir
5. Se o conflito foi em migration já aplicada: **verifique o schema real do banco** antes de
   assumir qual versão venceu — o arquivo e o banco podem ter divergido

## Modo degradado

Quando um agente falha, trava ou entrega fora do escopo:

- **Não reaproveite entrega parcial de área crítica.** Descarte e refaça com briefing corrigido.
  Código pela metade em caminho que move recurso é pior que nenhum código.
- **Entrega fora da allowlist** é sinal de briefing vago, não de agente ruim. Corrija o briefing.
- **Dois agentes com relatórios contraditórios** sobre o mesmo estado: pare o fan-out, verifique o
  estado real no repositório, e siga sequencial.

## Métricas de que o paralelismo está valendo a pena

Ao final de cada rodada, confira:

- Tempo de integração < tempo somado de implementação dos agentes?
- Zero conflito de arquivo?
- Suíte verde no primeiro `run` após cada integração?
- Nenhum agente precisou ser refeito por briefing incompleto?

Duas respostas "não" na mesma rodada: volte ao sequencial e reveja o manifesto — provavelmente
falta declarar alguma zona de contenção.
