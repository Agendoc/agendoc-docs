# Estudo: integração com o WhatsApp (parte 3)

> Estudo feito em 16/11/2026 para decidir como a API do AgenDoc manda mensagens de WhatsApp (resumo da triagem e lembretes) numa demo universitária hospedada numa única VM da Oracle. Fontes no fim. Preços e regras da Meta mudam a cada trimestre: conferir antes de ir para produção.

## Regras do WhatsApp que valem para qualquer caminho oficial

Todo caminho oficial passa pela **WhatsApp Business Platform** da Meta. Twilio, Zenvia, Gupshup e outros são revendedores (BSPs) da mesma plataforma, com as mesmas regras e uma taxa a mais.

- **Quem começa a conversa é o paciente.** A empresa só manda texto livre dentro da **janela de atendimento de 24 h**, que abre (e renova) cada vez que o usuário manda uma mensagem. Fora dela, só **templates** aprovados pela Meta.
- **Templates:** texto fixo com variáveis (`{{1}}`, `{{2}}`...), botões opcionais (link ou resposta rápida). Cada um é revisado pela Meta (em geral minutos, até 24 h) e classificado como **utility** (transacional: confirmação, lembrete de consulta), **marketing** ou **authentication** (código). Lembrete de consulta é o exemplo típico de utility. As variáveis não aceitam quebra de linha, então um resumo longo não cabe bem num template.
- **Opt-in:** é obrigatório ter o consentimento da pessoa antes de mandar templates, deixando claro o nome da empresa e o que ela vai receber. No app, isso é um aceite explícito ("Quero receber pelo WhatsApp") junto do número.
- **Preço (desde 01/07/2025, por mensagem entregue):** cobra por categoria e país do destinatário. Para o Brasil, utility custa cerca de **US$ 0,0068 (≈ R$ 0,04)** por mensagem; marketing custa ~US$ 0,06. Desde **01/10/2026**, utility dentro da janela também é cobrado. As mensagens livres de atendimento (service) têm **1.000 grátis por mês** por número. Mensagens que o usuário manda para a empresa são grátis.
- **Limite de envio:** contas novas falam com até **250 pessoas diferentes por 24 h** fora da janela (sobe para 2.000 com a verificação da empresa). Uma mensagem a cada 6 s para a mesma pessoa.
- **Número:** precisa de um chip que receba SMS ou ligação e **não esteja em uso no WhatsApp comum**, nome de exibição revisado pela Meta e PIN de verificação em duas etapas.
- **Webhooks:** a Meta avisa por HTTPS o status de cada mensagem (enviada, entregue, lida, falhou) e as mensagens recebidas (respostas, cliques em botões, "PARAR"). Precisa de um endpoint público com token de verificação e assinatura `X-Hub-Signature-256`. A VM já tem HTTPS (`agendoc.duckdns.org`).
- **Token:** o token temporário do painel expira em horas. Para a API, um **usuário do sistema** com token permanente (`whatsapp_business_messaging`, `whatsapp_business_management`).

## Opções

| | Meta Cloud API (direto) | Twilio | Outros BSPs (Zenvia, Gupshup, 360dialog...) | Bibliotecas não oficiais (Baileys, Evolution API, WPPConnect) |
|---|---|---|---|---|
| Como funciona | API HTTP da Meta (`graph.facebook.com/v27.0/{phone-id}/messages`), sem intermediário | API da Twilio sobre a da Meta | Painel e API próprios sobre a da Meta | Simula o WhatsApp Web num número comum, lendo um QR code |
| Teste grátis | **Número de teste criado na hora**, sem cartão, envia templates para até 5 números cadastrados no painel | Sandbox com número compartilhado (EUA); cada pessoa manda `join <código>`; **expira em 3 dias**; só templates prontos da Twilio; **pode estar bloqueado para o Brasil**; trial com 100 mensagens | Trials comerciais, com contato de vendas | Grátis |
| Produção | Número próprio + cartão na Meta; ~R$ 0,04 por lembrete | Preço da Meta + ~US$ 0,005 por mensagem; número próprio | Preço da Meta + mensalidade | Grátis |
| Templates próprios | Sim (painel ou API) | Sim, depois de registrar um número próprio | Sim | Não precisa |
| Riscos | Burocracia da Meta (portfólio, nome, cartão) | Sandbox frágil para o Brasil | Custo e contrato, nada a mais para a demo | **Viola os termos do WhatsApp**, o número pode ser banido, quebra a cada atualização |
| Na VM | Só HTTP de saída + webhook | Igual | Igual | Um contêiner a mais, sessão do celular presa à VM |

## Conclusão

- **Caminho recomendado: Meta Cloud API direto**, como a parte 2 fez com o SMTP do Gmail: sem intermediário, grátis na demo, e o mesmo código serve para produção trocando só o número e o token (variáveis de ambiente).
- **Na demo**, o número de teste da Meta basta: o apresentador cadastra o próprio celular no painel (até 5 números) e recebe os templates de verdade. Lembra as contas de teste da parte 2, cujo "E-mail de envio" leva os e-mails para a caixa do apresentador.
- **Para pacientes reais**, falta o que não é código: um chip dedicado, o nome de exibição aprovado e um cartão na conta da Meta. O custo é baixo (centavos por lembrete), mas é uma decisão do autor.
- **Twilio descartada:** o sandbox expira em 3 dias, aceita só templates prontos e pode não entregar no Brasil. **Bibliotecas não oficiais descartadas:** violam os termos e podem derrubar o número no meio da apresentação.
- **Desenho que as regras sugerem:**
  - lembretes como template utility, com variáveis curtas (clínica, data, hora, endereço) e um botão de link para a consulta no app;
  - o resumo da triagem não cabe bem num template (variáveis sem quebra de linha). Opções: um template curto com o essencial e o link, ou um template com um botão "Ver resumo" que, ao ser tocado, abre a janela de 24 h e deixa a API mandar o resumo completo como mensagem livre (grátis dentro da cota de 1.000/mês);
  - o número do paciente precisa estar em formato internacional (`+55...`) e com opt-in. O campo `phone` do Perfil hoje é texto livre e não é confirmado;
  - o canal já existe no modelo dos lembretes (`appointment_reminders.channel`, hoje só `EMAIL`), então o WhatsApp entra como mais um valor.

## Fontes

- Meta: [Pricing on the WhatsApp Business Platform](https://developers.facebook.com/documentation/business-messaging/whatsapp/pricing) (atualizada em 30/09/2026)
- Meta: [Cloud API Get Started](https://developers.facebook.com/documentation/business-messaging/whatsapp/get-started) (token permanente com usuário do sistema)
- Meta: [About the platform](https://developers.facebook.com/documentation/business-messaging/whatsapp/about-the-platform) (recursos de teste, opt-in)
- Meta: [Messaging limits](https://developers.facebook.com/documentation/business-messaging/whatsapp/messaging-limits) e [Business phone numbers](https://developers.facebook.com/documentation/business-messaging/whatsapp/business-phone-numbers/phone-numbers)
- Twilio: [Test WhatsApp messaging with the Sandbox](https://www.twilio.com/docs/whatsapp/sandbox)
- Preço do utility no Brasil em BRL e a cobrança dentro da janela desde 01/10/2026: [AgentDeals](https://agentdeals.dev/whatsapp-business-api-pricing-2026), [Reiko](https://reikosoftware.com.br/en/blog/whatsapp-api-pricing-october-2026)
- O limite de 5 destinatários do número de teste aparece no painel "API Setup" da Meta e não está nas páginas acima: confirmar ao criar o app.
