# AgenDoc: E-mail (parte 2): design

- **Data:** 2026-11-09
- **Afeta:** `agendoc-api` e `agendoc-web`
- **Legenda:** **[autor]**: decidido pelo autor. **[builder]**: escolha de quem escreveu este design, que pode ser revista.

## Intenção

Até a parte 1, qualquer pessoa cria conta com um e-mail que nem é dela, e o app nunca fala com o paciente fora da tela. Esta parte coloca o e-mail no meio da jornada: a conta prova que tem acesso ao e-mail, quem esquece a senha consegue voltar sozinho, e a consulta marcada chega na caixa de entrada com o resumo da pré-triagem, os avisos da clínica e os lembretes.

**Sucesso:** numa apresentação, a pessoa cria uma conta, recebe o código e confirma o e-mail. Depois, esquece a senha e redefine pelo código. Com uma conta de teste, marca uma consulta a partir da triagem e mostra, na caixa de entrada real do apresentador, o e-mail com o resumo da triagem e o lembrete de exemplo, sem esperar 24 h.

## Escopo

### Dentro [autor]
1. Confirmação de e-mail por código, com bloqueio do app até confirmar.
2. Recuperação de senha por código.
3. Contas de teste já confirmadas, com um "E-mail de envio" real que só elas veem no Perfil.
4. E-mail da consulta marcada com o resumo da triagem (automático), mais avisos de remarcação e da resposta da clínica.
5. Lembretes por e-mail com horários escolhidos pela pessoa, pensados para receber o WhatsApp depois.
6. Lembrete de exemplo enviado na hora, só para as contas de teste.

### Fora
- WhatsApp (parte 3). O modelo dos lembretes já prevê o canal, mas só o e-mail é implementado.
- E-mails para a clínica (nova solicitação, cancelamento pelo paciente).
- Trocar o e-mail da conta, editar os lembretes depois de marcar, link de descadastro, e-mail de marketing.
- Caixa de entrada de demonstração dentro do app (descartada pelo autor: complexa demais).

### Anti-objetivos
- Falha no envio de e-mail nunca impede marcar, remarcar, cadastrar ou responder uma solicitação.
- A recuperação de senha nunca revela se um e-mail tem conta.
- O campo "E-mail de envio" nunca aparece nem é aceito para contas normais **[autor: "isso é bem importante"]**.

## Envio

- **Produção [autor]:** SMTP do Gmail com a conta `agendoc.app@gmail.com` e uma senha de app (porta 587, STARTTLS; a Oracle bloqueia a 25). Remetente: `AgenDoc <agendoc.app@gmail.com>`. O limite do Gmail (cerca de 500 por dia) basta para a demo.
- **Código:** SMTP genérico (`spring-boot-starter-mail`), configurado só por variáveis de ambiente, para trocar de serviço sem mexer no código **[builder]**.
- **Desenvolvimento [builder]:** Mailpit no `compose.yaml` local. Ele captura tudo e mostra em `http://localhost:8025`. Nada sai de verdade.
- **Testes [builder]:** um enviador falso que guarda as mensagens em memória, verificado nos testes de integração.
- **Envio fora da transação [builder]:** o e-mail sai depois do commit, em segundo plano. Uma falha vira log e métrica (sem o código nem o conteúdo no log), e a ação da pessoa segue normal.
- **Para quem vai:** contas normais recebem no e-mail de login. Contas de teste recebem no "E-mail de envio". Sem ele, nada é enviado. Clínicas e admin não recebem nada nesta parte.

## Confirmação de e-mail

- **Fluxo [autor]:** o cadastro continua abrindo a sessão, mas a pessoa vai para a tela "Confirme seu e-mail" e fica **bloqueada** até digitar o código de 6 dígitos que recebeu. A API recusa chat, busca, consultas e perfil para quem não confirmou, com o erro `EMAIL_NOT_VERIFIED` **[builder: o código do erro]**. O login de uma conta não confirmada funciona e leva para a mesma tela. O bloqueio vale só para o papel `USER`: clínica e admin não mudam. Continuam abertos para quem não confirmou: `me`, `logout`, `csrf`, confirmar e reenviar o código **[builder]**.
- **Na tela [builder]:** "Enviamos um código para x@y", o campo do código, "Reenviar código" (com contagem regressiva) e "Sair".
- **Regras do código [builder]:**
  - 6 dígitos aleatórios, guardados só com hash.
  - Validade de 15 minutos e até 5 tentativas erradas. Depois disso, o código deixa de valer e é preciso pedir outro.
  - Um código novo invalida o anterior.
  - O reenvio tem espera de 60 segundos e limite de 5 por hora por conta.
- **Contas que já existem [builder]:** a migração marca todos os usuários atuais como confirmados. Ninguém em produção é bloqueado de repente. Logins de clínica criados pelo admin também nascem confirmados.
- **E-mail tomado por engano [builder]:** quem cadastrar o e-mail de outra pessoa não consegue confirmar. A dona real recupera a conta pelo "Esqueci a senha", que também confirma o e-mail.

## Recuperação de senha

- **Fluxo [autor: código, não link]:** "Esqueci a senha" no login leva a uma tela em dois passos. No primeiro, a pessoa informa o e-mail. No segundo, informa o código, a nova senha e a confirmação. No fim, volta para o login com a mensagem "Senha alterada. Entre com a nova senha." **[builder: sem login automático]**.
- **Regras [builder]:**
  - Mesmas regras de código da confirmação, com um código próprio para redefinição.
  - A resposta do primeiro passo é sempre a mesma, exista a conta ou não, com limites por e-mail e por IP.
  - Trocar a senha encerra todas as sessões abertas da conta (pelo índice de sessões por usuário do Spring Session JDBC) e marca o e-mail como confirmado.
  - Chega também um e-mail "Sua senha foi alterada", com a orientação de procurar ajuda se não foi a pessoa.
  - A nova senha segue as mesmas regras do cadastro.
- Vale também para as contas de teste. O código vai para o "E-mail de envio" e, sem ele, não chega a lugar nenhum.

## Contas de teste

- **[autor]** As contas de teste são criadas pelo seed, com e-mail no domínio falso `@demo.agendoc.dev`, já confirmadas. Elas guardam a marca "conta de teste".
- **[builder]** Há 3 contas: `paciente1`, `paciente2` e `paciente3@demo.agendoc.dev`, com a senha vinda de `DEMO_PATIENT_PASSWORD`. Sem essa variável, o seed não cria as contas, como já acontece com os logins de clínica. As contas são criadas sempre que faltarem, mesmo num banco que já tem clínicas, porque a produção já foi semeada.
- **"E-mail de envio" [autor]:** é um campo do Perfil que só aparece para contas de teste, com a explicação "Conta de teste: os e-mails desta conta vão para este endereço". A API recusa o campo vindo de qualquer outra conta. O campo é opcional e não exige confirmação **[builder]**. O seed não preenche o campo: antes da demo, o apresentador coloca o próprio e-mail no Perfil.
- **Lembrete de exemplo [autor]:** ao marcar com o lembrete ligado, a conta de teste recebe na hora um e-mail igual ao lembrete do primeiro horário escolhido, com o assunto começando por "[Exemplo]" e uma faixa no topo: "Exemplo: é assim que você receberá o lembrete 1 dia antes da consulta". Os lembretes reais continuam agendados normalmente.

## E-mails da consulta

Todos em pt-BR, em HTML simples nas cores do design system (tokens do `tokens.json`), com uma versão em texto puro e um botão "Ver consulta" que abre `/consultas/{id}` no app **[builder]**.

| E-mail | Quando | Conteúdo |
|---|---|---|
| Consulta marcada | Ao marcar, sempre **[autor]** | Clínica, médico, especialidade, data e hora, endereço, status ("confirmada" ou "aguardando a clínica"), forma de pagamento. Se veio de uma triagem: urgência, especialidade sugerida, condição provável, resumo do que a pessoa relatou e recomendações. Sempre: o aviso "orientações gerais, não substituem a avaliação médica" e os sinais de alerta com o 192. |
| Consulta remarcada | Ao remarcar | Horário antigo e novo, status da nova. |
| Resposta da clínica | Solicitação aceita, recusada ou expirada, ou consulta cancelada pela clínica (com o motivo) | O que aconteceu e o próximo passo ("Escolher novo horário" quando couber). |
| Lembrete | No horário escolhido | Data e hora, clínica, endereço, link de rota e a orientação de chegar com antecedência. |

- **Triagem no e-mail [autor delegou; builder]:** o autor liberou mandar o que for útil. Vão os campos acima, que são o que o cartão de triagem já mostra, mais o resumo. A conversa nunca vai. O resumo da triagem vai no e-mail da própria pessoa mesmo quando ela não compartilhou com a clínica: ele é lido da mensagem de triagem de origem (`triageMessageId`) na hora do envio, logo depois de marcar, sem guardar nada novo na consulta, então a clínica continua vendo só o que foi compartilhado. Se a mensagem não existir mais, o e-mail vai sem a parte da triagem. Remarcação, avisos e lembretes não levam a triagem.
- **Remarcação em clínica `MANUAL`:** o e-mail diz que a nova data aguarda a clínica. A resposta chega depois, pelo e-mail "Resposta da clínica".

## Lembretes

- **Escolha [autor]:** na confirmação da consulta, abaixo do Switch "Lembrete por e-mail", a pessoa marca um ou mais horários entre **1 semana, 1 dia, 2 horas e 1 hora antes**. O padrão é **1 dia e 2 horas**, como no design system. Com o Switch ligado, é preciso ter ao menos um horário. Horários que já teriam passado para aquela consulta (ex.: "1 semana" numa consulta daqui a 3 dias) aparecem desabilitados **[builder]**. Na remarcação, as escolhas passam para a consulta nova.
- **Canal [autor]:** cada lembrete guarda o canal (`EMAIL` por enquanto), para a parte 3 acrescentar o WhatsApp sem refazer o modelo.
- **Envio [builder]:**
  - Uma tarefa agendada roda a cada minuto e envia os lembretes cujo horário chegou.
  - Ela só envia se a consulta ainda estiver `CONFIRMED`. Cancelada, recusada, expirada ou trocada numa remarcação, a consulta simplesmente não recebe mais lembretes. Uma consulta `PENDING` aceita depois recebe os lembretes que ainda não passaram.
  - A API roda numa instância só (uma VM, um contêiner), então a tarefa não precisa de trava entre instâncias.
  - Horários que já passaram na hora de marcar ou de aceitar são pulados, sem envio atrasado.
  - Cada lembrete é enviado no máximo uma vez. Falhas tentam de novo até 3 vezes.
- A coluna `email_reminder` da parte 1 continua sendo o Switch. Os horários ficam numa tabela nova **[builder]**.

## Telas (web)

Seguem o design system (`agendoc-web/CLAUDE.md`): componentes de `src/components/ui` primeiro, estilo só por tokens.

- **Confirme seu e-mail (`/confirmar-email`) [builder: nome da rota]:** layout das telas de login. O layout do paciente manda para cá quem não confirmou. O cadastro também termina aqui.
- **Esqueci a senha (`/esqueci-senha`) [builder: nome da rota]:** os dois passos, com link a partir do login.
- **Campo do código [builder]:** `TextField` numérico (`inputMode="numeric"`, `autocomplete="one-time-code"`), já que o DS não tem um componente próprio de código.
- **Perfil:** o campo "E-mail de envio" só para contas de teste.
- **Confirmar consulta:** os horários do lembrete em `Chip` de múltipla escolha abaixo do Switch, escondidos com o Switch desligado. Para conta de teste, mais uma linha: "Conta de teste: você também recebe agora um exemplo do lembrete".
- O mock MSW cobre os endpoints novos, para o `npm run ci` e o desenvolvimento sem backend continuarem funcionando.

## Restrições

- Convenções dos repositórios: camadas (`controllers`, `command/<feature>/inputs|outputs`, `services`...), código em inglês, interface e e-mails em pt-BR, Swagger só nos controllers, poucos comentários, commits semânticos curtos em inglês.
- Migrações Flyway somente aditivas.
- Gates verdes a cada etapa: `./mvnw verify` na API e `npm run ci` na web. Testes de integração com MySQL real (Testcontainers) para o bloqueio, os códigos, os limites e os lembretes.
- Variáveis novas no `deploy/compose.yaml` e no `.env.example`, sem segredo no repositório: SMTP (host, porta, usuário, senha), remetente, URL pública do app (para os links) e `DEMO_PATIENT_PASSWORD`. A senha de app do Gmail vai só no `deploy/.env` da VM.
- Erros novos no padrão `ProblemDetail` com `code` estável (ex.: `EMAIL_NOT_VERIFIED`, `INVALID_CODE`, `CODE_EXPIRED`).

## Entregas [builder]

Mesmo fluxo da parte 1: branches por tarefa → `feat/email` em cada repositório → marcos testados pelo autor no front → `develop`. No fim, `develop` → `main` (deploy) só com o pedido do autor.

| Marco | Conteúdo |
|---|---|
| M1 | Envio (SMTP + Mailpit), confirmação de e-mail com bloqueio, recuperação de senha, contas de teste com "E-mail de envio" |
| M2 | E-mail da consulta com a triagem, avisos de remarcação e da clínica, lembretes com escolha de horários, lembrete de exemplo |
| M3 | Configuração de produção (Gmail), roadmap e documentação, revisão final e deploy |

## Deixado para o builder

Os contratos exatos da API, os nomes das tabelas, o mecanismo do bloqueio (autoridade da sessão, filtro...), o motor de templates dos e-mails, o texto final de cada e-mail e o visual dentro do design system.
