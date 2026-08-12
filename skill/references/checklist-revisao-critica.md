# Checklist do portão de revisão independente

Usado pelo perfil `reviewer`. Ler apenas quando a mudança toca área declarada como sensível no
manifesto do projeto.

**Cada item exige resposta com `arquivo:linha`, não com "parece ok".** Revisão que não aponta
localização não é revisão, é opinião.

---

## Núcleo — vale para qualquer domínio

### Autenticação e autorização
- [ ] Toda operação que lê ou altera dado de um titular exige credencial válida?
- [ ] O identificador do titular usado no filtro vem **da credencial**, nunca da URL ou do corpo da requisição?
- [ ] Ownership do recurso é verificado contra o dono real, não presumido pelo caminho da rota?
- [ ] Credencial é armazenada como hash, nunca em texto claro?
- [ ] Revogação tem efeito imediato no caminho de autenticação?

### Isolamento entre titulares (multi-tenant)
- [ ] Toda query filtra pelo identificador do tenant — sem exceção, inclusive em relatórios e agregados?
- [ ] Existe teste que prova explicitamente "tenant A não enxerga dado de B"?
- [ ] Nenhum endpoint devolve agregado cruzando tenants?

### Injeção e validação
- [ ] Toda query usa parâmetros vinculados — zero concatenação de string em SQL/NoSQL/comando?
- [ ] Entrada que vira identificador é validada antes do uso?
- [ ] Entrada que vira caminho de arquivo, URL ou comando é sanitizada?

### Segredos e exposição
- [ ] Nenhum token, chave ou senha commitado?
- [ ] Nenhum segredo em log, nem em nível debug?
- [ ] Mensagem de erro ao cliente não vaza estrutura interna, query ou stack trace?

### Resiliência de integração externa
- [ ] Falha de terceiro **nunca** desfaz operação já confirmada do usuário?
- [ ] Há timeout explícito em toda chamada de saída?
- [ ] Falha deixa rastro reprocessável (estado + motivo persistidos), não só log?
- [ ] Erro temporário e permanente têm tratamento distinto?

### Concorrência e idempotência
- [ ] Idempotência é garantida por **constraint no banco**, não só por checagem em código?
      (dois processos concorrentes passam pela checagem lógica ao mesmo tempo)
- [ ] Processo que disputa linhas usa bloqueio adequado (`SELECT FOR UPDATE SKIP LOCKED` ou equivalente)?
- [ ] Reprocessamento de evento/webhook não duplica efeito colateral?

### Trilha de auditoria
- [ ] Operação bloqueada por regra crítica deixa registro imutável?
- [ ] Registro de auditoria é insert-only — sem update, sem delete, sem endpoint de edição?

---

## Perfil: financeiro / pagamentos

- [ ] Zero ponto flutuante em campo monetário — busca por `float`/`double` no diff volta vazia?
- [ ] Unidade consistente em cada camada (menor unidade no banco, tipo decimal no domínio, string formatada na API)?
- [ ] Toda decomposição de valor fecha a soma; sobra de arredondamento tem destino definido e único?
- [ ] Arredondamento é explícito, não efeito colateral de serialização?
- [ ] Operação de reversão é proporcional — reversão parcial não cancela o total?
- [ ] Nenhum caminho reverte valor já liquidado/transferido para terceiro sem bloqueio explícito?
- [ ] Limites e validações de valor acontecem antes de chamar o provedor externo?

## Perfil: dado pessoal / regulado (LGPD, GDPR, HIPAA)

- [ ] Dado sensível é minimizado — só é coletado e trafegado o que a operação exige?
- [ ] Dado sensível bruto é bloqueado na borda, antes de chegar ao domínio?
- [ ] Há caminho de exclusão/anonimização que realmente remove o dado dos rastros?
- [ ] Log e telemetria não carregam dado identificável?
- [ ] Retenção tem prazo definido e mecanismo que o cumpre?

## Perfil: cartão / PCI DSS

- [ ] Dados de cartão (PAN, CVV) bloqueados na borda do handler, antes de qualquer desserialização de domínio?
- [ ] Nenhum dado de cartão em log, erro ou payload de saída?
- [ ] Sistema só trabalha com token do provedor, nunca com o número em si?

## Perfil: assinatura e verificação de mensagem (webhooks)

- [ ] Assinatura validada **antes** de qualquer efeito colateral?
- [ ] Comparação em tempo constante, nunca igualdade de string?
- [ ] Corpo usado no cálculo é o bruto recebido, não um re-serializado?
- [ ] Segredo de produção separado do de desenvolvimento, vindo de variável de ambiente?
- [ ] Mensagem repetida é idempotente?

---

## Formato do achado

```
ARQUIVO:LINHA
Problema: <o que está errado, em uma frase>
Exploração: <entrada concreta → efeito concreto>
Severidade: crítica | alta | média | baixa
```

**Crítica** = permite mover recurso de terceiro, obter acesso a dado de outro titular, ou expor
credencial/dado regulado. Crítica bloqueia a entrega; o resto é decisão de quem coordena.

Não proponha refatoração ampla. Aponte o problema e o menor caminho de correção — quem corrige é
o perfil `builder`, em invocação separada (segregação de funções).
