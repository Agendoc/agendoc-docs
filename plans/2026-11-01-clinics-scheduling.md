# Clínicas e agendamento (parte 1): plano de implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** fechar a jornada pré-triagem → busca no mapa → agendamento → minhas consultas → painel da clínica → avaliação.

**Architecture:** são fatias verticais. Cada tarefa entrega API + web + mock MSW juntos, com os gates verdes. Na API, novos pacotes por feature (`clinic`, `appointment`, `panel`, `profile`, `review`, `demo`), seguindo as camadas atuais. Na web, a área do paciente vai para o route group `(patient)`, com a barra inferior, e o painel fica em `/clinica`.

**Tech Stack:** Spring Boot 4.1 / Java 21 / MySQL 8.4 / Flyway / Testcontainers · Next.js 16 / React 19 / Vitest / MSW · `leaflet` + `react-leaflet` (novas).

**Spec:** [`../specs/2026-11-01-clinics-scheduling-design.md`](../specs/2026-11-01-clinics-scheduling-design.md). Leia o spec antes. Este plano só registra as decisões de implementação.

## Global Constraints

- Branch `feat/clinics-scheduling` em `agendoc-api` e em `agendoc-web`. Commits semânticos, curtos, em inglês, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- API: `JAVA_HOME=~/.asdf/installs/java/temurin-21.0.11+10.0.LTS ./mvnw verify` verde a cada tarefa (um teste: `... ./mvnw test -Dtest=NomeIT`). Web: `npm run ci` verde a cada tarefa.
- Camadas: `controllers/<f>`, `command/<f>/inputs/*Command`, `command/<f>/outputs/*DTO`, `services/<f>`, `entities`, `repositories`. Swagger só nos controllers. Código em inglês e interface em pt-BR. Poucos comentários.
- Migrações Flyway só aditivas, a partir de `V4`. Nunca editar `V1`–`V3`.
- Erros: `ApiException(ErrorCode.X)`. Novos códigos: `SLOT_UNAVAILABLE` (409, "Esse horário acabou de ser ocupado. Escolha outro."), `CANCELLATION_WINDOW_CLOSED` (409, "Só é possível cancelar ou remarcar até 2 h antes. Fale com a clínica."), `APPOINTMENT_LIMIT` (409, "Você já tem 5 consultas ativas."), `INVALID_STATUS` (409, "Esta consulta não permite essa ação."), `RESCHEDULE_PENDING` (409, "Já existe uma remarcação aguardando a clínica."). Espelhar em `agendoc-web/src/lib/api/types.ts` (`ErrorCode`).
- Recurso de outro dono → 404 (`NOT_FOUND`), como as conversas.
- Fuso de exibição e cálculo: `America/Sao_Paulo`. Armazenar `Instant` (UTC). Sempre usar o `Clock` injetado (`MutableClock` nos testes).
- Toda tela de paciente que mostra clínica ou médico exibe "Dados de demonstração".
- Valores fixos do spec: cancelar e remarcar até **2 h** antes; janela de agendamento de **agora + 1 h** até **30 dias**; pendente expira em **24 h** sem resposta ou **2 h** antes do início; **5** consultas ativas por paciente; **10** ações de marcar, remarcar ou cancelar por hora; comentário da avaliação com até **500** caracteres; `resumo` da triagem com até **300** caracteres; **3** locais de emergência mais próximos; fallback do mapa a mais de **50 km** → centro de SP (`-23.5614, -46.6559`, Av. Paulista).
- Variável de ambiente nova → `application.yml` + `.env.example` + `deploy/compose.yaml`.

## Review Focus

- **Corrida pelo mesmo horário:** duas reservas simultâneas do mesmo médico e horário resultam em uma 201 e uma 409 `SLOT_UNAVAILABLE`, nunca duas consultas (teste na Tarefa 5).
- **Horário perto da meia-noite:** um horário às 22:00 em SP (01:00 UTC do dia seguinte) aparece no dia local certo, e o filtro "hoje" usa o dia de SP (teste na Tarefa 2).
- **Tela desatualizada:** reservar um horário que virou bloqueio, já passou ou saiu da janela responde 409 `SLOT_UNAVAILABLE`, não 500 (teste na Tarefa 5).
- **Remarcação recusada em clínica manual:** a consulta original continua `CONFIRMED` e libera nova remarcação (teste na Tarefa 9).
- **Conversa apagada depois de marcar:** a consulta continua com o resumo copiado e `triageConversationId` fica nulo, sem erro no detalhe nem no painel (teste na Tarefa 7).

---

### Task 1: Base: AGENTS.md, schema, entidades e seed de demonstração

**Files (API):**
- Create: `AGENTS.md` (aponta para `agendoc-docs`, as convenções e os comandos); o mesmo em `agendoc-web/AGENTS.md`, acrescentando o link (esse arquivo já existe)
- Create: `src/main/resources/db/migration/V4__create_clinics_scheduling.sql`
- Create: `entities/{Clinic,ClinicKind,ConfirmationMode,Professional,ScheduleRule,ScheduleBlock,Appointment,AppointmentStatus,PaymentType,CancelledBy,Review,Insurer,Specialties,StringListConverter}.java`
- Modify: `entities/Role.java` (+`CLINIC`), `entities/User.java` (+ campos de perfil e `clinicId`), `services/assistant/TriageTool.java` (usa `Specialties`)
- Create: `repositories/{ClinicRepository,ProfessionalRepository,ScheduleRuleRepository,ScheduleBlockRepository,AppointmentRepository,ReviewRepository}.java`
- Create: `services/demo/DemoDataSeeder.java`, `src/main/resources/demo/clinics.json`
- Test: `repositories/RepositoriesIT.java` (estender), `services/demo/DemoDataSeederIT.java`

**Schema (V4):**
- `clinics`: `id BINARY(16)`, `name`, `kind` (`CLINIC|EMERGENCY_CARE`), `address`, `latitude DECIMAL(9,6)`, `longitude DECIMAL(9,6)`, `phone`, `whatsapp NULL`, `photo_url NULL`, `confirmation_mode` (`AUTOMATIC|MANUAL`), `insurers JSON` (códigos de `Insurer`), `guidelines JSON` (lista de textos), `created_at`, `updated_at`
- `professionals`: `id`, `clinic_id` FK, `name`, `crm`, `crm_state CHAR(2)`, `specialty`, `private_price_cents INT`, `slot_minutes INT`, `active BOOL`, timestamps
- `schedule_rules`: `id`, `professional_id` FK cascade, `day_of_week TINYINT` (1 = segunda … 7 = domingo), `start_time TIME`, `end_time TIME`. As pausas são várias regras no mesmo dia (08–12 e 13–17)
- `schedule_blocks`: `id`, `clinic_id` FK, `professional_id NULL` FK, `starts_at`, `ends_at`, `reason NULL`
- `appointments`: `id`, `patient_id` FK users, `professional_id` FK, `clinic_id` FK, `starts_at`, `ends_at`, `status`, `payment_type` (`PRIVATE|INSURANCE`), `insurer NULL`, `insurance_card NULL`, `triage_message_id NULL` FK messages **ON DELETE SET NULL**, `triage_snapshot JSON NULL`, `email_reminder BOOL`, `replaces_id NULL` FK appointments, `cancel_reason NULL`, `cancelled_by NULL` (`PATIENT|CLINIC|SYSTEM`), timestamps. Mais a coluna gerada `active_start DATETIME(6) AS (CASE WHEN status IN ('PENDING','CONFIRMED') THEN starts_at END) STORED` e `UNIQUE (professional_id, active_start)`: é a trava de banco contra horário duplicado
- `reviews`: `id`, `appointment_id` UNIQUE FK, `clinic_id` FK, `rating TINYINT`, `comment VARCHAR(500) NULL`, `created_at`
- `users` + `clinic_id NULL` FK, `phone NULL`, `address NULL`, `latitude NULL`, `longitude NULL`, `insurer NULL`, `insurance_card NULL`

**Interfaces:**
- Produces: `Specialties.ALL: List<String>` e `Specialties.DEFAULT = "Clínico Geral"` (move a lista do `TriageTool`); `Insurer` enum com `displayName()` (UNIMED "Unimed", BRADESCO "Bradesco Saúde", SULAMERICA "SulAmérica", AMIL "Amil", HAPVIDA "Hapvida", NOTREDAME "NotreDame Intermédica", PORTO "Porto Saúde"); `AppointmentStatus { PENDING, CONFIRMED, DECLINED, EXPIRED, CANCELLED, COMPLETED, NO_SHOW }` com `isActive()` (PENDING ou CONFIRMED); os repositórios Spring Data das entidades.
- Seeder: `DemoDataSeeder implements ApplicationRunner`, ativo com `agendoc.demo.enabled` (env `DEMO_DATA_ENABLED`, padrão `true`, `false` no perfil `test`). Só insere quando `clinics` está vazia. Cria logins `CLINIC` (`<slug>@demo.agendoc.dev`) somente se `DEMO_CLINIC_PASSWORD` estiver definida. Conteúdo do `clinics.json`: 12 clínicas em Paulista, Pinheiros e Vila Mariana, cobrindo todas as `Specialties.ALL`, com 1–3 médicos cada (CRM `DEMO-0001`… para ficar claramente fictício), regras seg–sex 08–12 e 13–18 (algumas com sábado de manhã), 3 clínicas `MANUAL`, 4 `EMERGENCY_CARE` e 2–4 avaliações de demonstração por clínica (consultas `COMPLETED` no passado de um paciente `demo-reviews@demo.agendoc.dev` sem senha utilizável).

- [ ] Criar branch nos dois repositórios e escrever os `AGENTS.md`.
- [ ] Testes que falham: `RepositoriesIT.rejectsSecondActiveAppointmentOnSameSlot` (2º insert ativo no mesmo `professional_id/starts_at` → `DataIntegrityViolationException`; um `CANCELLED` mais um `CONFIRMED` no mesmo horário → ok); `RepositoriesIT.deletingMessageNullsAppointmentTriageLink`; `DemoDataSeederIT.seedsOnceAndIsIdempotent` (rodar 2× → 16 `clinics`); `DemoDataSeederIT.createsNoClinicLoginsWithoutPassword`; `TriageToolTest` continua verde.
- [ ] Implementar migração, entidades, repositórios e seeder. Rodar `./mvnw verify` → verde.
- [ ] Commit (API): `feat: add clinics and scheduling schema with demo data`. Commit (web): `docs: point agents to agendoc-docs`.

### Task 2: Cálculo de horários livres

**Files:** Create `services/clinic/SlotCalculator.java`, `services/clinic/Slot.java` · Test `services/clinic/SlotCalculatorTest.java`

**Interfaces:**
- Produces: `record Slot(UUID professionalId, Instant startsAt, Instant endsAt)`; `List<Slot> SlotCalculator.freeSlots(Professional p, List<ScheduleRule> rules, List<ScheduleBlock> blocks, List<Appointment> active, LocalDate fromLocal, int days)`. Usa `Clock`; zona `America/Sao_Paulo`; recorta em `[now + 1h, now + 30d]`; horários alinhados ao início da regra a cada `slotMinutes`, sem passar do `end_time`; remove os horários que cruzam bloqueio (da clínica ou do médico) e os que coincidem com consulta ativa.

- [ ] Testes que falham (unitários, `MutableClock`): `generatesSlotsFromRulesWithBreak` (08–12 e 13–14, 30 min → 08:00…11:30, 13:00, 13:30); `excludesBlockedAndBooked`; `respectsOneHourLeadAndThirtyDayHorizon`; `lateEveningSlotBelongsToLocalDay` (regra 21–23, o horário das 22:00 tem `atZone(SP).toLocalDate()` igual ao dia pedido); `inactiveProfessionalHasNoSlots`.
- [ ] Implementar. Rodar o teste → verde.
- [ ] Commit: `feat: compute free slots from schedule rules`.

### Task 3: API de busca, detalhe da clínica, horários e emergência

**Files:** Create `controllers/clinic/ClinicController.java`, `services/clinic/{ClinicSearchService,Distance}.java`, `command/clinic/inputs/ClinicSearchCommand.java`, `command/clinic/outputs/{ClinicSummaryDTO,ClinicDetailDTO,ProfessionalDTO,SlotDTO,EmergencyCareDTO}.java` · Test `controllers/clinic/ClinicSearchIT.java`, `services/clinic/DistanceTest.java`

**Interfaces (todas com sessão; papéis `USER` ou `ADMIN`):**
- Modify `SecurityConfig`: `/api/clinics/**`, `/api/appointments/**`, `/api/me/**`, `/api/emergency-care`, `/api/catalog` → `hasAnyRole("USER","ADMIN")` (teste `ClinicSearchIT.clinicRoleIs403`).
- `GET /api/clinics?specialty&lat&lng&insurer&date(yyyy-MM-dd)&period(MORNING|AFTERNOON|EVENING)&maxKm&minRating&sort(EARLIEST|NEAREST|RATING)` → `ClinicSummaryDTO[] {id, name, address, latitude, longitude, distanceKm, rating(null|1 casa), reviewCount, insurers[], minPrivatePriceCents, nextSlots: SlotDTO[≤3]}`. Só `kind=CLINIC` com médico ativo da especialidade e ao menos 1 horário livre que passe nos filtros. Sem `lat/lng`, `distanceKm = null` e `NEAREST` cai para `EARLIEST`. Períodos: manhã < 12:00 ≤ tarde < 18:00 ≤ noite. `specialty` fora de `Specialties.ALL` → 400.
- `GET /api/clinics/{id}` → `ClinicDetailDTO {…summary sem nextSlots, phone, whatsapp, guidelines[], confirmationMode, professionals: ProfessionalDTO[{id,name,crm,crmState,specialty,privatePriceCents}]}`
- `GET /api/clinics/{id}/slots?specialty&from(yyyy-MM-dd)&days(1..14, padrão 7)` → `SlotDTO[] {professionalId, startsAt, endsAt}`
- `GET /api/emergency-care?lat&lng` → `EmergencyCareDTO[3] {id,name,address,latitude,longitude,phone,distanceKm}`. Sem coordenadas, ordena pela distância até o centro de SP.
- `Distance.km(lat1,lng1,lat2,lng2)`: haversine com R = 6371.
- `GET /api/catalog` → `{specialties: string[], insurers: [{code, name}]}`, para a web não duplicar listas.

- [ ] Testes que falham: `DistanceTest.paulistaToPinheirosAbout3Km` (entre 2,5 e 3,5); `ClinicSearchIT.filtersBySpecialtyAndInsurer`, `sortsByNearest`, `omitsClinicsWithoutFreeSlotsInPeriod`, `excludesEmergencyCareFromSearch`, `emergencyCareReturnsThreeNearest`, `unknownSpecialtyIs400`, `requiresSession` (401). Os ITs inserem os próprios dados (seed desligado no perfil `test`).
- [ ] Implementar (filtrar em memória: são poucas clínicas). `./mvnw verify` → verde.
- [ ] Commit: `feat: add clinic search, slots and emergency care endpoints`.

### Task 4: Web: área do paciente, busca com mapa e tela da clínica

**Files (web):**
- Move: `src/app/chat/**` → `src/app/(patient)/chat/**`. O layout de `chat` sobe para `src/app/(patient)/layout.tsx` (sessão + `BottomNav`; papel `CLINIC` → `redirect("/clinica")`)
- Create: `src/components/ui/BottomNav.tsx` (Conversas `/chat` · Consultas `/consultas` · Perfil `/perfil`), `src/components/ui/DemoBadge.tsx`
- Create: `src/features/scheduling/{api.ts,types.ts,SearchView.tsx,ClinicMap.tsx,ClinicSheet.tsx,Filters.tsx,ClinicView.tsx,SlotPicker.tsx,useLocation.ts}` e testes `SearchView.test.tsx`, `SlotPicker.test.tsx`, `useLocation.test.ts`
- Create: `src/app/(patient)/consultas/buscar/page.tsx`, `src/app/(patient)/consultas/clinica/[id]/page.tsx`
- Create: `src/mocks/scheduling.ts` (handlers + dados), registrado em `src/mocks/handlers.ts`; estender `src/mocks/db.ts`
- Delete: `src/app/chat/agendar/page.tsx`

**Interfaces:**
- `scheduling/api.ts`: `searchClinics(q: ClinicQuery): Promise<ClinicSummary[]>`, `getClinic(id)`, `getSlots(id, specialty, from, days)`, `getEmergencyCare(coords?)`, `getCatalog()`. Os tipos espelham os DTOs da Tarefa 3.
- `useLocation(): { coords: Coords | null; source: "browser" | "profile" | "default"; status }`: navegador → endereço do perfil (`GET /api/me/profile`, Tarefa 8; até lá, 404 é tratado como sem perfil) → centro de SP. Também cai no centro de SP se estiver a mais de 50 km dele.
- URL da busca: `/consultas/buscar?especialidade=X&triagem=<messageId>&conversa=<id>`. Os filtros ficam na query string.
- `ClinicMap`: `react-leaflet` via `next/dynamic` com `ssr:false`, tiles do OSM com atribuição visível, pins com "★nota · próximo horário". A lista (`ClinicSheet`) funciona sozinha, por teclado e leitor de tela.
- Tocar num horário leva para `/consultas/confirmar?medico=<id>&inicio=<iso>&clinica=<id>&triagem=…` (a tela fica pronta na Tarefa 6).

- [ ] Testes que falham: `SearchView` mostra o selo "Indicado na pré-triagem: Ortopedista" quando recebe `triagem`; aplicar o chip de convênio refaz a busca com `insurer`; o estado vazio sugere afrouxar os filtros; aparece "Dados de demonstração". `SlotPicker` agrupa por médico e por manhã/tarde, e o horário selecionado vai para a URL de confirmação. `useLocation` cai para `default` quando a geolocalização é negada.
- [ ] Instalar `leaflet react-leaflet @types/leaflet`, implementar e rodar `npm run ci` → verde. Conferir manualmente com `npm run mock:api` + `npm run dev`.
- [ ] Commit: `feat: add clinic search with map and clinic slots screen`.

### Task 5: API de agendamento (marcar e solicitar) e `resumo` da triagem

**Files:** Create `controllers/appointment/AppointmentController.java`, `services/appointment/{AppointmentService,AppointmentRateLimiter}.java`, `command/appointment/inputs/BookAppointmentCommand.java`, `command/appointment/outputs/{AppointmentDTO,TriageSnapshotDTO}.java` · Modify `entities/Triage.java` (+`summary`), `TriageTool` (propriedade `resumo`, até 300 caracteres, mesmos guards), `MockChatAssistant` (preenche `resumo`; o conteúdo `#emergencia` gera triagem `EMERGENCY`), `TriageDTO`, `LimitsProperties` (+`appointmentActionsPerHour=10`, `maxActiveAppointments=5`), `ErrorCode` · Test `controllers/appointment/BookingIT.java`, `TriageToolTest`, `MockChatAssistantTest`

**Interfaces:**
- `POST /api/appointments` `BookAppointmentCommand {professionalId, startsAt, paymentType, insurer?, insuranceCard?(≤30), triageMessageId?, shareTriage, emailReminder}` → 201 `AppointmentDTO {id, status, startsAt, endsAt, clinic:{id,name,address,latitude,longitude,phone,whatsapp,guidelines[]}, professional:{id,name,crm,crmState,specialty}, paymentType, insurer, insuranceCard, triage: TriageSnapshotDTO?, triageConversationId?, recommendations[], cancelReason, cancelledBy, canCancel, reviewable, rescheduleOf?}`
- Regras: o horário precisa estar em `SlotCalculator.freeSlots(...)` (se não, `SLOT_UNAVAILABLE`); trava `PESSIMISTIC_WRITE` na linha do `Professional` durante a transação, e `DataIntegrityViolationException` da unique → `SLOT_UNAVAILABLE`; `AUTOMATIC` → `CONFIRMED`, `MANUAL` → `PENDING`; `INSURANCE` exige um `insurer` aceito pela clínica (se não, 400); mais de 5 ativas → `APPOINTMENT_LIMIT`; mais de 10 ações por hora → `RATE_LIMITED`; `triageMessageId` precisa ser de uma conversa do próprio paciente (se não, 404). `shareTriage=true` copia `{condition, specialty, urgency, summary}` para `triage_snapshot`. As `recommendations` do DTO vêm da mensagem de triagem, se ela ainda existir.

- [ ] Testes que falham: `BookingIT.automaticClinicConfirms`, `manualClinicCreatesPending`, `concurrentBookingsOnlyOneWins` (2 threads, `CountDownLatch` → status `{201, 409}`), `blockedOrPastSlotIs409`, `outsideWindowIs409`, `sixthActiveIs409Limit`, `insurerNotAcceptedIs400`, `foreignTriageMessageIs404`, `snapshotOnlyWhenShared`; `TriageToolTest.keepsSummaryTruncatedTo300`; `MockChatAssistantTest.emergencyTrigger`.
- [ ] Implementar. `./mvnw verify` → verde.
- [ ] Commit: `feat: book appointments with slot locking and triage summary`.

### Task 6: Web: confirmar, consulta marcada e cartão de emergência

**Files:** Create `src/features/appointments/{api.ts,types.ts,ConfirmView.tsx,ConfirmedView.tsx,Guidelines.tsx}` + testes, `src/app/(patient)/consultas/confirmar/page.tsx` · Modify `src/features/chat/TriageCard.tsx` (link para a busca com `triagem=<messageId>`; em `EMERGENCY`: sem "Agendar", com `tel:192` e os 3 locais de `getEmergencyCare`), `src/mocks/scheduling.ts`

**Interfaces:**
- `appointments/api.ts`: `bookAppointment(cmd): Promise<Appointment>` (mais as funções das Tarefas 8 e 9).
- `ConfirmView`: pagamento (convênio padrão do perfil pré-selecionado, quando houver), checkbox **desmarcado** "Compartilhar o resumo da pré-triagem com a clínica" (só aparece com `triagem`), lembrete por e-mail (ligado e salvo), WhatsApp desabilitado com "em breve". O botão é "Confirmar agendamento" ou "Solicitar horário", conforme `confirmationMode`. Erro `SLOT_UNAVAILABLE` → volta para a tela da clínica com aviso e grade recarregada.
- Depois do sucesso → `/consultas/<id>?nova=1`, que renderiza `ConfirmedView` em cima do detalhe ("Tudo certo!" ou "Aguardando a clínica"; adicionar ao calendário (Tarefa 8); como chegar `https://www.openstreetmap.org/directions?to=lat,lng`; orientações = `clinic.guidelines` + `recommendations`; aviso "Orientações gerais, não substituem a avaliação médica."; sinais de alerta fixos com o 192).

- [ ] Testes que falham: o checkbox de compartilhar começa desmarcado e manda `shareTriage:false`; clínica manual mostra "Solicitar horário"; `SLOT_UNAVAILABLE` mostra o aviso; `TriageCard` em emergência não tem o link "Agendar consulta" e tem `href="tel:192"`.
- [ ] Implementar. `npm run ci` → verde.
- [ ] Commit: `feat: add booking confirmation and emergency card`.

### Task 7: API: minhas consultas, cancelar e calendário

**Files:** Modify `AppointmentController`, `AppointmentService` · Create `services/appointment/IcsWriter.java` · Test `controllers/appointment/MyAppointmentsIT.java`, `services/appointment/IcsWriterTest.java`

**Interfaces:**
- `GET /api/appointments?scope=UPCOMING|PAST` → `AppointmentDTO[]`. UPCOMING: ativas com início no futuro, mais as canceladas pela clínica ou recusadas/expiradas nos últimos 7 dias (para o paciente ver o aviso), em ordem crescente. PAST: o resto, em ordem decrescente, limite de 100.
- `GET /api/appointments/{id}` → `AppointmentDTO` (dono, senão 404). `canCancel` = ativa e `now ≤ startsAt − 2h`. `reviewable` = `COMPLETED` e sem review.
- `POST /api/appointments/{id}/cancel` → 200 `AppointmentDTO` (`CANCELLED`, `cancelledBy=PATIENT`). Fora da janela → `CANCELLATION_WINDOW_CLOSED`; não ativa → `INVALID_STATUS`. Conta no limite de ações.
- `GET /api/appointments/{id}/calendar.ics` → `text/calendar`, `VEVENT` com `UID=<id>@agendoc`, `DTSTART/DTEND` em UTC, `SUMMARY="Consulta: <especialidade> (<clínica>)"`, `LOCATION` com o endereço e `STATUS` (CONFIRMED ou TENTATIVE).

- [ ] Testes que falham: `MyAppointmentsIT.listsOnlyOwnByScope`, `otherPatientsAppointmentIs404`, `cancelWithinWindow`, `cancelLessThanTwoHoursBeforeIs409`, `deletedConversationKeepsSnapshot` (apagar a conversa → GET 200, `triage` presente, `triageConversationId` nulo), `icsDownload`; `IcsWriterTest.escapesCommasAndUsesUtc`.
- [ ] Implementar. `./mvnw verify` → verde.
- [ ] Commit: `feat: list, cancel and export patient appointments`.

### Task 8: Web: minhas consultas, detalhe e perfil (API + web)

**Files (API):** Create `controllers/profile/ProfileController.java`, `services/profile/{ProfileService,Geocoder,NominatimGeocoder}.java`, `command/profile/inputs/UpdateProfileCommand.java`, `command/profile/outputs/ProfileDTO.java` · Test `controllers/profile/ProfileIT.java` (com `Geocoder` falso via `@TestConfiguration`)
**Files (web):** Create `src/features/appointments/{AppointmentList.tsx,AppointmentDetail.tsx,CancelAppointment.tsx}` + testes, `src/features/profile/{api.ts,ProfileForm.tsx}` + teste, `src/app/(patient)/consultas/page.tsx`, `src/app/(patient)/consultas/[id]/page.tsx`, `src/app/(patient)/perfil/page.tsx`

**Interfaces:**
- `GET /api/me/profile` → `ProfileDTO {name, phone, address, hasLocation, latitude, longitude, insurer, insuranceCard}`. `PUT /api/me/profile` `UpdateProfileCommand {phone?(≤20), address?(≤200), insurer?, insuranceCard?}`: quando o endereço muda, chama `Geocoder.locate(address): Optional<LatLng>`; se falhar, salva sem coordenadas (`hasLocation=false`).
- `NominatimGeocoder`: `https://nominatim.openstreetmap.org/search?format=json&limit=1&countrycodes=br&q=…`, header `User-Agent` vindo de `GEOCODER_USER_AGENT` (padrão `AgenDoc/1.0 (contato no README)`), timeout de 5 s, no máximo 1 requisição por segundo (trava simples). Com `agendoc.geocoder.enabled=false` (perfil `test`), a implementação é no-op.
- Web: `/consultas` com abas Próximas/Anteriores, botão "Agendar consulta" (abre a busca com um seletor de especialidade vindo de `getCatalog`) e selos de status: Confirmada · Aguardando clínica · Cancelada pela clínica (motivo + "Escolher novo horário") · Recusada · Expirou · Cancelada · Realizada · Faltou. O detalhe mostra contato, rota, `.ics` (`<a href="/api/appointments/{id}/calendar.ics" download>`), orientações, a triagem de origem (com link para a conversa, se `triageConversationId`), remarcar (Tarefa 9) e cancelar com confirmação dentro da interface. Fora da janela, o botão de cancelar fica desabilitado, com o texto "Cancelamento e remarcação até 2 h antes."

- [ ] Testes que falham: `ProfileIT.updatesAndGeocodes`, `geocoderFailureSavesWithoutLocation`; web: a lista mostra o motivo do cancelamento da clínica; cancelar pede confirmação e atualiza o selo; perfil salva e mostra "Não encontramos esse endereço no mapa" quando `hasLocation=false`.
- [ ] Implementar (API + web + mock). Os dois gates verdes.
- [ ] Commits: API `feat: add patient profile with geocoding`; web `feat: add my appointments and profile screens`.

### Task 9: Remarcar (API + web)

**Files:** Modify `AppointmentController`, `AppointmentService` · Create `command/appointment/inputs/RescheduleCommand.java` · Test `controllers/appointment/RescheduleIT.java` · Web: `src/features/appointments/RescheduleView.tsx` + teste, `src/app/(patient)/consultas/[id]/remarcar/page.tsx` (reusa `SlotPicker` filtrado pela clínica e especialidade)

**Interfaces:**
- `POST /api/appointments/{id}/reschedule` `RescheduleCommand {professionalId, startsAt}` → 201 `AppointmentDTO` (a nova consulta). O médico precisa ser da mesma clínica e da mesma especialidade. Valem a janela de 2 h e o mesmo caminho de reserva da Tarefa 5. `AUTOMATIC`: cancela a antiga (`cancelReason="Remarcada"`, `cancelledBy=PATIENT`) e cria `CONFIRMED` na mesma transação. `MANUAL`: cria `PENDING` com `replaces_id`, e a antiga continua `CONFIRMED`. Já existe um `PENDING` com `replaces_id` = esta → `RESCHEDULE_PENDING`. O par conta como **1** no limite de ativas.
- O aceite, a recusa e a expiração do par são feitos no painel e no job (Tarefa 10). Aqui só se cria.
- `AppointmentDTO.rescheduleOf` = id da antiga (na nova). Na antiga, o detalhe mostra "Remarcação para <data> aguardando a clínica".

- [ ] Testes que falham: `RescheduleIT.automaticSwapsAtomically`, `newSlotTakenKeepsOriginal` (409 e a antiga continua `CONFIRMED`), `manualKeepsOriginalUntilAccepted`, `secondPendingRescheduleIs409`, `pairCountsAsOneActive`, `differentSpecialtyIs400`; web: remarcar envia a nova reserva e mostra o estado "aguardando a clínica" na consulta original.
- [ ] Implementar. Os dois gates verdes.
- [ ] Commits: `feat: reschedule appointments` (API e web).

### Task 10: API do painel da clínica e criação de clínicas pelo admin

**Files:** Create `controllers/panel/{PanelAgendaController,PanelSetupController}.java`, `controllers/admin/AdminClinicController.java`, `services/panel/{PanelAgendaService,PanelSetupService,PendingExpiryJob}.java`, `services/admin/AdminClinicService.java`, `command/panel/inputs/*`, `command/panel/outputs/*`, `command/admin/*` · Modify `SecurityConfig` (`/api/clinic/**` → `hasRole("CLINIC")`) · Test `controllers/panel/PanelAgendaIT.java`, `controllers/panel/PanelSetupIT.java`, `controllers/admin/AdminClinicIT.java`, `services/panel/PendingExpiryJobTest.java`

**Interfaces (o escopo é sempre o `clinicId` do usuário logado):**
- Agenda: `GET /api/clinic/agenda?weekStart(yyyy-MM-dd)&professionalId?` → `{slots livres, appointments: PanelAppointmentDTO[], blocks[]}`. `PanelAppointmentDTO {id, status, startsAt, endsAt, professional, patient:{name, phone}, paymentType, insurer, insuranceCard, triage?, rescheduleOf?}`. A clínica nunca vê `triageMessageId` nem a conversa.
- Ações: `POST /api/clinic/appointments/{id}/accept`. Se a consulta tiver `replaces_id`, a antiga vira `CANCELLED` ("Remarcada") na mesma transação. As demais: `…/decline` → `DECLINED`; `…/complete` e `…/no-show` (só depois de `startsAt`; senão `INVALID_STATUS`); `…/cancel {reason (obrigatório, ≤200)}` → `CANCELLED`, `cancelledBy=CLINIC`. `GET /api/clinic/requests` → pendentes, com `expiresAt`.
- `PendingExpiryJob` (`@Scheduled(fixedDelay="PT1M")`) → `PENDING` com `createdAt + 24h ≤ now` **ou** `startsAt − 2h ≤ now` vira `EXPIRED` (`cancelledBy=SYSTEM`). `expiresAt` é o menor dos dois.
- Configuração: CRUD `/api/clinic/professionals` (desativar = `active=false`); `PUT /api/clinic/professionals/{id}/rules` (substitui a lista; valida início < fim e ausência de sobreposição no mesmo dia); `GET|POST|DELETE /api/clinic/blocks`. `POST` responde `{block, conflictingAppointments: n}` e aceita `cancelConflicts: true` + `reason`, que cancela as conflitantes na mesma transação. `GET|PUT /api/clinic/settings` (contatos, convênios, orientações ≤ 10 × 200 caracteres, modo de confirmação; sem endereço nem coordenadas); `GET /api/clinic/reviews`.
- Admin: `POST /api/admin/clinics {clinic…, latitude, longitude, loginEmail}` → 201 `{clinicId, loginEmail, initialPassword}` (16 caracteres aleatórios com `SecureRandom`, mostrada uma única vez, guardada com BCrypt).

- [ ] Testes que falham: `PanelAgendaIT.seesOnlyOwnClinic` (outra clínica → 404), `patientCannotCallPanel` (403), `acceptReschedulePairCancelsOriginal`, `declineReschedulePairKeepsOriginal`, `completeBeforeStartIs409`, `cancelRequiresReason`, `panelNeverExposesConversation`; `PendingExpiryJobTest.expiresAfter24hOrTwoHoursBefore`; `PanelSetupIT.blockReportsAndCancelsConflicts`, `overlappingRulesIs400`, `deactivatedProfessionalLeavesSearch`; `AdminClinicIT.createsClinicWithOneTimePassword`, `nonAdminIs403`.
- [ ] Implementar. `./mvnw verify` → verde.
- [ ] Commit: `feat: add clinic panel and admin clinic endpoints`.

### Task 11: Web: painel da clínica

**Files:** Create `src/app/clinica/layout.tsx` (exige papel `CLINIC`, senão `redirect("/chat")`; menu lateral), `src/app/clinica/{page.tsx,solicitacoes/page.tsx,medicos/page.tsx,horarios/page.tsx,bloqueios/page.tsx,dados/page.tsx,avaliacoes/page.tsx}`, `src/features/clinic-panel/{api.ts,types.ts,WeekAgenda.tsx,AppointmentPanel.tsx,Requests.tsx,Professionals.tsx,ScheduleRules.tsx,Blocks.tsx,Settings.tsx,Reviews.tsx}` + testes, `src/mocks/panel.ts` · Modify `src/features/auth/useAuthForm.ts` (depois do login, `CLINIC` → `/clinica`, os demais → `/chat`), `src/lib/api/types.ts` (`Role` + `"CLINIC"`)

- O layout é mais largo (sem a coluna de 480px), com tokens do design system. Fica usável no celular: o menu vira gaveta.
- `WeekAgenda`: grade semanal por médico (seletor) com legenda confirmada/aguardando/livre/bloqueado. Ao clicar numa consulta, abre `AppointmentPanel` com o resumo da triagem (o aviso "Gerado por IA a partir do relato do paciente. Não é diagnóstico." e as ações).
- `Blocks`: ao criar um bloqueio com conflitos, mostra "N consultas neste período" e oferece "Cancelar todas" com motivo.

- [ ] Testes que falham: login de clínica redireciona para `/clinica`; paciente em `/clinica` → `/chat`; aceitar uma solicitação some com ela da lista; cancelar exige motivo; o resumo mostra o aviso de IA; bloqueio com conflito mostra a contagem.
- [ ] Implementar. `npm run ci` → verde.
- [ ] Commit: `feat: add clinic panel`.

### Task 12: Avaliações (API + web)

**Files (API):** Create `controllers/review/ReviewController.java`, `services/review/ReviewService.java`, `command/review/*` · Modify `ClinicSearchService` (média e contagem) · Test `controllers/review/ReviewIT.java`
**Files (web):** Create `src/features/reviews/{api.ts,StarRating.tsx,ReviewForm.tsx,ReviewList.tsx}` + testes · Modify `AppointmentList`/`AppointmentDetail` (estrelas quando `reviewable`), `ClinicView` (link "Avaliações ›" → lista)

**Interfaces:**
- `POST /api/appointments/{id}/review {rating 1..5, comment?(≤500, TextSanitizer)}` → 201. Só o dono, só `COMPLETED`, só uma vez (senão 409 `INVALID_STATUS`).
- `GET /api/clinics/{id}/reviews` → `{rating, reviewCount, items:[{rating, comment, label: "Paciente verificado · <mês>/<ano>"}]}`, os 20 mais recentes, sem nenhum dado do autor nem do médico.
- Placeholder do comentário: "Conte como foi o atendimento. Não inclua dados de saúde nem nomes."

- [ ] Testes que falham: `ReviewIT.onlyCompletedOwnOnce`, `publicListIsAnonymous` (o JSON não contém `patient`, `name`, `email` nem `professional`), `searchRatingReflectsReviews`; web: as estrelas aparecem só em consulta realizada sem avaliação; enviar troca as estrelas por "Obrigado pela avaliação".
- [ ] Implementar. Os dois gates verdes.
- [ ] Commits: `feat: add appointment reviews` (API e web).

### Task 13: Fechamento: docs, deploy e roadmap

- [ ] `agendoc-api/README.md`: tabela da API, limites, papel `CLINIC`, criação de clínica pelo admin, seed e variáveis (`DEMO_DATA_ENABLED`, `DEMO_CLINIC_PASSWORD`, `GEOCODER_USER_AGENT`); remover de "Limitações" o que deixou de valer. `agendoc-web/README.md`: rotas novas e o mapa (atribuição do OSM).
- [ ] `.env.example` e `deploy/compose.yaml` com as variáveis novas. Avisar o autor para preencher `DEMO_CLINIC_PASSWORD` no `.env` da VM **antes** do merge.
- [ ] Caddy: confirmar que os tiles do OSM carregam (se houver CSP, liberar `tile.openstreetmap.org` em `img-src`).
- [ ] Rodar a jornada completa localmente (API real + web): chat → triagem (mock) → busca → marcar (automático e manual) → painel aceita → realizada → avaliação → nota na busca → `.ics` → emergência com `#emergencia`.
- [ ] `agendoc-docs/roadmap.md`: parte 1 → `feito`. Abrir PRs nos dois repositórios.
- [ ] Commits: `docs: document clinics and scheduling` (API e web), `docs: mark part 1 as done` (docs).
