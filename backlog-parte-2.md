# Backlog da Parte 2

Itens que as revisões da Parte 2 encontraram e ficaram para depois. Nenhum impede o uso na demo. Origem: a revisão final dos dois repositórios (15/11/2026) e os registros de progresso da implementação (removidos depois desta parte).

## Decididos não corrigir (revisão final, 15/11/2026)

O autor decidiu corrigir só o aviso de "Enviamos um novo código" na tela de confirmar e-mail. Estes ficaram de fora:

- **Código no log quando o envio falha (API):** o assunto dos e-mails de confirmação e de redefinição leva o código ("Seu código AgenDoc: 123456"), e a linha de log de falha (`EmailService`) grava o assunto. Com o SMTP fora do ar, um código válido por 15 minutos vai para os logs. Correção: logar o nome do template em vez do assunto.
- **Sessão antiga presa entre `/chat` e `/confirmar-email` (API + web):** quem tem uma segunda sessão aberta antes de confirmar o e-mail fica com a página recarregando sem parar depois de confirmar na primeira, porque essa sessão guarda o papel `ROLE_UNVERIFIED` enquanto o `/me` lê o banco. Sai limpando os cookies. Correção: no `/me`, regravar o contexto da sessão quando o banco já diz confirmado (sem trocar o id da sessão).
- **STARTTLS não obrigatório (API):** `mail.smtp.starttls.enable` está ligado, mas falta `required`. Se o STARTTLS for retirado no caminho, a senha de app do Gmail vai sem criptografia. Correção: `starttls.required: ${SMTP_STARTTLS:false}`.
- **Lembrete de exemplo com horário que já passou (API):** o exemplo usa o primeiro horário escolhido mesmo se ele já passou. Só acontece chamando a API direto (a web desabilita esses horários).
- **Falhas de lembrete fora da métrica (API):** `ReminderJob` envia direto pelo `MailGateway` e não soma em `agendoc.mail.failures`.
- **Tarefas agendadas numa thread só (API):** um SMTP lento no envio de lembretes atrasa a expiração dos pendentes e a limpeza dos limites. Correção: `spring.task.scheduling.pool.size: 2`.
- **E-mails na fila perdidos no deploy (API):** o `mailExecutor` não espera as tarefas ao desligar.
- **Consultas marcadas antes do deploy sem lembretes (API):** a V9 não criou lembretes para as consultas futuras que já tinham o lembrete ligado.
- **Acessibilidade (web):**
  - o botão de mostrar a senha troca o texto e também usa `aria-pressed` (anuncia o estado duas vezes);
  - o aviso "Escolha ao menos um horário" não está ligado aos chips por `aria-describedby`;
  - não há dica de por que um horário de lembrete está desabilitado.
- **Tela de confirmar e-mail (web):** sem contagem regressiva do "Reenviar código" ao chegar do cadastro (um clique logo em seguida mostra "Aguarde um minuto"); o "Sair" é só um ícone.
- **Admin não confirmado (web):** `homeOf` mandaria um admin não confirmado para `/confirmar-email`, que só aceita pacientes. Não acontece hoje (só o cadastro cria contas não confirmadas).

## Outros itens menores

Considerados aceitáveis para a demo na revisão final.

- **Códigos (API):**
  - dois reenvios ao mesmo tempo podem passar pela espera de 60 s (o limite por hora continua valendo e só o código mais novo funciona);
  - depois de 5 tentativas erradas, a confirmação de e-mail responde "código expirado";
  - uma falha ao montar o e-mail depois de gerar o código obriga a esperar 60 s para reenviar;
  - "Esqueci a senha" e a redefinição demoram um pouco mais para contas que existem (bcrypt e SMTP).
- **Envio (API):** o logo é lido do classpath a cada envio; as versões em texto puro não têm o título; as tabelas `email_codes` e `appointment_reminders` nunca são limpas.
- **Lembretes (API):** as linhas escondidas do lembrete são filtradas pelos rótulos em pt-BR; a busca de lembretes não tem limite inferior; o índice é só por status; a validação dos horários roda depois do limite de agendamentos.
- **"E-mail de envio" sem confirmação:** quem tem a senha das contas de teste pode apontar os e-mails delas (inclusive os códigos de redefinição) para qualquer endereço. Decisão do autor: o campo não exige confirmação.
- **Testes:** caminhos sem teste próprio (conta de teste sem "E-mail de envio" na redefinição, falha de envio no "Esqueci a senha", conta comum já usando `pacienteN@demo.agendoc.dev`); o gancho de pausa do `CapturingMailGateway` ignora o resultado do timeout; o mock MSW não simula a espera do reenvio nem o 403 `EMAIL_NOT_VERIFIED`.
