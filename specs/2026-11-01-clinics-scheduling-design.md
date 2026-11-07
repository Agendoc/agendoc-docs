# AgenDoc: Clínicas e agendamento (parte 1): design

- **Data:** 2026-11-01
- **Afeta:** `agendoc-api` e `agendoc-web`
- **Rascunhos aprovados:** [`assets/`](assets/). Abra os `.html` no navegador. São wireframes: o visual final segue o design system do `agendoc-web`.
- **Legenda:** **[autor]**: decidido pelo autor. **[builder]**: escolha de quem escreveu este design, que pode ser revista.

## Intenção

Hoje a jornada para no cartão de pré-triagem. O botão "Agendar consulta" leva para uma tela "Em construção". Esta parte fecha o ciclo: **pré-triagem → achar e comparar clínicas → marcar → acompanhar a consulta**. Do outro lado, a clínica recebe e gerencia os agendamentos.

O objetivo é um MVP que as pessoas poderiam usar de verdade, construído **como se** houvesse integração com clínicas reais **[autor]**. Os dados são de demonstração (fictícios e identificados), mas o sistema funciona de ponta a ponta. Ninguém deve precisar "fingir" um passo.

A pesquisa de validação ([`../research/`](../research/)) apontou a dor ("não sei que médico procurar"), os fluxos alternativos que faltavam (sem horário, remarcar, cancelar) e a confiança na IA como ponto crítico.

**Sucesso:** numa demonstração, uma pessoa conversa no chat, recebe a pré-triagem, encontra uma clínica no mapa, marca um horário com um médico e vê a consulta em "Minhas consultas". A clínica vê a consulta no painel (com o resumo da triagem, se a pessoa compartilhou) e marca como realizada. A pessoa avalia, e a nota aparece na busca.

## Escopo

### Dentro
1. Cadastro de clínicas, médicos, horários de atendimento, bloqueios, convênios e locais de pronto atendimento (UPA e hospitais 24 h).
2. Busca com mapa e filtros (paciente).
3. Agendar: escolher dia e horário, revisar, confirmar ou solicitar.
4. Minhas consultas: listar, detalhe, cancelar, remarcar, adicionar ao calendário (.ics).
5. Avaliação depois da consulta realizada.
6. Emergência: sem agendamento; mostra 192 e pronto atendimentos próximos.
7. Perfil do paciente: telefone, endereço, convênio.
8. Painel da clínica.
9. Papel `CLINIC` e criação de clínicas por um admin.

### Fora (não-objetivos)
- E-mail e WhatsApp (resumo da triagem, lembretes, confirmação de e-mail, recuperação de senha): são as partes 2 e 3. Esta parte só **guarda** a preferência de lembrete.
- Chat entre clínica e paciente dentro do app ([roadmap](../roadmap.md)).
- Pagamento, checagem de elegibilidade do convênio, telemedicina, prontuário.
- Vários usuários por clínica ou login por médico. Clínica que se cadastra sozinha. Interface de admin.
- Chatbot na tela de busca (roadmap, se sobrar tempo).

### O que tornaria isto um fracasso, mesmo funcionando (anti-objetivos)
- Dado fictício parecendo real: toda tela de paciente com clínicas mostra "Dados de demonstração", e nenhum nome é de clínica ou médico real.
- Dois pacientes no mesmo horário do mesmo médico.
- A clínica ver o chat do paciente, ou ver qualquer dado de paciente sem consulta com ela.
- Linguagem de diagnóstico ("você tem X") em qualquer tela nova.
- Emergência levando para agendamento em vez de 192.

## Modelo (conceitual)

O builder define nomes de tabelas e colunas, seguindo as convenções do repositório.

- **Clínica:** nome, endereço, latitude e longitude, telefone, WhatsApp, foto opcional, convênios aceitos, orientações fixas pré-consulta (lista curta de textos), modo de confirmação (`AUTOMATIC` | `MANUAL`, padrão `AUTOMATIC` **[builder]**), tipo (`CLINIC` | `EMERGENCY_CARE`). As UPAs e os hospitais 24 h são registros `EMERGENCY_CARE`: aparecem só no cartão de emergência, sem médicos nem agenda **[builder]**.
- **Médico (profissional):** pertence a uma clínica. Tem nome, CRM e UF, especialidade, valor particular e duração da consulta **[autor: agenda por médico; detalhes builder]**.
- **Especialidades:** o catálogo é a lista `TriageTool.SPECIALTIES` que já existe ("Ortopedista", "Clínico Geral"...). É uma fonte só, usada pela triagem e pela busca. Na interface pode aparecer como "Ortopedia" etc. **[builder]**.
- **Convênios:** lista fixa no código (Unimed, Bradesco Saúde, SulAmérica, Amil, Hapvida, NotreDame Intermédica, Porto Saúde). A clínica escolhe quais aceita **[builder]**. "Particular" é sempre aceito, com o valor do médico.
- **Horários de atendimento:** regras semanais por médico (dia da semana, início, fim, pausas). Os horários livres são **calculados**: regras menos bloqueios menos consultas ativas. Não se guarda uma linha por horário **[builder]**.
- **Bloqueio:** um intervalo de datas e horas, para a clínica inteira ou para um médico (feriado, férias).
- **Consulta:** paciente, médico, início e fim, forma de pagamento (convênio + carteirinha opcional, ou particular), status, motivo do cancelamento, quem cancelou, link opcional para a mensagem de pré-triagem, **cópia** do resumo da triagem quando compartilhado, e preferência de lembrete.
- **Avaliação:** uma por consulta realizada, de 1 a 5 estrelas, com comentário opcional em texto puro (até 500 caracteres) **[builder]**. A nota da clínica é a média das avaliações de todas as suas consultas, com a contagem.
- **Usuário:** ganha o papel `CLINIC` e o vínculo com uma clínica (uma clínica tem um login **[autor]**). Ganha também os campos de perfil do paciente: telefone, endereço com latitude e longitude, convênio padrão e carteirinha.

### Status da consulta
```
AUTOMATIC: (marcar) ───────────────▶ CONFIRMED
MANUAL:    (solicitar) ─▶ PENDING ─aceitar─▶ CONFIRMED
                            ├─recusar──▶ DECLINED
                            └─sem resposta─▶ EXPIRED
CONFIRMED ─▶ COMPLETED | NO_SHOW (clínica)  |  CANCELLED (paciente ou clínica)
PENDING   ─▶ CANCELLED (paciente)
```

## Regras

- **Sem horário duplicado:** a garantia fica **no banco** (restrição ou trava transacional), não só em verificação na aplicação. Horários com consulta `PENDING` ou `CONFIRMED` ficam indisponíveis. Quando dois pacientes tentam o mesmo horário, o segundo recebe um erro claro ("horário acabou de ser ocupado") e a grade atualizada.
- **Fuso:** horários exibidos e calculados em `America/Sao_Paulo`; armazenados em UTC, como no resto da API.
- **Janela de agendamento:** de agora + 1 h até 30 dias à frente **[builder]**.
- **Cancelar e remarcar (paciente):** até 2 h antes **[autor]**. Depois disso, a interface orienta a falar com a clínica. Remarcar = cancelar a antiga e criar a nova **na mesma transação**, com o mesmo médico ou outro da mesma clínica e especialidade. Se o novo horário falhar, a antiga continua valendo.
- **Remarcar em clínica `MANUAL` [builder]:** a nova consulta nasce `PENDING`, com um vínculo "substitui a consulta X", e a antiga continua `CONFIRMED`, segurando o horário dela. Se a clínica aceitar, a antiga é cancelada automaticamente (motivo "remarcada") na mesma transação. Se recusar ou expirar, a antiga continua valendo e o paciente vê o aviso. Enquanto isso, o par conta como **uma** consulta no limite de ativas. Só pode haver uma remarcação pendente por consulta.
- **Cancelar (clínica):** a qualquer momento antes da consulta, com motivo obrigatório. O paciente vê o motivo e o botão "Escolher novo horário".
- **Expiração do `PENDING`:** expira quando passam 24 h sem resposta **ou** quando falta 2 h para a consulta, o que vier primeiro **[builder; ajusta o palpite "24 h antes", que não funcionava para horários do mesmo dia]**. Pode ser resolvido na leitura ou por um job agendado: escolha do builder.
- **Realizada ou faltou:** a clínica só marca depois do início da consulta. "Faltou" não tem nenhuma penalidade no MVP **[autor]**.
- **Avaliação:** só para consulta `COMPLETED` e só pelo próprio paciente. Uma vez enviada, não se edita no MVP **[builder]**.
- **Avaliações públicas [builder]:** o paciente vê, na tela da clínica ("Avaliações ›"), a média, a contagem e os comentários mais recentes, todos **anônimos** ("Paciente verificado · mês/ano"), sem nome nem médico. O campo de comentário avisa para não incluir dados de saúde ou nomes. No MVP não há moderação nem resposta da clínica. A clínica vê as mesmas avaliações, também anônimas.
- **Limites contra abuso [builder]:** no máximo 5 consultas ativas (`PENDING` ou `CONFIRMED`) por paciente e 10 ações de marcar, remarcar ou cancelar por hora. Mesmo padrão de contador em memória que a API já usa.
- **Isolamento:** o paciente só vê as próprias consultas. A clínica só vê as consultas, os médicos, os bloqueios e as avaliações dela. Recurso de outro dono responde 404, como nas conversas.
- **Resumo da triagem para a clínica:** opt-in na tela de confirmar. **Desmarcado por padrão [builder; muda o palpite aprovado]**: a LGPD exige consentimento específico e destacado para dado de saúde, e uma caixa já marcada não vale como consentimento. No momento da marcação, o resumo é **copiado** para a consulta, para a clínica continuar vendo mesmo se o paciente apagar a conversa. Vão só condição provável, especialidade, urgência e o resumo de sintomas. Nunca o chat.
- **Resumo de sintomas:** a ferramenta `registrar_triagem` ganha o campo `resumo` (uma ou duas frases com o que o paciente relatou, até 300 caracteres), com as mesmas proteções de saída das outras propriedades. Triagens antigas sem `resumo` compartilham só o que têm.
- **Emergência:** com `urgency = EMERGENCY`, o cartão de triagem não mostra "Agendar consulta". Mostra "Ligar 192" (`tel:192`) e os 3 locais `EMERGENCY_CARE` mais próximos, com distância e rota (link para o mapa externo).

## Telas: paciente (web, coluna de celular)

As referências visuais são os `.html` em `assets/`.

- **Navegação [builder]:** barra inferior com Conversas (`/chat`), Consultas e Perfil, presente nas telas do paciente. Em "Consultas" também há um botão "Agendar consulta" que abre a busca com a especialidade a escolher, para quem quer marcar sem passar pela triagem **[builder]**.
- **Busca: mapa primeiro com lista deslizante [autor, opção B]:** filtros em chips (convênio, dia (hoje, amanhã ou data), período, distância, nota mínima) e ordenação (mais cedo, mais perto, melhor nota). A especialidade e o convênio do Perfil vêm pré-aplicados, com o selo "Indicado na pré-triagem: X" quando vem do chat. Os pins mostram nota e próximo horário. Os cartões mostram nota e contagem, distância, convênios, valor particular e os próximos horários (tocar no horário já seleciona). Estados: sem resultados com os filtros (sugere afrouxar), sem localização, carregando e erro.
- **Clínica, dia e horário:** dados da clínica, botões de ligar, WhatsApp (`wa.me`) e rota, faixa de dias e horários por médico, separados em manhã e tarde. Horário ocupado aparece riscado. Com mais de um médico da especialidade, os horários aparecem agrupados por médico (nome e CRM).
- **Revisar e confirmar:** resumo do horário, convênio (com carteirinha opcional) ou particular, opt-in da triagem, lembrete por e-mail (guardado, o envio vem na parte 2) e WhatsApp "em breve" (desabilitado). O botão é "Confirmar agendamento" (automático) ou "Solicitar horário" (manual).
- **Confirmado:** "Tudo certo!" ou "Aguardando a clínica". Mostra: adicionar ao calendário, como chegar, orientações (as fixas da clínica + as `recommendations` da triagem, sem nova chamada à IA **[builder]**), aviso "orientações gerais, não substituem a avaliação médica" e sinais de alerta fixos com o 192. "Enviar resumo por e-mail" fica fora até a parte 2.
- **Minhas consultas:** abas Próximas e Anteriores, com status em selo. As consultas realizadas sem avaliação mostram as estrelas.
- **Detalhe da consulta:** contato e rota, calendário, orientações, a pré-triagem de origem (com link para a conversa, se ela ainda existir) e os botões remarcar e cancelar (confirmação dentro da interface, como já se faz ao apagar conversa).
- **Perfil:** nome (somente leitura), telefone, endereço, convênio padrão e carteirinha.
- **Rótulo "Dados de demonstração"** em toda tela de paciente que mostra clínicas ou médicos.

## Telas: painel da clínica

- Mesmo app Next.js, em uma área própria (ex.: `/clinica`), com layout mais largo (menu lateral) **[builder]**. Pensado para computador, mas usável no celular.
- Depois do login, quem tem o papel `CLINIC` vai para o painel, e o paciente vai para `/chat`. As rotas de paciente e de clínica ficam protegidas por papel (servidor e cliente), como a proteção atual do `/chat`.
- **Agenda:** semana por médico (com filtro), com consultas confirmadas, solicitações pendentes, horários livres e bloqueios. Ao clicar numa consulta, aparece o detalhe: paciente (nome, telefone, convênio e carteirinha), resumo da triagem (se compartilhado, com o aviso "gerado por IA, não é diagnóstico") e os botões realizada, faltou e cancelar (com motivo).
- **Solicitações:** lista de pendentes com aceitar e recusar, mais a regra de expiração explicada na tela.
- **Horários de atendimento:** regras semanais por médico. **Médicos:** adicionar, editar e desativar (desativar não apaga as consultas existentes, só tira o médico da busca) **[builder]**.
- **Bloqueios:** criar e remover. Quando um bloqueio cai sobre consultas existentes, a interface avisa quantas são e oferece cancelar todas com um motivo **[builder]**.
- **Dados da clínica:** endereço, contatos, convênios, orientações fixas e modo de confirmação. A clínica não altera a localização no mapa (ela vem do admin) **[builder]**.
- **Avaliações:** lista somente leitura, com a média.

## Admin e dados de demonstração

- **Criar clínica e login:** endpoint `ADMIN` documentado no Swagger, sem interface **[builder]**. Recebe os dados da clínica, com latitude e longitude, e o e-mail do login. A senha inicial é gerada e devolvida uma única vez. Nada de senha fixa no código (regra do prompt original).
- **Seed de demonstração:** cerca de 12 clínicas fictícias em São Paulo (Paulista, Pinheiros e Vila Mariana) **[builder, autor delegou a cidade]**, cobrindo todas as especialidades do catálogo, com 1 a 3 médicos cada, mais 4 locais `EMERGENCY_CARE`. Algumas clínicas são `MANUAL`, para mostrar o fluxo de aprovação. Há avaliações iniciais marcadas como de demonstração. Nomes, CRMs e endereços **fictícios** (números de CRM fora de qualquer faixa plausível, ou claramente marcados). As senhas dos logins de demonstração das clínicas vêm de variável de ambiente; sem ela, o seed não cria logins. O seed precisa rodar também em produção (a demo é lá) e ser idempotente. O mecanismo (migração Flyway separada, runner com perfil...) é escolha do builder, respeitando a regra de migrações só aditivas.

## Localização e mapa [builder]

- Leaflet com os tiles do Esri World Street Map (sem chave e sem custo, atribuição "Tiles © Esri" visível). *Atualizado em 07/11/2026:* os tiles padrão do OpenStreetMap ficaram feios no teste, e a CARTO passou a exigir chave. O mapa interativo do Google exige cobrança, então fica de fora. Se o volume ou a política virarem problema, trocar por um provedor gratuito com chave.
- **Rota** ("Como chegar"): abre o app de mapas da pessoa por link, sem API e sem custo. O menu oferece Google Maps e Waze, e também Apple Mapas em iPhone e Mac. A origem é a localização atual da pessoa (o próprio app pega). No cartão de emergência, o link vai direto para o Google Maps, sem menu, para ser mais rápido.
- A localização do paciente vem da geolocalização do navegador (com permissão). Se o paciente recusar, usa o endereço do Perfil. Sem nenhum dos dois, ou a mais de 50 km da área da demo, o mapa centraliza em São Paulo e as distâncias são medidas a partir do centro do mapa.
- Endereço do Perfil para latitude e longitude: geocodificação no backend (ex.: Nominatim, respeitando o limite de 1 requisição por segundo e um User-Agent identificado), feita ao salvar o perfil. Se falhar, salva sem coordenadas e avisa.
- A distância é calculada no backend (haversine). Com 12 clínicas não precisa de índice geoespacial.
- A localização do paciente não é guardada no servidor, exceto o endereço que ele mesmo salvou no Perfil.

## Calendário

"Adicionar à agenda" abre um menu:

- **Google Agenda:** link `calendar.google.com/calendar/render?action=TEMPLATE…` que abre o evento já preenchido (título, horário, endereço, médico e link de volta para a consulta), sem baixar arquivo. Os dados vão para o Google só quando a pessoa toca no botão, e nunca incluem a triagem.
- **iPhone, Outlook e outros:** `GET` de um `.ics` por consulta (só o dono), com título, local, horário e status, servido como `inline` para o Safari do iPhone abrir direto "Adicionar ao Calendário".

*Atualizado em 07/11/2026:* baixar o arquivo era estranho, e o Google Agenda no Android não abre `.ics` sozinho.

## Integração com o que existe

- O cartão de triagem passa a abrir a busca levando a especialidade **e** a referência da mensagem de triagem (para o vínculo e o opt-in). A rota `/chat/agendar` placeholder é substituída.
- O mock MSW do `agendoc-web` precisa cobrir os endpoints novos (busca, horários, consultas, avaliações, painel), para o `npm run ci` e o desenvolvimento sem backend continuarem funcionando.
- O mock do assistente (`MockChatAssistant`) passa a preencher `resumo`, e precisa existir um gatilho para gerar triagem `EMERGENCY` em dev e testes.
- Erros novos entram no padrão `ProblemDetail` com `code` estável (ex.: `SLOT_UNAVAILABLE`, `CANCELLATION_WINDOW_CLOSED`, `APPOINTMENT_LIMIT`).

## Restrições

- Convenções dos repositórios: camadas (`controllers`, `command/<feature>/inputs|outputs`, `services`...), código em inglês, interface em pt-BR, Swagger só nos controllers, poucos comentários, commits semânticos curtos em inglês.
- Migrações Flyway somente aditivas (o rollback automático do deploy depende disso).
- Gates verdes: `./mvnw verify` na API e `npm run ci` na web, a cada etapa. Testes de integração com MySQL real (Testcontainers) para as regras de concorrência, janela, isolamento e status.
- Toda variável de ambiente nova (ex.: as senhas dos logins de demonstração, o User-Agent do geocoding) entra no `deploy/compose.yaml` e no `.env.example`.
- Acessibilidade: o mapa nunca é o único jeito de escolher. A lista funciona sozinha, por teclado e leitor de tela.

## Prazo e execução [autor]

Sem data fixa: a ideia é avançar o máximo possível, com a versão completa de cada item (nada de versão simplificada por falta de tempo). Uma pessoa só implementa, então a entrega vai **feature por feature, de ponta a ponta** (API + web + mock juntos em cada etapa). Os planos são enxutos.

## Decisões durante a implementação

- **07/11/2026:** a lista de Minhas consultas e o cancelamento foram adiantados (antes do Perfil), porque sem eles não havia onde ver a consulta marcada.
- **07/11/2026:** consulta confirmada ou pendente cujo horário já passou aparece como **"Encerrada"**, sem ações, até a clínica registrar "Realizada" ou "Faltou" no painel.
- O registro completo das decisões de implementação, com o custo de cada uma se estiver errada, fica no ledger do plano (`.superpowers/`, fora do git); as que mudam o produto entram aqui.

## Deixado para o builder

A ordem das entregas (sugestão: modelo e seed → busca → marcar → minhas consultas → painel → avaliações → emergência), os contratos exatos da API, os nomes das rotas web, a paginação, o visual final dentro do design system, a estratégia de trava contra horário duplicado e de expiração dos pendentes.
