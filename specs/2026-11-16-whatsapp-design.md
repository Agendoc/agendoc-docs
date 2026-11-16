# AgenDoc: WhatsApp (parte 3): design

- **Data:** 2026-11-16
- **Afeta:** `agendoc-api` e `agendoc-web`
- **Estudo:** [research/2026-11-16-whatsapp-integracao.md](../research/2026-11-16-whatsapp-integracao.md)
- **Legenda:** **[autor]**: decidido pelo autor. **[builder]**: escolha de quem escreveu este design, que pode ser revista.

## Intenção

A parte 2 levou a consulta para a caixa de e-mail. Esta parte leva o resumo da triagem e os lembretes para o WhatsApp, que é onde o paciente brasileiro realmente lê mensagens. Na pesquisa de validação, notificações e lembretes foram o recurso mais elogiado depois da triagem. É uma integração de demonstração: mostra o WhatsApp oficial funcionando de ponta a ponta para o grupo, sem virar um canal aberto a qualquer paciente.

**Sucesso:** numa apresentação, o apresentador entra com uma conta de teste que tem o próprio celular no "WhatsApp de envio", marca uma consulta a partir da triagem e, segundos depois, mostra no celular a mensagem "Consulta marcada" com o resumo da pré-triagem e o lembrete de exemplo, bem formatados, sem esperar 24 h.

## Escopo

### Dentro [autor]
1. Envio pela **API oficial da Meta (WhatsApp Cloud API)**, com o **número de teste** da Meta, só para demonstração. Os 4 celulares do grupo são cadastrados como destinatários no painel da Meta (o número de teste aceita até 5).
2. Campo **"WhatsApp de envio"** no Perfil, só para as contas de teste (`paciente1..3@demo.agendoc.dev`), igual ao "E-mail de envio".
3. Mensagem **"Consulta marcada"** pelo WhatsApp, com o resumo da triagem quando a consulta veio de uma.
4. **Lembretes** pelo WhatsApp nos horários que a pessoa escolhe, no mesmo modelo de lembretes da parte 2 (canal `WHATSAPP`).
5. **Lembrete de exemplo** enviado na hora também pelo WhatsApp.

### Fora
- Pacientes comuns. Para eles nada muda e nada de WhatsApp aparece.
- Número próprio, nome de exibição, cartão na Meta e verificação da empresa (só seriam necessários para pacientes reais).
- Webhooks da Meta: status de entrega, respostas, botões de resposta rápida (ex.: "Confirmo presença"), "PARAR". O envio é só de ida **[autor, 16/11/2026: fica como ideia no roadmap]**.
- Remarcação e respostas da clínica pelo WhatsApp (continuam só por e-mail). Mensagens para a clínica.
- Confirmar o número por código. A `paciente4` (o autor dispensou por agora).

### Anti-objetivos
- Falha no WhatsApp nunca impede marcar, remarcar nem nenhuma outra ação, e nunca impede o e-mail.
- O campo "WhatsApp de envio" nunca aparece nem é aceito para contas que não são de teste, como na parte 2.
- O app nunca manda WhatsApp para um número que não seja o "WhatsApp de envio" de uma conta de teste.
- O token da Meta nunca vai para o repositório nem para os logs.

## Envio

- **Provedor [autor]:** Meta WhatsApp Cloud API direto, sem intermediário (Twilio e bibliotecas não oficiais descartadas no estudo).
- **Código [autor: abordagem A]:** uma interface de envio no molde do `MailGateway`, implementada com o `RestClient` do Spring chamando `POST https://graph.facebook.com/{versão}/{phone-number-id}/messages` com mensagens do tipo `template`. Tudo configurado por variáveis de ambiente: ID do número, token permanente (usuário do sistema), versão da API (padrão `v27.0`) e idioma dos templates (`pt_BR`) **[builder: nomes das variáveis]**.
- **Sem configuração [builder]:** sem o ID do número ou o token, o envio vira só log da mensagem montada (o papel do Mailpit). Com as variáveis, o envio é real, também na máquina local, porque só precisa de HTTP de saída.
- **Testes [builder]:** um enviador falso em memória, verificado nos testes de integração, mais um teste do cliente HTTP contra um servidor falso (payload e tratamento de erro).
- **Fora da transação [builder]:** como o e-mail, a mensagem sai depois do commit, em segundo plano. Uma falha vira log (com o código de erro da Meta, sem o token e sem o conteúdo) e a ação da pessoa segue normal. Timeouts curtos no cliente HTTP, para uma Meta lenta não travar as tarefas agendadas.
- **Enviado = aceito pela Meta [builder]:** sem webhooks, "enviado" quer dizer que a Meta respondeu 200 com o id da mensagem. Um número fora da lista do painel falha na hora (erro da Meta) e conta como falha.
- **Para quem vai [autor]:** só contas de teste com o "WhatsApp de envio" preenchido. Sem ele, nada é enviado pelo WhatsApp.

## "WhatsApp de envio"

- **[autor]** Campo do Perfil que só aparece para contas de teste, com a explicação "Conta de teste: as mensagens de WhatsApp desta conta vão para este número". A API recusa o campo vindo de qualquer outra conta.
- **Opt-in [autor]:** preencher o campo vale como o consentimento exigido pela Meta. Apagar o campo para os envios.
- **Formato [builder]:** a pessoa digita o celular com DDD (máscara `(11) 91234-5678`); a API guarda só os dígitos no formato internacional (`5511912345678`), acrescentando o `55` quando faltar. Sem confirmação por código. O seed não preenche o campo.
- O número precisa ser um dos cadastrados no painel da Meta. Se não for, os envios falham e só aparecem no log, o que é aceitável para a demo **[builder]**.

## Mensagens

Três templates da categoria **utility**, em `pt_BR`, com texto fixo formatado (negrito, emoji, quebras de linha) e uma variável por campo. O valor de uma variável não pode ter quebra de linha nem mais de 4 espaços seguidos, então textos livres são achatados numa linha. O texto completo, depois de preenchido, precisa caber no limite de 1.024 caracteres da Meta: o relato e as recomendações são cortados com "..." quando necessário **[builder: limites de corte]**.

| Template | Quando | Conteúdo |
|---|---|---|
| Consulta marcada | Ao marcar, sempre, se a conta tem "WhatsApp de envio" **[builder: mesmo gatilho do e-mail]** | Clínica, médico e especialidade, data e hora, endereço, status ("Confirmada" ou "Aguardando a clínica") e forma de pagamento. Botão "Ver consulta". |
| Consulta marcada com triagem | Ao marcar a partir de uma triagem | O mesmo, mais o bloco "Sua pré-triagem": urgência, especialidade sugerida, condição provável, relato resumido e recomendações (separadas por " · "), e o aviso "orientações gerais, não substituem a avaliação médica; sinais de alerta: ligue 192". |
| Lembrete | No horário escolhido, e na hora como exemplo | Título que muda com o horário ("sua consulta é amanhã", "em 2 horas"...), clínica, médico e especialidade, data e hora, endereço, "Chegue 15 minutos antes". Botões "Ver consulta" e "Como chegar". No exemplo, o título começa com "[Exemplo]" e diz de qual horário é o exemplo. |

Visual aprovado pelo autor (16/11/2026):

```
🩺 *Consulta marcada*

🏥 *Clínica Vida Plena*
👨‍⚕️ Dra. Ana Souza · Cardiologia
📅 qui, 20/11 às 14:30
📍 Av. Brasil, 1200 - Centro
✅ Confirmada · Pagamento: Unimed

📋 *Sua pré-triagem*
🟠 Urgência: moderada
🔎 Especialidade sugerida: Cardiologia
💭 Condição provável: arritmia benigna
📝 Palpitações há 3 dias, pioram à noite, sem dor no peito...
💡 Evite cafeína · Anote os horários das crises

_Orientações gerais, não substituem a avaliação médica. Sinais de alerta: ligue 192._

        [ Ver consulta ]
```

```
⏰ *Lembrete: sua consulta é amanhã*

🏥 *Clínica Vida Plena*
👨‍⚕️ Dra. Ana Souza · Cardiologia
📅 qui, 20/11 às 14:30
📍 Av. Brasil, 1200 - Centro

Chegue 15 minutos antes. 😉

   [ Ver consulta ]   [ Como chegar ]
```

- **Triagem [builder]:** mesmas regras do e-mail da parte 2: lida da mensagem de triagem de origem na hora do envio, vai mesmo quando a pessoa não compartilhou com a clínica, nunca leva a conversa; se a mensagem não existir mais, vai o template sem triagem.
- **Botões [builder]:** links dinâmicos com a base fixa no template. "Ver consulta" abre `https://agendoc.duckdns.org/consultas/{id}` (o template fixa o domínio de produção, então em desenvolvimento o botão também abre a produção). "Como chegar" abre a busca do Google Maps pelo endereço da clínica, que sempre existe, em vez das coordenadas.
- **Templates no repositório [autor: abordagem A]:** o nome, a categoria, o texto e as variáveis de cada template ficam num arquivo de referência no `agendoc-api`. O autor cadastra os templates uma vez no painel da Meta seguindo esse arquivo. O código só referencia os templates pelo nome. Se a Meta recusar um template ou mudar a categoria dele para marketing, o texto é ajustado no arquivo e cadastrado de novo; enquanto isso, o desenvolvimento segue com o envio falso e o de log **[builder]**.

## Lembretes

- **Canal [autor]:** o WhatsApp entra como o valor `WHATSAPP` em `appointment_reminders.channel`, na mesma tabela e com a mesma tarefa agendada da parte 2. Cada canal tem a própria linha, o próprio status e as próprias tentativas.
- **Escolha na confirmação [builder]:** para conta de teste com o "WhatsApp de envio" preenchido, aparece um segundo Switch, "Lembrete pelo WhatsApp", ao lado do "Lembrete por e-mail". Os horários escolhidos (chips 1 semana, 1 dia, 2 h, 1 h) valem para os canais ligados; os chips aparecem quando pelo menos um Switch está ligado. Conta de teste sem o número vê uma linha "Cadastre o WhatsApp de envio no Perfil para receber por lá". Contas comuns não veem nada novo.
- **Envio [builder]:** a tarefa escolhe o envio pelo canal da linha. Valem as regras da parte 2: só consulta `CONFIRMED`, no máximo uma vez, horários passados e atrasos de mais de 10 minutos pulados, até 4 falhas. Se o "WhatsApp de envio" foi apagado depois, o lembrete é pulado.
- **Remarcação [builder]:** os canais e horários passam para a consulta nova, como já acontece com os horários.
- **Exemplo [autor]:** ao marcar com o lembrete pelo WhatsApp ligado, a conta de teste recebe na hora o lembrete de exemplo do primeiro horário escolhido, também pelo WhatsApp.

## Telas (web)

Seguem o design system (`agendoc-web/CLAUDE.md`).

- **Perfil:** o campo "WhatsApp de envio" só para contas de teste, logo abaixo do "E-mail de envio", com máscara de celular.
- **Confirmar consulta:** o Switch "Lembrete pelo WhatsApp" e a linha de ajuda descritos acima. A linha "Conta de teste: você também recebe agora um exemplo do lembrete" passa a valer para os dois canais. A tela já tem um Switch "WhatsApp · Em breve" desabilitado: para conta de teste ele vira o Switch de verdade; para contas comuns continua como está **[builder]**.
- **Detalhe da consulta [builder]:** hoje não mostra os lembretes, e continua assim. A API passa a devolver `whatsappReminder` junto de `emailReminder`.
- O mock MSW cobre os campos novos.

## Configuração na Meta [autor]

Feita uma vez pelo autor, com a conta dele, antes do M1 terminar (o passo a passo vai junto do arquivo de templates **[builder]**):
1. Criar a conta de desenvolvedor e um app com o caso de uso WhatsApp (o portfólio e o número de teste são criados juntos).
2. Cadastrar os 4 celulares do grupo como destinatários do número de teste.
3. Criar um usuário do sistema com token permanente (`whatsapp_business_messaging`, `whatsapp_business_management`).
4. Cadastrar os 3 templates e esperar a aprovação.
5. Pôr o ID do número e o token no `.env` local (para testar) e, no M3, no `deploy/.env` da VM.

## Restrições

- Convenções dos repositórios: camadas, código em inglês, interface e mensagens em pt-BR, Swagger só nos controllers, poucos comentários, commits semânticos curtos em inglês.
- Migrações Flyway somente aditivas.
- Gates verdes a cada etapa: `./mvnw verify` na API e `npm run ci` na web. Testes de integração com MySQL real (Testcontainers) para o campo, os lembretes por canal e o exemplo.
- Variáveis novas no `deploy/compose.yaml` e no `.env.example`, sem segredo no repositório. O token vai só no `deploy/.env` da VM.

## Entregas [builder]

Mesmo fluxo das partes 1 e 2: branches por tarefa → `feat/whatsapp` em cada repositório → marcos testados pelo autor no front → `develop`. No fim, `develop` → `main` (deploy) só com o pedido do autor.

| Marco | Conteúdo |
|---|---|
| M1 | Cliente da Cloud API (log, falso e real), arquivo de templates com o passo a passo, "WhatsApp de envio" no Perfil, "Consulta marcada" com e sem triagem |
| M2 | Lembretes pelo WhatsApp (canal na tabela, Switch na confirmação, remarcação, detalhe da consulta), lembrete de exemplo |
| M3 | Configuração de produção na VM, roadmap e documentação, revisão final e deploy |

## Deixado para o builder

Os contratos exatos da API, o nome da coluna nova, os nomes exatos dos templates e das variáveis, os limites de corte do texto, os timeouts, o texto final de cada mensagem dentro do visual aprovado e o visual das telas dentro do design system.
