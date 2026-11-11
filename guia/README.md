# Guia de funcionalidades do AgenDoc

Tutorial do que o AgenDoc já faz, tela por tela, para quem está chegando no grupo.

- **Situação em 11/11/2026:** MVP (cadastro, login e chat com IA) e **Parte 1: Clínicas e agendamento**, no ar desde 09/11/2026. A Parte 2 (e-mail) está em andamento e ainda não aparece aqui ([roadmap](../roadmap.md)).
- **Onde testar:** https://agendoc.duckdns.org (veja [Como testar](#como-testar)).
- **Sobre as imagens:** foram tiradas do app rodando localmente, com os dados de demonstração. As telas do paciente são de celular (390 px, em 3x) e as do painel da clínica são de computador (1440 px, em 2x). Clique em qualquer imagem para abrir em tamanho real.

> **Aviso:** o AgenDoc não substitui atendimento profissional. O assistente dá uma orientação inicial: não diagnostica nem prescreve. Todas as clínicas, médicos e avaliações são **fictícios** (por isso o selo "Dados de demonstração").

## Sumário

- [Visão geral](#visão-geral)
- [Como testar](#como-testar)
- **Paciente**
  1. [Cadastro e login](#1-cadastro-e-login)
  2. [Conversas com o assistente](#2-conversas-com-o-assistente)
  3. [Pré-triagem](#3-pré-triagem)
  4. [Emergência](#4-emergência)
  5. [Buscar clínicas](#5-buscar-clínicas)
  6. [Escolher clínica, dia e horário](#6-escolher-clínica-dia-e-horário)
  7. [Confirmar a consulta](#7-confirmar-a-consulta)
  8. [Consulta agendada ou aguardando a clínica](#8-consulta-agendada-ou-aguardando-a-clínica)
  9. [Minhas consultas](#9-minhas-consultas)
  10. [Adicionar à agenda e como chegar](#10-adicionar-à-agenda-e-como-chegar)
  11. [Remarcar](#11-remarcar)
  12. [Cancelar](#12-cancelar)
  13. [Avaliar a consulta](#13-avaliar-a-consulta)
  14. [Perfil](#14-perfil)
- **Clínica**
  1. [Agenda](#1-agenda)
  2. [Detalhe da consulta](#2-detalhe-da-consulta)
  3. [Solicitações](#3-solicitações)
  4. [Médicos](#4-médicos)
  5. [Horários de atendimento](#5-horários-de-atendimento)
  6. [Bloqueios](#6-bloqueios)
  7. [Dados da clínica](#7-dados-da-clínica)
  8. [Avaliações](#8-avaliações)
- [Regras importantes](#regras-importantes)
- [Bastidores](#bastidores)

## Visão geral

O AgenDoc ajuda a pessoa a entender **que tipo de médico procurar** e a **marcar a consulta** numa clínica perto dela. São dois perfis no mesmo app:

| Perfil | Onde | O que faz |
|---|---|---|
| **Paciente** | `/chat`, `/consultas`, `/perfil` (layout de celular) | Conversa com o assistente, recebe a pré-triagem, busca clínicas, marca, remarca, cancela e avalia consultas |
| **Clínica** | `/clinica` (layout de computador, com menu lateral) | Vê a agenda, responde pedidos, cadastra médicos, horários e bloqueios, edita os dados e vê as avaliações |

Depois do login, quem é paciente vai para `/chat` e quem é clínica vai para `/clinica`. Um perfil não acessa as telas do outro.

A jornada completa:

```mermaid
flowchart LR
    A[Cadastro / login] --> B[Chat com o assistente]
    B --> C{Pré-triagem}
    C -- emergência --> D[Ligar 192 e<br/>pronto atendimentos]
    C -- demais casos --> E[Buscar clínicas<br/>mapa e filtros]
    E --> F[Clínica, dia<br/>e horário]
    F --> G[Confirmar]
    G --> H[Minhas consultas]
    H --> I[Clínica marca<br/>como realizada]
    I --> J[Paciente avalia]
    J --> E
```

O status de uma consulta muda assim:

```mermaid
stateDiagram-v2
    [*] --> CONFIRMED: clínica com confirmação automática
    [*] --> PENDING: clínica que confirma manualmente
    PENDING --> CONFIRMED: clínica aceita
    PENDING --> DECLINED: clínica recusa
    PENDING --> EXPIRED: sem resposta até 2 h antes
    PENDING --> CANCELLED: paciente cancela
    CONFIRMED --> COMPLETED: clínica marca "realizada"
    CONFIRMED --> NO_SHOW: clínica marca "faltou"
    CONFIRMED --> CANCELLED: paciente ou clínica cancela
```

Na tela, os status aparecem como selos: **Confirmada**, **Aguardando a clínica**, **Sem resposta** (pedido atrasado), **Recusada**, **Expirou**, **Cancelada** ou **Cancelada pela clínica**, **Realizada**, **Faltou** e **Encerrada** (consulta cujo horário já passou e que a clínica ainda não marcou como realizada ou falta).

## Como testar

**Em produção** (https://agendoc.duckdns.org):

- **Paciente:** crie uma conta em "Criar conta". Não precisa de e-mail real.
- **Clínica:** cada clínica de demonstração tem um login `<apelido>@demo.agendoc.dev`. A senha é a mesma para todas e fica só no `.env` da VM (não vai para o git); peça a quem cuida do deploy.

| Login | Clínica | Confirmação |
|---|---|---|
| `movimento@demo.agendoc.dev` | Clínica Movimento | automática |
| `coracao-paulista@demo.agendoc.dev` | Instituto Coração Paulista | automática |
| `dermavida@demo.agendoc.dev` | DermaVida | **manual** |
| `bem-viver@demo.agendoc.dev` | Clínica Bem Viver Pinheiros | automática |
| `neurocentro@demo.agendoc.dev` | NeuroCentro | automática |
| `otorrino-cia@demo.agendoc.dev` | Otorrino & Cia | **manual** |
| `visao-clara@demo.agendoc.dev` | Visão Clara Oftalmologia | automática |
| `respira@demo.agendoc.dev` | Respira Pneumologia | automática |
| `endosaude@demo.agendoc.dev` | EndoSaúde Vila Mariana | automática |
| `uroclinica@demo.agendoc.dev` | UroClínica Mariana | **manual** |
| `espaco-mulher@demo.agendoc.dev` | Espaço Mulher e Criança | automática |
| `mente-leve@demo.agendoc.dev` | Mente Leve Psiquiatria | automática |

Dica para a demonstração: marque uma consulta como paciente numa clínica **manual** (ex.: DermaVida, Dermatologista) e depois entre como a clínica para aceitar o pedido.

**Localmente:** siga o README de cada repositório ([`agendoc-api`](https://github.com/Agendoc/agendoc-api#rodando-localmente) e [`agendoc-web`](https://github.com/Agendoc/agendoc-web)). Sem chave de IA, o assistente roda em **modo simulação**: as respostas começam com "⚠️ Modo simulação" e a pré-triagem sai na terceira mensagem. Escrever `#emergencia` numa mensagem gera uma triagem de emergência. Algumas imagens deste guia foram tiradas nesse modo.

---

# Paciente

## 1. Cadastro e login

Na primeira tela a pessoa entra com e-mail e senha ou cria uma conta. O cadastro pede só **nome, e-mail e senha** (8 a 72 caracteres) e já entra no app.

<p align="center">
  <a href="img/paciente/01-login.png"><img src="img/paciente/01-login.png" width="270" alt="Tela de login"></a>
  &nbsp;&nbsp;
  <a href="img/paciente/02-cadastro.png"><img src="img/paciente/02-cadastro.png" width="270" alt="Tela de cadastro preenchida"></a>
</p>

A sessão fica num cookie do servidor (não há token no navegador). O botão **Sair** fica no topo das telas do paciente.

## 2. Conversas com o assistente

Em **Conversas** ficam todas as conversas da pessoa, da mais recente para a mais antiga. O **+** no topo (ou "Nova conversa") abre uma conversa nova.

<p align="center">
  <a href="img/paciente/03-conversas-vazio.png"><img src="img/paciente/03-conversas-vazio.png" width="270" alt="Lista de conversas vazia"></a>
  &nbsp;&nbsp;
  <a href="img/paciente/04-conversas-lista.png"><img src="img/paciente/04-conversas-lista.png" width="270" alt="Lista com duas conversas"></a>
</p>

No chat, a pessoa descreve o que está sentindo e o assistente faz perguntas, uma de cada vez. A resposta aparece **aos poucos** (streaming), como num chat de IA. O rodapé lembra sempre que o assistente não substitui atendimento profissional e mostra o 192. O ícone de lixeira apaga a conversa.

<p align="center">
  <a href="img/paciente/05-chat-nova.png"><img src="img/paciente/05-chat-nova.png" width="270" alt="Conversa nova com a saudação do assistente"></a>
  &nbsp;&nbsp;
  <a href="img/paciente/06-chat-conversa.png"><img src="img/paciente/06-chat-conversa.png" width="270" alt="Conversa em andamento"></a>
</p>

Cada pessoa só vê as próprias conversas. Em produção o assistente usa um modelo de IA via OpenRouter, com um prompt em português que proíbe diagnóstico e prescrição.

## 3. Pré-triagem

Quando já tem informação suficiente, o assistente fecha a conversa com o **cartão de pré-triagem**:

- **Condição provável** (sempre como possibilidade, nunca "você tem X");
- **Especialidade indicada** (do catálogo de 15 especialidades);
- **Urgência**: baixa, média, alta ou emergência;
- **Recomendações básicas**;
- o aviso "Este resultado não é um diagnóstico";
- o botão **Agendar consulta**, que abre a busca já filtrada pela especialidade indicada.

<p align="center">
  <a href="img/paciente/07-chat-triagem.png"><img src="img/paciente/07-chat-triagem.png" width="270" alt="Chat terminando com a pré-triagem"></a>
  &nbsp;&nbsp;
  <a href="img/paciente/08-cartao-triagem.png"><img src="img/paciente/08-cartao-triagem.png" width="270" alt="Cartão de pré-triagem em detalhe"></a>
</p>

A triagem também guarda um **resumo dos sintomas** (uma ou duas frases). Esse resumo só vai para a clínica se o paciente marcar a opção na hora de confirmar a consulta (veja o [passo 7](#7-confirmar-a-consulta)).

## 4. Emergência

Se a urgência for **emergência**, o cartão fica vermelho e **não oferece agendamento**. No lugar aparecem:

- o botão **Ligar para o SAMU (192)**;
- os **3 pronto atendimentos 24 h mais próximos**, com distância e link de rota direto para o Google Maps.

<p align="center">
  <a href="img/paciente/09-emergencia.png"><img src="img/paciente/09-emergencia.png" width="300" alt="Cartão de emergência com 192 e pronto atendimentos"></a>
</p>

## 5. Buscar clínicas

A busca abre com o **mapa primeiro** e a lista de clínicas deslizando por cima. Vindo do chat, ela já mostra o selo **"Indicado na pré-triagem: Clínico Geral"**.

- **Pins** no mapa com a nota e o próximo horário de cada clínica, e o pin **"Você"** com a localização da pessoa.
- **Filtros:** convênio, dia (hoje ou amanhã), período (manhã, tarde, noite), distância (até 5 km) e nota 4+.
- **Ordenação:** mais cedo, mais perto ou melhor nota.
- **Cartão da clínica:** nota e número de avaliações, distância, endereço, próximo horário, convênios aceitos e preço particular "a partir de".

<p align="center">
  <a href="img/paciente/10-busca-mapa.png"><img src="img/paciente/10-busca-mapa.png" width="270" alt="Busca com mapa"></a>
  &nbsp;&nbsp;
  <a href="img/paciente/11-busca-lista.png"><img src="img/paciente/11-busca-lista.png" width="270" alt="Busca com a lista aberta"></a>
</p>

A localização vem do navegador (com permissão). Se a pessoa recusar, o app usa o endereço do [Perfil](#14-perfil); sem nenhum dos dois, o mapa centraliza em São Paulo. O mapa usa Leaflet com os tiles da Esri e a lista funciona sozinha, sem o mapa.

## 6. Escolher clínica, dia e horário

A página da clínica mostra nota (com link para as avaliações), endereço, distância, preço particular e os botões **Ligar**, **WhatsApp** e **Rota**. Abaixo:

- uma **faixa de dias** (até 14 dias; fins de semana sem atendimento ficam desabilitados);
- os **horários livres de cada médico** da especialidade, com nome, CRM, duração e preço, separados em manhã, tarde e noite.

Ao tocar num horário, aparece o botão fixo **"Continuar · qui, 12 nov · 14:20"**.

<p align="center">
  <a href="img/paciente/12-clinica.png"><img src="img/paciente/12-clinica.png" width="270" alt="Página da clínica com dias e horários"></a>
  &nbsp;&nbsp;
  <a href="img/paciente/13-clinica-horario.png"><img src="img/paciente/13-clinica-horario.png" width="270" alt="Horário selecionado"></a>
</p>

Os horários livres são **calculados**: horários semanais do médico, menos bloqueios, menos consultas já marcadas.

## 7. Confirmar a consulta

A última etapa (passo 4 de 4) revisa tudo:

- **Resumo:** médico, clínica, data, hora, CRM, duração e endereço.
- **Pagamento:** convênio (só os que a clínica aceita, com carteirinha opcional) ou particular, com o valor.
- **Pré-triagem:** a caixa "Compartilhar o resumo da pré-triagem com a clínica" vem **desmarcada** (consentimento da LGPD). Vão só os sintomas relatados, a condição provável e a urgência; **a conversa nunca é enviada**.
- **Lembretes:** e-mail (o envio chega na Parte 2) e WhatsApp "Em breve".
- O botão diz **"Confirmar agendamento"** (clínica automática) ou **"Solicitar horário"** (clínica manual).

<p align="center">
  <a href="img/paciente/14-confirmar.png"><img src="img/paciente/14-confirmar.png" width="300" alt="Tela de confirmação"></a>
</p>

Se outra pessoa pegar o mesmo horário no mesmo instante, a segunda recebe a mensagem de que o horário acabou de ser ocupado e a grade atualizada.

## 8. Consulta agendada ou aguardando a clínica

Depois de confirmar, a tela mostra:

- **"Consulta agendada"** (clínica automática) ou **"Aguardando a clínica"** com o prazo de resposta, por exemplo "Responde até qui 18:00" (clínica manual);
- **Adicionar à agenda** e **Como chegar**;
- **Antes da consulta:** as orientações fixas da clínica mais as recomendações da pré-triagem;
- **Se piorar:** sinais de alerta e o 192;
- **Pré-triagem que originou**, com link para a conversa;
- **Remarcar** e **Cancelar consulta**.

<p align="center">
  <a href="img/paciente/15-confirmado.png"><img src="img/paciente/15-confirmado.png" width="250" alt="Consulta agendada"></a>
  &nbsp;
  <a href="img/paciente/16-aguardando-clinica.png"><img src="img/paciente/16-aguardando-clinica.png" width="250" alt="Pedido aguardando a clínica"></a>
  &nbsp;
  <a href="img/paciente/17-pedido-aceito.png"><img src="img/paciente/17-pedido-aceito.png" width="250" alt="Pedido aceito pela clínica"></a>
</p>

Da esquerda para a direita: consulta confirmada na hora, pedido aguardando a clínica e o mesmo pedido depois que a clínica aceitou.

## 9. Minhas consultas

A aba **Consultas** tem duas partes:

- **Nova consulta:** atalhos por especialidade ("Ver todas as especialidades" mostra as 15), para quem quer marcar sem passar pelo chat.
- **Minhas consultas:** abas **Próximas** e **Anteriores**, com o selo de status em cada cartão e atalhos para "Como chegar" e "Remarcar".

<p align="center">
  <a href="img/paciente/18-minhas-consultas.png"><img src="img/paciente/18-minhas-consultas.png" width="250" alt="Próximas consultas"></a>
  &nbsp;
  <a href="img/paciente/19-especialidades.png"><img src="img/paciente/19-especialidades.png" width="250" alt="Lista de especialidades"></a>
  &nbsp;
  <a href="img/paciente/25-anteriores.png"><img src="img/paciente/25-anteriores.png" width="250" alt="Consultas anteriores"></a>
</p>

Em **Anteriores**, a consulta realizada e ainda sem avaliação mostra as estrelas "Como foi?".

## 10. Adicionar à agenda e como chegar

- **Adicionar à agenda:** **Google Agenda** (abre o evento já preenchido, sem baixar arquivo) ou **iPhone, Outlook e outros** (arquivo `.ics`). A triagem nunca vai para o calendário.
- **Como chegar:** abre o app de mapas da pessoa (Google Maps ou Waze; Apple Mapas em iPhone e Mac), saindo da localização atual.

<p align="center">
  <a href="img/paciente/20-adicionar-agenda.png"><img src="img/paciente/20-adicionar-agenda.png" width="270" alt="Menu Adicionar à agenda"></a>
  &nbsp;&nbsp;
  <a href="img/paciente/21-como-chegar.png"><img src="img/paciente/21-como-chegar.png" width="270" alt="Menu Como chegar"></a>
</p>

## 11. Remarcar

Mostra o **horário atual** e a grade de horários da **mesma clínica e especialidade** (pode ser com outro médico). Trocar é uma operação só: se o novo horário falhar, o antigo continua valendo.

<p align="center">
  <a href="img/paciente/22-remarcar.png"><img src="img/paciente/22-remarcar.png" width="270" alt="Tela de remarcar"></a>
</p>

Em clínica manual, o novo horário vira um pedido e o antigo **continua confirmado** até a clínica aceitar. Se ela recusar ou não responder, a consulta original continua valendo.

## 12. Cancelar

O cancelamento pede confirmação na própria tela ("O horário fica livre para outra pessoa"). Depois, a consulta aparece como **Cancelada** e vai para Anteriores.

<p align="center">
  <a href="img/paciente/23-cancelar.png"><img src="img/paciente/23-cancelar.png" width="270" alt="Confirmação de cancelamento"></a>
  &nbsp;&nbsp;
  <a href="img/paciente/24-cancelada.png"><img src="img/paciente/24-cancelada.png" width="270" alt="Consulta cancelada"></a>
</p>

Paciente só cancela ou remarca **até 2 h antes**. Depois disso, a tela orienta a falar com a clínica. Quando é a **clínica** que cancela, o paciente vê o motivo e o botão "Escolher novo horário".

## 13. Avaliar a consulta

Depois que a clínica marca a consulta como **realizada**, o paciente pode dar de **1 a 5 estrelas** e deixar um comentário opcional (até 500 caracteres). A avaliação é **anônima** ("Paciente verificado · mês/ano"), não pode ser editada e já entra na nota da clínica.

Na página da clínica, o link **Avaliações** mostra a média e os comentários.

<p align="center">
  <a href="img/paciente/26-avaliar.png"><img src="img/paciente/26-avaliar.png" width="270" alt="Formulário de avaliação"></a>
  &nbsp;&nbsp;
  <a href="img/paciente/27-avaliacoes-clinica.png"><img src="img/paciente/27-avaliacoes-clinica.png" width="270" alt="Avaliações de uma clínica"></a>
</p>

## 14. Perfil

Nome (só leitura), **telefone**, **endereço** e **convênio** com o número da carteirinha.

- O endereço vira coordenadas no servidor (Nominatim/OpenStreetMap) e serve para medir distâncias quando a pessoa não compartilha a localização.
- O convênio salvo já vem escolhido na busca e na confirmação.
- O telefone aparece para a clínica nas consultas marcadas com ela.

<p align="center">
  <a href="img/paciente/28-perfil.png"><img src="img/paciente/28-perfil.png" width="270" alt="Perfil do paciente"></a>
</p>

---

# Clínica

O painel fica em `/clinica`, com menu lateral: Agenda, Solicitações, Médicos, Horários de atendimento, Bloqueios, Dados da clínica e Avaliações. A clínica só vê os próprios dados e só vê dados de pacientes que marcaram consulta com ela. **Nunca vê o chat do paciente.**

## 1. Agenda

A semana inteira numa grade, com uma cor por médico. Dá para filtrar por médico, andar entre semanas e voltar para hoje. A legenda explica os estados: confirmada, aguardando você, prazo vencido, remarcação, livre, fechado e bloqueado.

[![Agenda da semana](img/clinica/01-agenda.png)](img/clinica/01-agenda.png)

## 2. Detalhe da consulta

Ao clicar numa consulta, o painel da direita mostra o paciente (nome, telefone com Ligar e WhatsApp, convênio e carteirinha), o médico, o horário e, **se o paciente compartilhou**, o resumo da pré-triagem com o aviso "Gerado por IA, não é diagnóstico".

[![Detalhe da consulta com o resumo da pré-triagem](img/clinica/02-detalhe-consulta.png)](img/clinica/02-detalhe-consulta.png)

Ações disponíveis:

- **Cancelar consulta** (antes do horário, com motivo obrigatório, que o paciente vê);
- **Marcar como realizada** ou **Faltou** (só depois que a consulta começou).

[![Consulta passada com Marcar como realizada e Faltou](img/clinica/03-marcar-realizada.png)](img/clinica/03-marcar-realizada.png)

## 3. Solicitações

Só aparece movimento aqui nas clínicas com **confirmação manual**. É a lista de pedidos de horário e de remarcação que esperam resposta, com o prazo de cada um. O menu mostra quantos estão aguardando.

[![Solicitações pendentes com Aceitar e Recusar](img/clinica/04-solicitacoes.png)](img/clinica/04-solicitacoes.png)

Regra do prazo: a clínica deve responder **até o fim do próximo dia de atendimento do médico**. Depois disso o pedido fica "atrasado" (com destaque vermelho), mas ainda pode ser aceito. Ele só **expira 2 h antes da consulta**. Assim, um pedido feito na sexta à noite não se perde no fim de semana.

## 4. Médicos

Lista dos médicos com CRM, situação (ativo ou inativo), especialidades e carga semanal. Cada médico pode ter **de 1 a 5 especialidades**, cada uma com **duração e preço particular próprios** (ex.: Cardiologista 30 min por R$ 300,00 e Clínico Geral 20 min por R$ 150,00).

[![Lista de médicos](img/clinica/05-medicos.png)](img/clinica/05-medicos.png)

[![Edição de médico com duas especialidades](img/clinica/06-medico-editar.png)](img/clinica/06-medico-editar.png)

A agenda do médico é uma só: consultas de especialidades diferentes nunca se sobrepõem. Desativar um médico tira ele da busca, mas não apaga as consultas.

## 5. Horários de atendimento

Os horários semanais de cada médico: um ou mais períodos por dia (ex.: 07:30 às 12:00 e 13:00 às 19:00), com uma "visão do dia" em barras. Há atalhos para **copiar segunda para os dias úteis** e **limpar a semana**, e um resumo com dias, horas por semana, primeiro e último horário.

[![Horários de atendimento](img/clinica/07-horarios.png)](img/clinica/07-horarios.png)

Os horários livres que o paciente vê saem daqui, menos os bloqueios e as consultas marcadas.

## 6. Bloqueios

Folgas, férias, feriados, congressos e reuniões. O bloqueio vale para a **clínica inteira** ou para **um médico**, por dias inteiros ou por um intervalo de horas. A lista separa próximos e passados.

[![Lista de bloqueios](img/clinica/08-bloqueios.png)](img/clinica/08-bloqueios.png)

Se o período cair sobre consultas já marcadas, o formulário avisa quantas são e quem são, e oferece **cancelar todas e avisar os pacientes** com um motivo.

[![Aviso de consultas no período do bloqueio](img/clinica/09-bloqueio-conflito.png)](img/clinica/09-bloqueio-conflito.png)

## 7. Dados da clínica

O que os pacientes veem na busca e na consulta:

- **Contatos:** telefone e WhatsApp;
- **Convênios aceitos** (particular é sempre aceito);
- **Como os pedidos são confirmados:** automática (confirma na hora) ou "a clínica confirma" (vira pedido em Solicitações);
- **Orientações para o paciente** antes da consulta (até 10, em ordem).

Nome, endereço e localização no mapa são definidos pela equipe do AgenDoc.

[![Dados da clínica](img/clinica/10-dados-clinica.png)](img/clinica/10-dados-clinica.png)

## 8. Avaliações

Somente leitura: a nota média, o total e os comentários, **anônimos** como o paciente vê.

[![Avaliações da clínica](img/clinica/11-avaliacoes.png)](img/clinica/11-avaliacoes.png)

---

## Regras importantes

| Regra | Como funciona |
|---|---|
| Janela para marcar | De agora + 1 h até 30 dias à frente. Em clínica manual, o primeiro horário oferecido fica pelo menos 4 h de expediente depois do pedido |
| Cancelar e remarcar (paciente) | Até 2 h antes da consulta |
| Limites por paciente | No máximo 5 consultas ativas e 10 ações (marcar, remarcar, cancelar) por hora |
| Horário duplicado | Impossível: a garantia fica no banco (trava por médico), não só na tela |
| Consultas sobrepostas | A mesma pessoa não pode ter duas consultas ativas no mesmo horário, nem em clínicas diferentes |
| Pedido sem resposta | Fica "atrasado" depois do prazo e só expira 2 h antes da consulta |
| Consulta que já passou | Aparece como "Encerrada" até a clínica marcar realizada ou falta |
| Avaliação | Só de consulta realizada, só pelo próprio paciente, uma vez, anônima |
| Pré-triagem para a clínica | Só com o consentimento marcado na confirmação; vai uma cópia do resumo, nunca o chat |
| Isolamento | Paciente só vê o que é dele; clínica só vê o que é dela. Recurso de outro dono responde 404 |
| Fuso | Tudo exibido em horário de São Paulo e guardado em UTC |

## Bastidores

Coisas que não têm tela, mas fazem parte do que foi entregue:

- **Criação de clínicas:** endpoint de admin (`POST /api/admin/clinics`, documentado no Swagger), que cria a clínica e o login dela e devolve a senha inicial uma única vez. Não há tela de admin.
- **Dados de demonstração:** 12 clínicas fictícias em São Paulo (Paulista, Pinheiros e Vila Mariana), cobrindo as 15 especialidades, mais 4 pronto atendimentos 24 h e avaliações iniciais. CRMs no formato `DEMO-0000` para não parecerem reais.
- **Expiração automática:** um job muda para "Expirou" os pedidos não respondidos até 2 h antes da consulta.
- **Stack:** API em Java 21 + Spring Boot 4 + MySQL ([`agendoc-api`](https://github.com/Agendoc/agendoc-api)) e web em Next.js 16 + React 19 ([`agendoc-web`](https://github.com/Agendoc/agendoc-web)). O contrato completo da API está no README da API e no Swagger.
- **Deploy:** merge na `main` dispara o CI, que publica na VM sozinho. Detalhes em [`agendoc-api/docs/deploy.md`](https://github.com/Agendoc/agendoc-api/blob/main/docs/deploy.md).
- **Mais detalhes de produto:** [spec da Parte 1](../specs/2026-11-01-clinics-scheduling-design.md), [redesenho do painel](../specs/2026-11-08-clinic-panel-v2-design.md) e [roadmap](../roadmap.md).
