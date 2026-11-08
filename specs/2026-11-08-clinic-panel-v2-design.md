# AgenDoc: painel da clínica v2, prazo dos pedidos e médico com várias especialidades: design

- **Data:** 2026-11-08
- **Status:** em revisão
- **Parte de:** Parte 1, Marco 3 (ainda não aprovado). Complementa [2026-11-01-clinics-scheduling-design.md](2026-11-01-clinics-scheduling-design.md), que continua valendo onde este documento não fala.
- **Marcação:** **[autor]** = decidido pelo dono do produto nesta conversa. **[builder]** = escolha do builder, que o autor pode mudar.

## Intenção

O autor testou o painel da clínica (Marco 3) com dados de demonstração e achou três problemas.

1. **O visual está cru.** A agenda tem uma barra de filtros feia, setas que parecem travadas, um detalhe de consulta que parece HTML puro e bloqueios numa lista solta embaixo. As telas de horários, bloqueios e convênios são difíceis de ler. Quem usa o painel é a recepcionista, o dia inteiro: precisa bater o olho e entender.
2. **A regra de expiração castiga o paciente.** Hoje um pedido `PENDING` expira 24 h depois de criado. Um paciente que pede na sexta à noite perde o pedido no sábado, porque a clínica só abre na segunda, e precisa remarcar.
3. **O médico só pode ter uma especialidade.** Na vida real, um cardiologista também atende como clínico geral, com preço e duração diferentes.

Sucesso: a recepcionista entende a semana numa olhada; o paciente não perde pedido por causa de fim de semana; a clínica cadastra o médico como ele realmente atende.

## Escopo

### Dentro
1. Nova regra de prazo para pedidos `PENDING`: prazo de resposta, estado "atrasado" e expiração só perto da consulta.
2. Médico com uma ou mais especialidades, cada uma com preço particular e duração próprios.
3. Redesenho das telas do painel: Agenda (com o detalhe da consulta), Solicitações, Médicos, Horários, Bloqueios e Dados da clínica, seguindo os mockups aprovados.
4. Os ajustes do lado do paciente que essas regras exigem: busca e horários por especialidade, prazo e estado "atrasado" na consulta.
5. Mocks MSW e testes acompanhando.

### Fora
- **Atendimento de madrugada e plantão [autor]:** um período continua dentro de um dia (00:00 a 23:59), sem cruzar a meia-noite. Fica para depois.
- A clínica criar consulta pela agenda (clicar num horário livre).
- Avaliações: a tela continua como está **[autor: "Ok"]**.
- Mudanças visuais nas telas do paciente além das citadas no item 4.

### Anti-objetivos
- Perder ou mudar o significado de consultas já existentes. As migrações continuam **só aditivas**.
- Deixar dois horários do mesmo médico se sobreporem por causa das durações diferentes.
- Expirar ou recusar pedido sem o paciente entender o que aconteceu e o que fazer.

## Regra 1: prazo dos pedidos [autor]

**Prazo de resposta (`respondBy`):** é o **fim do primeiro dia de atendimento do médico que começa depois do pedido**, limitado a **2 h antes da consulta**. Vale o que vier primeiro.

- "Dia de atendimento" é um dia em que o médico tem regra semanal. O fim do dia é o horário final do último período desse dia.
- "Começa depois do pedido" significa que o primeiro período do dia começa depois do momento do pedido. Exemplos:
  - pedido na sexta às 22h vale até segunda no fim do expediente;
  - pedido na segunda às 9h, com a clínica já aberta, vale até terça no fim do expediente;
  - pedido na segunda às 7h, antes de abrir, vale até segunda no fim do expediente.
- Dias **totalmente cobertos por bloqueio** (do médico ou da clínica inteira, como um feriado) não contam e são pulados **[builder]**.
- O prazo é **calculado e guardado na criação do pedido** **[builder]**. Mudar horários ou bloqueios depois não mexe nos pedidos já feitos, e o paciente vê sempre a mesma data.
- Se o médico não tiver nenhum dia de atendimento nos próximos 30 dias, o prazo cai para "2 h antes da consulta" **[builder]**.
- Remarcação em clínica `MANUAL`: o pedido de remarcação segue a mesma regra.

**Atrasado [autor]:** passado o `respondBy`, o pedido **continua `PENDING`** e é marcado como atrasado. Esse estado é derivado (`agora > respondBy`) e não é um status novo.
- A clínica ainda pode aceitar ou recusar.
- No painel, o pedido aparece com "!" vermelho e "prazo vencido", no topo de Solicitações.
- O paciente vê "A clínica ainda não respondeu" e um botão "Escolher outro horário". Esse botão cancela o pedido e abre os horários da mesma clínica e especialidade **[builder]**.
- No caso de um **pedido de remarcação** atrasado, o botão cancela **só o pedido**. A consulta original continua `CONFIRMED`, e o paciente começa uma nova remarcação dela. É o mesmo comportamento de quando a clínica recusa **[builder]**.

**Expiração [autor]:** o pedido só vira `EXPIRED` **2 h antes do horário da consulta**. O job atual muda de critério. Os pedidos antigos sem `respondBy` seguem a regra nova; para eles, o prazo exibido é "criado + 24 h" **[builder]**.

**Horários oferecidos em clínica `MANUAL` [autor]:** entre o pedido e a consulta precisa haver pelo menos **4 horas de expediente do médico**. Só conta o tempo dentro das regras semanais dele, e os dias totalmente bloqueados são pulados, como no prazo.
- Pedido na sexta às 22h: as horas começam a contar na segunda às 8h, e o primeiro horário oferecido é segunda às 12h.
- Pedido na segunda às 9h: o primeiro horário oferecido é segunda às 13h.

Como o prazo é limitado a 2 h antes da consulta, a clínica sempre tem pelo menos 2 horas de expediente para responder. As clínicas `AUTOMATIC` continuam com a janela atual (agora + 1 h). A remarcação usa o mesmo filtro.

**Telas do paciente:**
- Em "Aguardando a clínica", aparece "A clínica responde até seg 18:00".
- No estado atrasado, aparecem o aviso e a ação descritos acima.
- O pedido expirado mantém o aviso atual.

## Regra 2: médico com várias especialidades [autor]

- **Cadastro:** um médico tem **uma ou mais especialidades**. Cada uma tem **preço particular** e **duração da consulta** próprios **[autor]**.
  - As especialidades vêm do catálogo existente (`Specialties.ALL`).
  - O nome, o CRM e a UF, a situação (ativo ou inativo), as regras semanais e os bloqueios continuam sendo do médico.
- **Uma agenda só [autor]:** uma consulta de qualquer especialidade ocupa o tempo do médico. Duas consultas do mesmo médico **nunca se sobrepõem**, nem com durações diferentes. Hoje a garantia no banco é o índice único por início. Como inícios diferentes passam a poder se sobrepor, a garantia vira **trava transacional**: a linha do médico já é travada antes de marcar e remarcar, e, dentro da trava, a aplicação verifica a sobreposição com as consultas ativas do médico. O índice atual continua existindo.
- **Horários livres por especialidade:** a grade de cada (médico, especialidade) começa no início de cada período e anda de acordo com a duração daquela especialidade. Dela saem:
  - os horários que se sobrepõem a qualquer consulta ativa do médico;
  - os horários que caem em bloqueios;
  - a janela de agendamento;
  - o filtro de clínica `MANUAL`.
- **Busca:**
  - quem busca uma especialidade encontra os médicos que a atendem, com o preço e a duração dela;
  - a nota e a distância continuam sendo da clínica;
  - o cartão da clínica mostra o menor preço entre os médicos daquela especialidade **[builder]**.
- **Consulta:** guarda a especialidade escolhida, numa coluna nova.
  - O fim da consulta usa a duração dessa especialidade.
  - O comando de marcar recebe a especialidade. Ela é obrigatória quando o médico tem mais de uma; com uma só, é assumida **[builder]**.
  - A remarcação mantém a especialidade e pode trocar para outro médico da mesma clínica que a atenda.
  - O painel, a lista "Minhas consultas" e o detalhe mostram "Médico · Especialidade".
- **Migração (só aditiva):**
  - nova tabela de especialidades do médico (médico, especialidade, preço, duração), com uma linha por médico existente copiada das colunas atuais;
  - nova coluna de especialidade na consulta, preenchida nas linhas antigas com a especialidade do médico.
  - As colunas antigas do médico ficam no banco. Elas continuam preenchidas com a primeira especialidade, para nada quebrar, mas **deixam de ser a fonte** **[builder]**.
- **Limites [builder]:**
  - até 5 especialidades por médico;
  - sem repetir a mesma especialidade;
  - duração de 10 a 120 min, de 5 em 5;
  - preço de 0 a R$ 10.000.

## Regra 3: categoria do bloqueio [builder]

O bloqueio ganha uma categoria: `VACATION` (Férias), `DAY_OFF` (Folga), `CONGRESS` (Congresso), `HOLIDAY` (Feriado), `MEETING` (Reunião) ou `OTHER` (Outro).
- É uma coluna nova, com padrão `OTHER` para os bloqueios que já existem.
- O motivo em texto continua opcional.
- Os botões de motivo pronto no formulário escolhem a categoria e preenchem o texto, que pode ser editado.

## Telas do painel

As referências visuais são os mockups aprovados em `assets/`: `panel-agenda.html`, `panel-hours.html`, `panel-blocks.html` e `panel-doctors-settings.html`. Eles mostram a intenção. Na implementação entram os ícones do projeto (lucide), não os emojis dos mockups, e os tokens de design que já existem **[builder]**.

### Comum
- **Cor por médico [builder]:** cada médico tem uma cor fixa, escolhida numa paleta pela ordem de cadastro. A cor não é guardada no banco e se repete na agenda, nos filtros, em Médicos, em Horários e em Bloqueios. Médico inativo fica cinza.
- **Seleção de médico:** etiquetas clicáveis com a cor, no lugar do `<select>` **[autor]**.
- Botões com estado visível de hover, foco e clique. Toda navegação que carrega dados mostra que está carregando **[autor: "as setas parecem travadas"]**.

### Agenda [autor] (`panel-agenda.html`, versão final "v3")
- **Barra do topo:**
  - semana com setas e "Hoje";
  - período escrito por extenso ("9 – 15 de novembro · 2026");
  - etiquetas de médico, com **"Todos os médicos" como padrão ao entrar** **[autor]**;
  - atalho "N pedidos aguardando resposta", que leva a Solicitações.
- **Calendário em grade de horas:**
  - cada consulta tem a altura da duração dela;
  - horário de atendimento em branco e fora dele em cinza;
  - o intervalo de horas mostrado vai do primeiro ao último horário dos médicos visíveis **[builder]**;
  - dia sem nenhum atendimento fica com a coluna estreita;
  - consultas que caem no mesmo horário (médicos diferentes, em "Todos") ficam lado a lado **[builder]**;
  - os horários livres não aparecem um a um, porque as durações mudam por especialidade; o fundo branco já mostra quando o médico atende **[builder]**.
- **Cartão da consulta:**
  - padrão único: **nome do paciente primeiro** **[autor]**;
  - o ícone de situação vem antes do nome: "?" aguardando, "!" prazo vencido, "↻" remarcação;
  - de 30 min para cima, o horário aparece na segunda linha; abaixo disso, só o nome;
  - nome longo termina em "…" e aparece completo ao passar o mouse;
  - nada pode sair cortado **[autor]**.
- **Bloqueios dentro do calendário [autor]:** faixa hachurada no próprio dia, com o médico (ou "Clínica inteira") e o motivo. A lista "Bloqueios nesta semana" sai da agenda.
- **Legenda no topo do calendário [autor]:** confirmada (cor do médico), aguardando você, prazo vencido, remarcação, livre, fechado e bloqueado.
- **Detalhe da consulta [autor]:**
  - **começa alinhado com o topo do calendário e acompanha a rolagem**, preso na tela;
  - mostra as iniciais do paciente, o nome e o status;
  - se for pedido, mostra o prazo ("Responda até qui 12/11 08:20");
  - linhas com ícone: data e horário com duração, médico e especialidade, pagamento com valor ou convênio e carteirinha, telefone com "Ligar" (`tel:`) e "WhatsApp" (`wa.me`);
  - a pré-triagem num cartão com o nível de urgência e o aviso de IA;
  - as ações presas embaixo: Aceitar/Recusar, Realizada/Faltou, ou Cancelar com motivo.
- **Celular:** a grade rola na horizontal dentro do cartão e o detalhe vai para baixo dela **[builder]**.

### Solicitações
- Mesmo visual do detalhe da agenda.
- Lista ordenada com os **atrasados primeiro** e depois pelo prazo mais próximo **[builder]**.
- Cada pedido mostra "Responder até…" ou "prazo vencido".
- O texto que explica a regra na tela passa a descrever a Regra 1.

### Médicos [autor] (`panel-doctors-settings.html`, parte de cima)
- **Cartões de médico:**
  - avatar com iniciais na cor do médico, nome, CRM-UF e situação;
  - uma linha por especialidade (especialidade · duração · preço);
  - o resumo da semana ("seg a sex · 40 h/semana").
- **Edição ao lado do cartão:**
  - nome, CRM, UF e Ativo/Inativo em botões segmentados;
  - uma tabela de especialidades (especialidade · duração · preço) com "+ Adicionar especialidade".
- **Espaço maior entre o botão "+ Novo médico" e os cartões [autor].**

### Horários [autor] (`panel-hours.html`, versão final "v3")
- **Tabela** com colunas fixas: Dia | 1º período | 2º período | + período | Visão do dia. Todos os períodos têm a mesma largura e ficam alinhados entre os dias.
  - Se algum dia tiver um 3º período, a coluna aparece para todos.
  - O máximo é de 4 períodos por dia **[builder]**.
- **Ladrilho do dia** (SEG, TER…): preenchido com ✓ na cor do médico quando ele atende, tracejado quando não atende. Um clique alterna.
- **Edição do período** dentro da própria célula, com dois seletores em 24 h, de 15 em 15 min, sem AM/PM e sem empurrar o layout.
- **Erros:**
  - sobreposição ou início depois do fim aparecem na linha do dia;
  - o período com problema fica vermelho.
- **Atalhos:** "Copiar segunda para os dias úteis" e "Limpar semana".
- **Abaixo da tabela:** o resumo da semana (dias, horas, primeiro e último horário) e as especialidades do médico com duração e preço.
- **Barra de salvar presa no rodapé,** com "Alterações não salvas", "Descartar" e "Salvar horários". Trocar de médico com alterações não salvas pede confirmação dentro da interface **[builder]**.

### Bloqueios [autor] (`panel-blocks.html`, versão final "v2")
- **Três zonas em cada bloqueio:**
  - **quando:** calendarinho com dia da semana, data e mês (num período de vários dias, "SEG–QUA / 09–11 / NOV", sem corte) e o horário em negrito com a duração, ou "Dia inteiro";
  - **quem:** etiqueta na cor do médico, ou escura para "Clínica inteira";
  - **por quê:** etiqueta da categoria mais o texto, ou "Sem motivo informado".
  - A borda esquerda tem a cor de quem está bloqueado. À direita ficam "começa amanhã", "em 3 dias" ou "agora", e a lixeira.
- **Organização da lista:**
  - agrupada por mês;
  - abas "Próximos" e "Passados" (os que terminaram nos últimos 90 dias) **[builder]**;
  - filtro por médico ou "Clínica inteira".
- **Formulário ao lado,** começando **na altura do primeiro bloqueio** **[autor]**:
  - "Clínica inteira / Um médico";
  - "Dia inteiro";
  - datas e horas;
  - motivos prontos e texto;
  - aviso de consultas atingidas **com a lista** (paciente, horário e status), não só a contagem, e a opção de cancelar todas com motivo.
  - Por isso a API de conflitos passa a devolver a lista **[builder]**.
- O "+" do botão "+ Novo bloqueio" (e dos outros botões com "+") é um ícone alinhado ao texto **[autor]**.

### Dados da clínica [autor] (`panel-doctors-settings.html`, parte de baixo)
- **Convênios:**
  - cartões clicáveis com a sigla em cor da nossa paleta (nunca a cor ou a marca do convênio) **[builder]** e um ✓ quando marcados;
  - "Particular" fixo, como sempre aceito;
  - contador "N de 7 convênios".
- **Modo de confirmação:** dois cartões de escolha. O texto de "A clínica confirma" cita o prazo da Regra 1.
- **Orientações:**
  - lista numerada com reordenação;
  - para arrastar, há a alça; para teclado e leitor de tela, há botões "subir" e "descer" **[builder]**;
  - também é possível remover e adicionar.
- Os outros campos (telefone e WhatsApp) seguem o mesmo estilo de cartão.

## Demonstração em produção [builder]

O seed de demonstração só roda com o banco vazio, então em produção as novidades não apareceriam sozinhas. Para isso, entra uma **migração Flyway de dados, só aditiva**, que mexe apenas nas linhas de demonstração conhecidas:
- acrescenta uma segunda especialidade (com preço e duração próprios) a dois ou três médicos de demonstração;
- dá uma categoria aos bloqueios de demonstração, se houver.

O JSON do seed também é atualizado, para que um banco novo já nasça igual. O estado "atrasado" aparece sozinho com o uso.

## Testes esperados [builder]
- **API, prazo:** sexta à noite → fim de segunda; pedido durante o expediente → fim do dia seguinte de atendimento; feriado da clínica pulado; limite em 2 h antes da consulta; pedido atrasado continua `PENDING` e ainda pode ser aceito; o job só expira 2 h antes; horários de clínica `MANUAL` com 4 horas de expediente de antecedência (sexta à noite → segunda às 12h; segunda às 9h → segunda às 13h).
- **API, especialidades:**
  - grade por especialidade com durações diferentes;
  - nenhuma sobreposição entre especialidades do mesmo médico, inclusive em marcações concorrentes;
  - especialidade obrigatória com mais de uma;
  - remarcação mantém a especialidade;
  - migração dos dados existentes;
  - busca e preço por especialidade.
- **API, bloqueios:** a categoria e a lista de conflitos.
- **Web:**
  - componentes das telas novas;
  - "Todos os médicos" como padrão;
  - cartões sem corte (texto completo no `title`/`aria-label`);
  - edição de período sem AM/PM;
  - aviso de alterações não salvas;
  - estados "atrasado" no painel e no paciente;
  - mocks MSW atualizados.
- **Passada no Chrome** pelas telas, no computador e com 390 px.

## Fica com o builder
- Nomes de tabelas, colunas, DTOs e endpoints, seguindo as convenções do repositório.
- O algoritmo exato de posicionamento lado a lado na grade e a escala de pixels por minuto.
- A paleta de cores dos médicos (precisa ter contraste AA para o texto sobre o fundo claro).
- A divisão em tarefas e a ordem. Sugestão: API primeiro (especialidades → prazo → bloqueios), depois a web, tela por tela.
