# Backlog da Parte 1

Itens menores que as revisões da Parte 1 encontraram e ficaram para depois. Nenhum impede o uso na demo. Origem: os registros de progresso da implementação (removidos depois do deploy de 09/11/2026).

## Estacionados por decisão

Casos raros de concorrência. Decidido em 09/11/2026 não corrigir, porque o app é só de demonstração.

- **Duas remarcações ao mesmo tempo:** o MySQL pode dar deadlock entre as duas, e a perdedora recebe 409.
- **"Desistir da remarcação" numa página antiga, depois que a clínica já aceitou:** o paciente perde as duas consultas. A correção seria mandar o `expectedStatus` no cancelamento.
- **Outros casos raros na API:**
  - timeout de lock (erro 1205) vira 409, quando deveria ser 503;
  - ciclos de lock possíveis ao criar bloqueios e ao expirar pendentes em lote;
  - `cancelConflicts` dos bloqueios grava entidades lidas sem lock, e quem cancelou pode ficar errado;
  - `addressChanged` do perfil é calculado a partir de uma leitura sem lock;
  - o `AddressChangeLimiter` não é atômico.

## Visíveis para o usuário

- **Agenda, filtro "Todos os médicos":** o bloqueio de vários dias de um só médico hachura o dia inteiro e parece a clínica fechada.
- **Cancelamento pela clínica:** aparecem dois cards "Cancelada pela clínica" (questão de produto em aberto).
- **Detalhe da consulta:** mostra "Responde até" num pedido de remarcação pendente, e a lista não mostra.
- **Nota "A clínica ainda não respondeu":** aparece duas vezes (no badge e no aviso).
- **Localização do paciente:** `usePatientLocation` não tem tempo limite ao buscar o perfil.
- **Agendamento:** quando o profissional não existe (404) e há pré-triagem, também aparece o aviso "pré-triagem apagada".
- **Voltar das avaliações:** o link de volta perde o `?especialidade`.
- **Painel, cards da agenda:** consultas de menos de 20 minutos se sobrepõem.
- **Painel, bloqueios:**
  - trocar o modelo de bloqueio apaga o texto que a pessoa já editou;
  - "1 dia" é uma aproximação para bloqueios com horário que passam de um dia;
  - um erro ao carregar os passados substitui a página inteira.
- **Painel, horários:**
  - erros da API aparecem num banner só, e não em cada campo;
  - mais de 4 períodos não mostram mensagem na linha;
  - o `nextInterval` pode criar o período 23:59–23:59.
- **Faixa de datas (`.ad-dates`):** o anel de foco pode ser cortado nas bordas.

## Acessibilidade

- **Avaliações:**
  - o foco vai para o `body` depois de enviar;
  - o `aria-live` já é montado com conteúdo;
  - as estrelas da lista têm 32px.
- **Painel:**
  - o foco não vai para o detalhe ao escolher um pedido, nem para o editor de horários;
  - `.requestItem` não tem `:focus-visible`;
  - na confirmação de exclusão de bloqueio, o foco não é tratado.
- **Formulário de bloqueio:** os textos "Das" e "Às" não estão ligados aos selects (só têm `aria-label`).
- **Chips:** altura mínima de 36px, abaixo dos 44px recomendados para toque.
- **Orientações:** a linha inteira é arrastável e atrapalha a seleção de texto; o `aria-live` faz barulho demais.

## Código (API)

- `roundRating` é público em `ClinicSearchService` e reaproveitado em `ReviewService`: mover para um helper.
- `AvailabilityService`:
  - a sobrecarga de 3 argumentos perde o `@Transactional(readOnly)` (chamada interna);
  - `orElseThrow` sem argumento vira 500 se o contrato quebrar.
- `freeSlots` faz 2 consultas a mais mesmo sem clínica `MANUAL`. `AppointmentViews` calcula `deadline(...)` duas vezes.
- `requireDoctorFree` repete o filtro de sobreposição de `requireFreeSlot`.
- `Professional`:
  - `replaceSpecialties` não protege contra lista vazia;
  - as especialidades são carregadas EAGER (atenção a N+1 nas listas).
- `Collectors.toMap(User::getName)` supõe nomes não nulos.
- Seeder:
  - a V6 fixa `position = 1`;
  - o fallback do seeder dá NPE se `price` ou `slot` faltarem no JSON.
- Pedidos `PENDING` vencidos continuam aceitáveis até o job rodar (até 1 minuto, ou mais se o job parar).
- Swagger:
  - `ClinicController` não descreve horários por especialidade, profissionais por especialidade e `slotMinutes`;
  - `conflicts` ainda diz "Count".

## Código (web)

- Painel:
  - chaves de React duplicadas em períodos iguais no filtro "Todos";
  - linhas usam o índice como chave;
  - o snapshot `changed` nunca é limpo;
  - um erro antigo pisca ao voltar para uma semana que falhou.
- `DateField` chama `setText` durante o render e pode sobrescrever uma data digitada.
- Erros engolidos: nome da clínica em `ReviewList` e profissionais no painel.
- `AppointmentView` importa `Reviews.module.css`: extrair `ReviewThanks`.
- Mocks:
  - `CreateBlockCommand.category` é obrigatório no TS, mas a API usa `OTHER` como padrão;
  - o escopo "passados" do mock usa `>=` e a API usa `>`;
  - a nota média do mock não bate com a nota fixa da clínica.
- Cores fixas (`white`/`rgb`) em `color-mix` e sombras; `.segment` sem `:active`.

## Testes

- API:
  - faltam: 409 de avaliação duplicada concorrente, limites exatos de `scope=past`, vazamento entre clínicas, `respondBy` gravado na remarcação `MANUAL`, V6 rodando duas vezes;
  - o teste de concorrência não confere o código `SLOT_UNAVAILABLE`;
  - `manualClinicStoresRespondBy` falha perto de 06:00 e da meia-noite;
  - o teste A1 usa `sleep(1500)`;
  - `ResponseDeadlineTest` tem um nome errado e um comentário desatualizado.
- Web:
  - faltam caminhos de 409/400 nas avaliações e falhas ao aceitar e carregar no painel;
  - faltam `formatRespondBy` perto da meia-noite UTC e `weeklySummary`;
  - faltam períodos encostados, limite de 10 orientações, posição dos bloqueios na agenda e testes do mock;
  - `AppointmentList` e `AppointmentView` ainda usam a hora real.

## Documentação

- README da API: caminhos de `/api/clinic` abreviados numa linha.
- README do web: "ver Rotas abaixo" deveria ser "acima".
