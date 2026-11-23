# Notas de apresentação — RPG Time Machine

> **Material para quem facilita o workshop**, não para substituir o README dos participantes.
> **Formato:** demonstração + construção por prompts + perguntas + testes. **Tempo-alvo:** 110 minutos de núcleo, com desafios opcionais se houver tempo.

## 1. Mensagem central

**Event Sourcing não significa “guardar logs”. Significa que o registro de eventos de domínio é a fonte da verdade a partir da qual o estado é reconstruído.**

O objetivo do encontro é que a turma entenda isso **por experiência própria**, não apenas por uma explicação. Eles criarão um pequeno RPG, adicionarão o histórico, viajarão no tempo, persistirão eventos e publicarão o projeto.

### Frase de abertura sugerida

> “Como saber o que aconteceu, so olhando o estado atual? Não. Hoje vamos construir uma aplicação em que o histórico não é um recurso extra: o histórico *é* o sistema.”

### Três mensagens que devem ficar claras

1. **Command não é Event.** O usuário solicita `BuySword`; a compra pode falhar. Só depois de aceita existe `SwordPurchased`.
2. **O estado é derivado.** Um reducer puro aplica os eventos em ordem de versão e chega aos atributos atuais.
3. **Imutabilidade não elimina problemas operacionais.** Persistência precisa de segurança, idempotência, concorrência e estratégias de evolução de schemas.

---

## 2. Preparação do facilitador — antes do evento

### Uma semana antes

- Testar o fluxo completo no Google AI Studio e ajustar prompts se alguma opção de UI tiver mudado.
- Confirmar que integração Firebase e publicação no Cloud Run funcionam com o tipo de conta da turma.
- Escolher uma região e projeto Cloud apropriados, confirmando limites, cotas e custos.
- Preparar um app de referência concluído e **um checkpoint de código/exportação após as etapas 3 e 4**, caso alguém fique bloqueado.
- Fazer uma execução com conta Google secundária para testar isolamento de dados.
- Verificar se a conexão do local comporta vários participantes fazendo build e deploy.
- Preparar QR code para o repositório e links oficiais.

### No dia

- Deixar o README aberto e uma janela do AI Studio pronta.
- Ter duas contas de teste para demonstrar segurança e separação de dados.
- Ter uma versão do Event Store pronta, com cerca de 12 eventos, para demonstrar o slider.
- Conferir custos e limites antes de incentivar deploys individuais.
- Avisar que certas etapas dependem de autorizações do Google e podem exigir ações manuais.

### Links de apoio

- [README do workshop](README.md)
- [AI Studio](https://aistudio.google.com/apps)
- [Firebase Console](https://console.firebase.google.com/)
- [Google Cloud Console](https://console.cloud.google.com/)
- [Codelab de inspiração](https://codelabs.developers.google.com/codelabs/cloud-run/launch-your-web-based-video-game-with-ai?hl=pt-br)

---

## 3. Agenda sugerida

| Bloco | Minutos | Marco para a turma |
| --- | ---: | --- |
| Contexto e demonstração da ideia | 0–10 | Entender CRUD × histórico de eventos |
| 1. RPG convencional | 10–25 | Personagem funcional |
| 2. Event Sourcing | 25–40 | Event Store e replay |
| 3. Time Travel | 40–50 | Estado histórico visual |
| 4. Firestore + Auth | 50–70 | Persistência e segurança |
| 5. Aggregate e regras | 70–85 | Comandos válidos × inválidos |
| 8. Revisão + publicação | 85–105 | URL funcionando |
| Discussão e perguntas | 105–110 | Recapitulação |

**Extensões opcionais (30 min):** projeções/snapshots (15 min) e laboratório de bugs (15 min). Se houver 120 minutos totais, use os 10 restantes como buffer de integração — não prometa concluir todas as extensões.

---

## 4. Roteiro de fala por etapa

### Abertura — 0 a 10 min

**Mostrar:** uma ficha de personagem (ex.: HP 70, XP 50, ouro 20, Iron Sword) sem histórico.

**Perguntar:** “Que sequência de ações produziu isso? Como eu volto ao instante em que o jogador tinha 100 HP?”

**Explicar em linguagem simples:**

- CRUD tradicional costuma persistir o estado atual; auditoria pode existir à parte.
- Event Sourcing persiste eventos que descrevem mudanças já confirmadas.
- O estado atual é calculado aplicando esses eventos; não é preciso guardar cada versão inteira.
- Não precisamos transformar todo projeto em Event Sourcing; escolhemos agregados nos quais o histórico é parte fundamental do domínio.

**Exemplo no quadro:**

```text
CharacterCreated (HP 100, XP 0, ouro 0)
TrainingCompleted (+25 XP)
BattleCompleted (-15 HP, +10 XP, +10 ouro)
CharacterRested (+15 HP efetivos)

Replay → HP 100, XP 35, ouro 10
```

**Nota:** a cura efetiva do exemplo é +15, porque o personagem estava em 85 HP e não pode passar de 100. Reforçar que o evento deve conter dados suficientes para o replay determinístico.

**Transição:** “Vamos primeiro construir da maneira tradicional para sentir a diferença.”

### Etapa 1 — 10 a 25 min: o RPG convencional

**Ação:** mandar abrir o AI Studio → Apps → New App e colar Prompt 1 do README.

**Enquanto gera:** explicar que a IA gera código, mas a pessoa continua responsável por testar regras. Mostrar visualmente o painel de atributos.

**Demonstração:** `Train` duas vezes, `Battle` uma vez, `Rest` uma vez. Perguntar como saber a causa do HP exibido.

**Pergunta ao público:** “Se amanhã aparecer um bug de ouro duplicado, como descobrimos sua origem?”

**Erro didático a evitar:** sugerir que CRUD não permite auditoria. Permite, mas exige outro mecanismo e outra modelagem.

**Checkpoint:** ao menos 3 botões funcionam e uma regra inválida é rejeitada.

### Etapa 2 — 25 a 40 min: eventos viram a verdade

**Ação:** colar Prompt 2.

**Desenhar:** `Command → Aggregate → Event → Event Store → Reducer → State`.

**Conceitos para enfatizar:**

- Event store é uma sequência lógica de fatos por agregado.
- `version` é referência principal de ordem; timestamp sozinho não resolve concorrência.
- Reducer puro: mesmo estado + mesmo evento = mesmo resultado.
- Evento significa fato ocorrido; nomes devem refletir passado (`BattleCompleted`).
- Uma ação inválida não deve gerar um evento de domínio de sucesso. Pode haver logs técnicos separados, mas eles não devem alterar o agregado.

**Demonstração:** expandir JSON de um evento e explicar `aggregateId`, `version`, `type` e `payload`.

**Pergunta ao público:** “E se eu renomear um evento amanhã? Como vou conseguir ler os eventos de hoje?”

**Resposta curta:** manter compatibilidade de schemas e, em sistemas maiores, versionar/upcast eventos.

**Checkpoint:** mostrar que os atributos da ficha são reconstruídos pelo reducer.

### Etapa 3 — 40 a 50 min: time travel

**Ação:** colar Prompt 3, gerar pelo menos 8 eventos e mover o slider.

**Momento de demonstração:** voltar para o primeiro evento, avançar um a um e apontar os atributos que mudam.

**Fala:** “Estamos consultando um estado passado — não viajando ao passado de verdade, nem apagando o que aconteceu.”

**Cuidado conceitual:** o codelab inspirador menciona restauração/rollback. **Nosso laboratório escolhe somente leitura histórica**. Para realmente desfazer algo no estado atual seria necessário um novo evento compensatório ou regras explícitas de branching.

**Pergunta:** “Se eu quiser corrigir um dano lançado errado, devo apagar o evento?”

**Resposta:** geralmente não; crie evento de correção ou compensação com semântica clara.

**Checkpoint:** passar do último ao primeiro evento não escreve nada novo.

### Etapa 4 — 50 a 70 min: Firebase e segurança

**Ação:** colar Prompt 4. Quando o AI Studio oferecer integração, autorizar. Abrir Firebase Console.

**Mostrar:** estrutura de dados aproximada:

```text
users/{uid}/characters/{id}
  events/{eventId}
  projections/current
  snapshots/{version}  # opcional
```

**Falas importantes:**

- “Firebase Authentication responde quem é o usuário. A API precisa verificar o token e se o personagem pertence a ele.”
- “Firestore Rules protegem acesso direto dos clientes. Um SDK administrativo no backend ignora essas regras; o servidor precisa fazer checagens de autorização.”
- “A projeção pode ser recalculada. O evento é o dado que preservamos.”
- “Uma transação atômica precisa confirmar a nova versão junto com o evento.”

**Demonstração ao vivo:** `Battle` → reload → mesmo histórico; abrir o documento Firestore para mostrar a persistência real.

**Aviso explícito:** NÃO corrigir `permission-denied` com regras `allow read, write: if true`.

**Checkpoint:** reload preserva eventos e outra conta não consegue acessar os mesmos dados.

**Plano B:** mostrar o app de referência e fazer os demais acompanharem o passo a passo no README, se o provisionamento estiver indisponível.

### Etapa 5 — 70 a 85 min: domínio e consistência

**Ação:** colar Prompt 5.

**Demonstração planejada:** personagem com ouro 10 tenta comprar Iron Sword de ouro 30 → erro sem evento. Depois ganha ouro suficiente e compra uma única vez.

**Distinguir:**

- Command: intenção, pode falhar.
- Aggregate: regras de negócio e invariantes.
- Event: resultado confirmado.
- Event Store: registro persistido.
- Projection: visão de leitura.

**Pergunta:** “Se duas abas tentarem gastar os mesmos 30 de ouro, qual delas ganha?”

**Resposta:** apenas a que confirmar sobre a versão válida; a outra deve receber conflito ou ser reavaliada conforme política do domínio. Verificar expectedVersion e transações; frontend não basta.

**Checkpoint:** o total de eventos não aumenta quando o comando é inválido.

### Etapa 8 — 85 a 105 min: publish

**Ação:** colar Prompt 8 para revisão. Depois usar **Publish → Get Started (se houver) → escolher projeto → Publish App**.

**Mostrar:** a URL aberta fora do editor. Fazer login, executar uma ação, recarregar e mostrar histórico.

**Falar:** “O Cloud Run está hospedando o servidor e a aplicação, enquanto o Firestore preserva o histórico. O AI Studio acelerou a construção e a publicação, mas o modelo de domínio e a segurança continuam sendo nossa responsabilidade.”

**Pontos de atenção:** faturamento/cotas podem aparecer; não prometer custo zero. Domínio publicado pode precisar ser autorizado no login. Não publicar com acesso administrativo exposto.

**Checkpoint:** pelo menos o app demonstrativo deve estar acessível em URL real.

### Encerramento — 105 a 110 min

**Pedir resposta de três perguntas:**

1. Qual é a fonte da verdade?
2. Como descobrimos o estado no evento 4?
3. Por que um comando rejeitado não pode virar `SwordPurchased`?

**Resumo sugerido:** “Construímos uma aplicação em que o presente é uma projeção do passado. Event Sourcing faz sentido quando esse passado tem valor suficiente para justificar a complexidade adicional.”

---

## 5. Desafios extras para uma turma avançada

### Etapa 6 — Projeções e snapshots

**Pergunta de abertura:** “Será que preciso reproduzir dez mil eventos sempre que abro a ficha?”

- Mostre o `replayEvents` completo.
- Explique snapshots como checkpoints **derivados**, nunca substitutos do histórico.
- Demonstre duas projeções: ficha do personagem e estatísticas de aventuras.
- Rode `Rebuild Projections` e compare resultados.
- Explique que gráficos podem ser reconstruídos a partir dos eventos, se os payloads históricos tiverem informações suficientes.

**Pegadinha útil:** snapshot incompatível com o código atual exige versionamento/migração ou replay completo.

### Etapa 7 — Debug Lab

**Pergunta:** “Se o ouro da interface mostrar 999 mas os eventos resultarem em 20, o que fazer?”

- Corrupted projection: comparar projeção vs replay e reconstruir.
- Duplicated command: demonstrar retry com a **mesma** idempotency key.
- Concurrent commands: demonstrar optimistic concurrency e conflito de versão.

**Cuidado:** não sugerir que Event Sourcing, sozinho, garante exactly-once delivery. Retries e consumidores ainda precisam de idempotência.

---

## 6. Perguntas frequentes e respostas

**Event Sourcing é igual a Event-Driven Architecture?**

Não. Um sistema pode publicar eventos para comunicar mudanças sem usá-los como fonte da verdade. Em Event Sourcing, o estado é derivado da sequência persistida de eventos.

**Preciso de Kafka?**

Não. O evento pode ser persistido em Firestore, PostgreSQL ou uma solução dedicada. Um broker pode ajudar a distribuir eventos, mas não é requisito básico.

**CQRS é obrigatório?**

Não. Ele costuma ser complementar: separar escrita e leitura ajuda quando as projeções têm necessidades distintas. O workshop usa projeções para ensinar a ideia, sem exigir dois serviços independentes.

**Por que usar Firestore?**

Porque a integração com AI Studio permite criar rapidamente um app com login, dados persistidos e deploy. É uma decisão didática, não uma afirmação de que Firestore é sempre a melhor opção de Event Store.

**Por que não deixar o browser escrever eventos diretamente?**

Porque a validação do domínio precisa ser confiável. Um usuário pode modificar requisições; regras de propriedade, saldo e concorrência devem ser executadas do lado confiável.

**O histórico nunca pode ser alterado?**

Em geral, eventos de domínio confirmados são append-only. Erros são tratados por eventos corretivos e projeções recalculadas. Requisitos legais, retenção e privacidade exigem projeto específico — não confundir imutabilidade lógica com permissão para reter dados pessoais indefinidamente.

**Event Sourcing seria a melhor escolha para qualquer CRUD?**

Não. Ele adiciona evolução de schema, projeções, concorrência, snapshots e operações de manutenção. Deve ser escolhido quando o histórico e a reconstituição dos estados são importantes para o domínio.

**Quanto custa publicar?**

Depende do tier disponível, projeto, região, recursos, cotas e uso de Firestore/Cloud Run. Mostre as opções que aparecerem na conta em vez de prometer gratuidade.

---

## 7. Rubrica de avaliação da oficina

| Critério | Sinal de aprendizado |
| --- | --- |
| Command × Event | Sabe explicar por que intenção não é fato |
| Fonte da verdade | Indica Event Store, não `useState` ou projeção |
| Replay | Recupera uma versão passada sem alterar histórico |
| Determinismo | Entende por que resultados aleatórios precisam estar nos eventos |
| Domínio | Mostra que uma operação inválida não gera evento confirmado |
| Persistência | Confere eventos e versões reais no Firestore |
| Segurança | Consegue justificar ID Token, autorização e Rules |
| Deploy | Acessa o app publicado, autentica e mantém dados após reload |

Uma pessoa pode concluir o aprendizado arquitetural mesmo que uma falha de cota/conta impeça o deploy individual. Nesse caso, registrar a limitação e usar a versão demonstrativa.

## 8. Plano de contingência

- **AI Studio gerou código incorreto:** usar o prompt de diagnóstico do README, preservando eventos e sem liberar segurança.
- **Configuração Firebase falhou:** demonstrar o ambiente já preparado, sem forçar um Firestore com regras abertas.
- **Atraso maior que 10 min:** pular etapas 6 e 7, que são extras.
- **Deploy indisponível para um participante:** validar o app pelo Preview e demonstrar a URL publicada do facilitador.
- **Erros em produção:** testar autenticação e domínio autorizado; verificar logs de Cloud Run, tokens, regras Firestore e permissões do serviço.

---

## 9. Encerramento e próximos passos

Sugestões para aprofundamento após o workshop:

- Implementar branching de histórico sem apagar eventos originais;
- Criar eventos de compensação em vez de rollback;
- Versionar eventos (`schemaVersion`) e suportar upcasting;
- Adicionar eventos de domínio em um sistema transacional real;
- Integrar consumidores assíncronos com controle de idempotência;
- Medir custos e limites de crescimento do Event Store.

**Mensagem final para o público:** “IA pode gerar a infraestrutura e o código rapidamente. Entender a arquitetura é o que permite saber se o sistema gerado está realmente correto.”

### Referências para fundamentação

- [Codelab do Google: videogame na Web com IA](https://codelabs.developers.google.com/codelabs/cloud-run/launch-your-web-based-video-game-with-ai?hl=pt-br)
- [Integração Google AI Studio + Firebase](https://firebase.google.com/docs/ai-assistance/ai-studio-integration)
- [Documentação do Build Mode](https://ai.google.dev/gemini-api/docs/aistudio-build-mode)
- [Firestore Security Rules](https://firebase.google.com/docs/firestore/security/get-started)
- [Cloud Run](https://cloud.google.com/run/docs)
