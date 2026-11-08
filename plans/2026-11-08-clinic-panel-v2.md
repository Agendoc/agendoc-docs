# Clinic Panel v2 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Doctors with several specialties (own price and duration), the new deadline rule for pending requests, and the redesigned clinic panel, end to end (API + web).

**Architecture:**
- **API (`agendoc-api`):**
  - One additive Flyway migration (V5) for the schema and one data migration (V6) for the production demo.
  - The specialty becomes part of slot calculation, booking, search and every DTO.
  - A small `ResponseDeadline` component owns the "working hours" math: the deadline and the 4-hour minimum lead.
- **Web (`agendoc-web`):**
  - Types and the MSW mock are updated first, so every later screen task builds on the same contracts.
  - Then the patient changes, the shared panel pieces and one task per panel screen.

**Tech Stack:**
- API: Spring Boot 4.1, Java 21, MySQL 8.4 (Testcontainers in ITs), Flyway, Spotless, SpotBugs.
- Web: Next.js 16, React 19, CSS modules with design tokens, Vitest, MSW, Testing Library, lucide-react.

**Spec:** `agendoc-docs/specs/2026-11-08-clinic-panel-v2-design.md`.
- It complements `agendoc-docs/specs/2026-11-01-clinics-scheduling-design.md`.
- Mockups: `agendoc-docs/specs/assets/panel-agenda.html`, `panel-hours.html`, `panel-blocks.html` and `panel-doctors-settings.html`. They are the visual reference; read the yellow note at the top of each.

## Global Constraints

### Branches
- Both repos start from `feat/clinics-scheduling`. One branch per task: `feat/panel-v2-NN-<slug>` (NN = task number).
- When a task's gates are green, merge it locally with `git merge --no-ff` into `feat/clinics-scheduling`.
- Never push `main` or `release`.
- Do not merge into `develop` until the user approves the milestone.

### Gates
- API: `./mvnw verify`, which runs tests, Spotless (palantir) and SpotBugs with 0 bugs.
  - In this environment a hook blocks `./mvnw` in Bash. Run Maven through `ctx_execute` with the scratchpad script `mvn.sh` (its arguments are the Maven goals).
- Java: always set `JAVA_HOME=$HOME/.asdf/installs/java/temurin-21.0.11+10.0.LTS`.
- Web: `npm run ci` (lint, typecheck, tests and build).
- Never run `npx prettier` on the web repo.

### Commits and code
- Commits are semantic, short and in English, with the trailer `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- Code is in English; UI copy is in pt-BR.
- Few comments. Swagger annotations only on controllers.

### Database
- Flyway migrations are additive only: never drop or rename a column, never change existing data meaning.

### Processes
- To stop a server, kill by the PID from `ss -ltnp`. Never use `pkill -f`.

### Time
- All calendar math uses `America/Sao_Paulo` (`SlotCalculator.ZONE`), and storage is in UTC.

### Errors
- Use the existing `ErrorCode` and `ProblemDetail` pattern. New codes go in `ErrorCode`, `src/lib/api/types.ts`, `src/lib/api/errors.ts` and `src/mocks/support.ts`.

### Values pinned by the spec
- Specialties per doctor:
  - 1 to 5 per doctor, with no duplicates;
  - the specialty name comes from `Specialties.ALL`;
  - duration 10 to 120 min, in steps of 5;
  - price 0 to 1 000 000 cents.
- Periods: at most 4 per day; a period never crosses midnight; the time pickers step by 15 min, 24 h.
- Deadline (`respondBy`):
  - It is the end of the first working day of the doctor whose first period starts after the request, capped at `startsAt − 2h`.
  - Days fully covered by blocks are skipped.
  - If the doctor has no working day in the next 30 days, `respondBy = startsAt − 2h`.
  - The value is stored when the request is created.
- Expiry: `PENDING` becomes `EXPIRED` only when `now ≥ startsAt − 2h`. "Late" means `now > respondBy`; it is not a status.
- Lead time:
  - MANUAL clinics only offer slots starting at least 4 working hours of that doctor after now, skipping fully blocked days.
  - AUTOMATIC clinics keep `now + 1h`.
- Block categories: `VACATION`, `DAY_OFF`, `CONGRESS`, `HOLIDAY`, `MEETING`, `OTHER` (the default).
- "Past" blocks are those that ended within the last 90 days.

### UI rules
- Doctor color comes from a palette, by creation order (`createdAt`, then `id`). It is never stored. Inactive doctors are gray.
- The default doctor filter on the Agenda is "Todos os médicos".
- Agenda cards always start with the patient name. The status icon goes before the name: `?` pending, `!` late, `↻` reschedule. The time is on a second line only when the card is 30 min or longer. Text is never clipped: ellipsis, plus the full text in `title` and `aria-label`.
- Insurer badges use our palette, never the insurers' brand colors.

## Review Focus

1. **Different durations racing for one doctor.** Booking 08:00/30 min (Cardiologia) and 08:20/20 min (Clínico Geral) at the same moment for the same doctor must leave exactly one active appointment. The other gets 409 `SLOT_TAKEN`. Test: Task 2 `concurrentOverlappingSpecialtiesKeepOne`.
2. **Request near midnight and around a holiday.**
   - Request on Thursday 23:59, Friday is a clinic-wide holiday, the doctor works Monday to Friday: `respondBy` = end of Monday.
   - Request at 23:59 on a working day: the next day counts as "starts after the request".
   - Test: Task 3 `skipsFullyBlockedDaysAndHandlesLateNight`.
3. **Late reschedule request cancelled by the patient.** Only the request is cancelled. The original stays `CONFIRMED` and can be rescheduled again. Test: Task 3 `cancellingLateRescheduleKeepsOriginal`.
4. **A specialty removed from a doctor who has future appointments in it.**
   - The edit is allowed. The appointments keep their specialty and still render.
   - A reschedule to the same doctor in that specialty fails with a clear 409 (`SPECIALTY_UNAVAILABLE`).
   - Test: Task 4 `removingSpecialtyKeepsExistingAppointments`.
5. **Unsaved weekly hours when switching doctor.** Switching asks for confirmation inside the UI. Cancelling keeps the edits; confirming discards them. Test: Task 12 `asksBeforeDiscardingUnsavedHours`.

---

## API tasks (`agendoc-api`)

### Task 1: Schema for specialties, deadline and block category

**Branch:** `feat/panel-v2-01-schema`

**Files:**
- Create: `src/main/resources/db/migration/V5__panel_v2.sql`
- Create: `entities/ProfessionalSpecialty.java` (embeddable or entity, builder's choice)
- Create: `entities/BlockCategory.java`
- Modify: `entities/Professional.java`, `entities/Appointment.java`, `entities/ScheduleBlock.java`
- Test: `src/test/java/.../migration/PanelV2MigrationIT.java`

**Interfaces:**
- **Produces:**
  - Table `professional_specialties (professional_id, specialty, private_price_cents, slot_minutes)`, with primary key `(professional_id, specialty)` and an FK with cascade.
  - Column `appointments.specialty VARCHAR(60)`: nullable in the DDL, backfilled from `professionals.specialty`, always set by the code.
  - Column `appointments.respond_by DATETIME(6) NULL`.
  - Column `schedule_blocks.category VARCHAR(20) NOT NULL DEFAULT 'OTHER'`.
  - Backfill: one `professional_specialties` row per existing professional, copied from `specialty`, `private_price_cents` and `slot_minutes`.
  - `Professional.getSpecialties(): List<ProfessionalSpecialty>` (ordered by insertion; the first is "primary"), and `ProfessionalSpecialty(String specialty, int privatePriceCents, int slotMinutes)`.
  - `Professional.replaceSpecialties(List<ProfessionalSpecialty>)`, which also copies the first one into the legacy columns.
  - `Professional.specialty(String name): Optional<ProfessionalSpecialty>`.
  - `Appointment.getSpecialty()/setSpecialty`, `getRespondBy()/setRespondBy(Instant)`.
  - `ScheduleBlock.getCategory()/setCategory(BlockCategory)`.

- [ ] Write `PanelV2MigrationIT`. It uses Flyway's `target("4")`, inserts one clinic, professional, appointment and block with JDBC, then migrates to the latest version and asserts:
  - the professional has one specialty row with the same price and duration;
  - the appointment's `specialty` equals the professional's specialty;
  - the block's `category = 'OTHER'`;
  - `respond_by IS NULL`.

  Use a dedicated schema or clean it before and after, so other ITs are not affected.
- [ ] Run it: it fails because V5 is missing.
- [ ] Write V5, map the new columns and collection, and make every place that creates a `Professional` (seed, panel create, `SchedulingTestSupport`) also create its specialty row.
- [ ] Run `./mvnw verify`: everything passes.
- [ ] Commit: `feat: add schema for multi-specialty doctors, deadlines and block categories`.

### Task 2: Slots, search and booking by specialty

**Branch:** `feat/panel-v2-02-specialty-booking`

**Files:**
- Modify:
  - `services/clinic/SlotCalculator.java`, `services/clinic/AvailabilityService.java`, `services/clinic/ClinicSearchService.java`
  - `command/clinic/outputs/ProfessionalDTO.java`, `ClinicSummaryDTO.java`, `SlotDTO.java`
  - `command/appointment/inputs/BookAppointmentCommand.java`, `RescheduleCommand.java`
  - `services/appointment/AppointmentService.java`, `services/appointment/AppointmentViews.java`
  - `command/appointment/outputs/AppointmentDTO.java`
  - `exceptions/ErrorCode.java`, plus the matching controllers' Swagger.
- Test: `BookingIT`, `RescheduleIT`, `ClinicSearchIT` (or the existing search IT), `SlotCalculatorTest`.

**Interfaces:**
- **Consumes:** Task 1 (`Professional.specialties`, `Appointment.specialty`).
- **Produces:**
  - `SlotCalculator.freeSlots(Professional, ProfessionalSpecialty, rules, blocks, appointments, LocalDate from, int days)`:
    - the grid starts at each rule start and steps by the specialty's `slotMinutes`;
    - it excludes any overlap with the doctor's active appointments (all specialties) and with blocks.
  - `Slot` gains `specialty`, and `SlotDTO(professionalId, specialty, startsAt, endsAt)`.
  - Search and slots:
    - `GET /api/clinics?specialty=X` matches doctors that have X in `professional_specialties`;
    - `minPrivatePriceCents` is the minimum over those doctors for X;
    - `ProfessionalDTO(id, name, crm, crmState, specialty, privatePriceCents, slotMinutes)` carries the values of the searched specialty;
    - `GET /api/clinics/{id}/slots?specialty=X` returns only X's grid.
  - Booking:
    - `BookAppointmentCommand.specialty` (`String`, nullable): required when the doctor has more than one specialty, inferred when there is one.
    - An unknown specialty, or one the doctor lacks, gives 409 `SPECIALTY_UNAVAILABLE` with the message "Esse médico não atende essa especialidade." (new `ErrorCode`). With several specialties and none sent, the response is 400 VALIDATION.
  - Overlap guard: inside the existing professional row lock, `book` and `reschedule` reject any overlap with the doctor's active appointments (`[start, end)` intersects). It returns the existing 409 `SLOT_TAKEN`.
  - Reschedule keeps the original's specialty. The target doctor must have it (`SPECIALTY_UNAVAILABLE` otherwise).
  - `AppointmentDTO.ProfessionalInfo` shows the **appointment's** specialty, and `AppointmentDTO` gains `privatePriceCents` for that specialty.

- [ ] Tests first:
  - `SlotCalculatorTest.gridFollowsSpecialtyDuration`: 08:00–09:00 at 20 min gives 08:00, 08:20 and 08:40; at 30 min it gives 08:00 and 08:30.
  - `SlotCalculatorTest.otherSpecialtyAppointmentHidesOverlappingSlots`: an 08:00–08:30 appointment hides the 20-min slots at 08:00 and 08:20.
  - `BookingIT.bookingInferSingleSpecialty`.
  - `BookingIT.multiSpecialtyRequiresSpecialty` (400).
  - `BookingIT.unknownSpecialtyIs409` (`SPECIALTY_UNAVAILABLE`).
  - `BookingIT.endsAtUsesSpecialtyDuration`.
  - `BookingIT.concurrentOverlappingSpecialtiesKeepOne`: two threads, exactly one 201 and one 409 `SLOT_TAKEN`.
  - `RescheduleIT.keepsSpecialtyAndRejectsDoctorWithoutIt`.
  - Search IT `searchMatchesSecondSpecialtyWithItsPrice`.
- [ ] Run: the new tests fail.
- [ ] Implement.
- [ ] `./mvnw verify` passes.
- [ ] Commit: `feat: book and search by doctor specialty`.

### Task 3: Request deadline, late state and MANUAL lead time

**Branch:** `feat/panel-v2-03-deadline`

**Files:**
- Create: `services/clinic/ResponseDeadline.java`.
- Modify:
  - `services/appointment/AppointmentService.java` (set `respondBy` on MANUAL book and reschedule);
  - `services/panel/PendingExpiryJob.java` and `repositories/AppointmentRepository.java` (`expirePending`);
  - `services/clinic/AvailabilityService.java` (MANUAL lead);
  - `services/panel/PanelViews.java` and `PanelAgendaService.java`;
  - `command/panel/outputs/PanelAppointmentDTO.java` and `command/appointment/outputs/AppointmentDTO.java`.
- Test: `ResponseDeadlineTest`, `PendingExpiryJobIT`, `PanelAgendaIT`, `BookingIT`, `RescheduleIT`.

**Interfaces:**
- **Consumes:** Task 1 (`Appointment.respondBy`), Task 2.
- **Produces:**
  - `ResponseDeadline.respondBy(List<ScheduleRule> rules, List<ScheduleBlock> blocks, Instant requestedAt, Instant startsAt): Instant`, as defined in Global Constraints.
  - `ResponseDeadline.earliestStart(List<ScheduleRule> rules, List<ScheduleBlock> blocks, Instant now): Instant`. It returns the instant at which 4 working hours have passed after `now`, or `null` when the doctor has no working time in the next 30 days. With `null`, no MANUAL slots are offered.
  - `ResponseDeadline.WORKING_LEAD = Duration.ofHours(4)`.
  - A "fully blocked day" is one where every rule period of that day is covered by blocks of that doctor or of the whole clinic.
  - `respondBy` for old rows (null) is computed as `createdAt + 24h`, capped at `startsAt − 2h`.
  - The job: `expirePending(now)` expires `PENDING` where `startsAt ≤ now + 2h`. It no longer uses `createdAt`.
  - DTOs:
    - `PanelAppointmentDTO` replaces `expiresAt` with `respondBy` and gains `boolean late`;
    - `AppointmentDTO` gains `respondBy` and `late` (both only meaningful for `PENDING`);
    - `PendingReschedule` gains `respondBy` and `late`.
  - `GET /api/clinic/requests` orders late requests first (oldest `respondBy` first), then the rest by `respondBy` ascending.
  - Accept and decline work while `PENDING`, whether late or not; they are refused only after `EXPIRED`.

- [ ] Tests first:
  - `ResponseDeadlineTest` (plain unit tests with a fixed clock, doctor working Monday to Friday 08–12 and 14–18):
    - `fridayNightGivesEndOfMonday`;
    - `duringBusinessHoursGivesEndOfNextWorkingDay`;
    - `beforeOpeningGivesEndOfSameDay`;
    - `cappedTwoHoursBeforeStart`;
    - `skipsFullyBlockedDaysAndHandlesLateNight`;
    - `noWorkingDayIn30DaysFallsBackToStartMinusTwoHours`;
    - `earliestStartFridayNightIsMondayNoon`;
    - `earliestStartMondayNineIsMondayOnePm`;
    - `earliestStartSkipsLunchGap`: from Monday 11:00, 1 h before lunch plus 3 h after gives Monday 17:00.
  - `PendingExpiryJobIT`:
    - `lateRequestStaysPending`;
    - `expiresTwoHoursBeforeStart`.
  - `PanelAgendaIT`:
    - `acceptsLateRequest`;
    - `requestsListLateFirst`.
  - `BookingIT`:
    - `manualClinicStoresRespondBy`;
    - `manualClinicHidesSlotsInsideWorkingLead` (slots API);
    - `automaticClinicKeepsOneHourLead`.
  - `RescheduleIT.cancellingLateRescheduleKeepsOriginal`: the patient cancels the pending request through the existing cancel endpoint, the original stays `CONFIRMED`, and a new reschedule of the original is allowed.
- [ ] Run: the new tests fail.
- [ ] Implement.
- [ ] `./mvnw verify` passes.
- [ ] Commit: `feat: give clinics a working-day deadline for pending requests`.

### Task 4: Panel API for specialties, blocks and agenda periods

**Branch:** `feat/panel-v2-04-panel-api`

**Files:**
- Modify:
  - `command/panel/inputs/ProfessionalCommand.java` and `CreateBlockCommand.java`;
  - `command/panel/outputs/PanelProfessionalDTO.java`, `BlockDTO.java`, `ConflictsDTO.java` and `AgendaDTO.java`;
  - `services/panel/PanelSetupService.java` and `PanelAgendaService.java`;
  - `controllers/panel/*` (Swagger).
- Test: `PanelSetupIT` and `PanelAgendaIT`.

**Interfaces:**
- **Consumes:** Tasks 1 to 3.
- **Produces (JSON contracts the web uses):**
  - `ProfessionalCommand { name, crm, crmState, active, specialties: [{ specialty, privatePriceCents, slotMinutes }] }`. The old single-specialty fields are removed from the command (the web is the only client). Validation follows the pinned values.
  - `PanelProfessionalDTO { id, name, crm, crmState, active, createdAt, specialties: [{ specialty, privatePriceCents, slotMinutes }], rules: [...] }`. The list is ordered by `createdAt`, then `id`, so the web palette index is stable.
  - `PUT /api/clinic/professionals/{id}/rules`: at most 4 rules per day (400 otherwise). Everything else stays the same.
  - `CreateBlockCommand` gains `category` (defaults to `OTHER` when null). `BlockDTO` gains `category`.
  - `GET /api/clinic/blocks?scope=upcoming|past`:
    - `upcoming` is `endsAt > now`, ascending;
    - `past` is `now − 90d < endsAt ≤ now`, descending;
    - the default is `upcoming`.
  - `GET /api/clinic/blocks/conflicts` returns `{ count, appointments: [{ id, patientName, startsAt, endsAt, status, professionalId }] }`.
  - `AgendaDTO` gains `periods: [{ professionalId, startsAt, endsAt }]`: the weekly rules materialized as instants for the requested week and doctors. It is what the web paints as "open". Keep `freeSlots` for now; the web stops using it in Task 9.
  - `PanelAppointmentDTO` gains `specialty` and `privatePriceCents` (from the appointment).

- [ ] Tests first:
  - `PanelSetupIT`:
    - `createsDoctorWithTwoSpecialties`;
    - `rejectsDuplicateOrTooManySpecialties`;
    - `rejectsFifthRuleOnSameDay`;
    - `removingSpecialtyKeepsExistingAppointments`;
    - `blocksByScope`;
    - `blockCategoryRoundTrip`;
    - `conflictsListAffectedAppointments`.
  - `PanelAgendaIT`:
    - `agendaReturnsWeeklyPeriods`;
    - `panelAppointmentShowsSpecialtyAndPrice`.
- [ ] Run: the new tests fail.
- [ ] Implement.
- [ ] `./mvnw verify` passes.
- [ ] Commit: `feat: manage doctor specialties, block categories and agenda periods in the panel`.

### Task 5: Demo data

**Branch:** `feat/panel-v2-05-demo`

**Files:**
- Create: `src/main/resources/db/migration/V6__demo_multi_specialty.sql`.
- Modify: `src/main/resources/demo/clinics.json` and `services/demo/DemoDataSeeder.java`.
- Test: `DemoDataSeederIT` (existing, or create it).

**Interfaces:**
- **Consumes:** Task 1.
- **Produces:**
  - The seed JSON supports `"specialties": [{ specialty, privatePriceCents, slotMinutes }]` per doctor. The old single fields remain accepted for doctors that keep them.
  - Two or three demo doctors get a second specialty, for example a cardiologist with Clínico Geral at 20 min.
  - V6 inserts the same extra rows in production, matching the demo doctors by their fictitious CRM and the demo clinic name. It uses `INSERT … SELECT … WHERE NOT EXISTS`, so it is a no-op on an empty database (the seeder runs after Flyway and creates them from the JSON).

- [ ] Tests first:
  - `DemoDataSeederIT.seedsSecondSpecialty`: a fresh DB contains a demo doctor with 2 specialties;
  - `V6IsNoOpOnEmptyDatabase`: it migrates cleanly with no clinics.
- [ ] Run: the new tests fail.
- [ ] Implement.
- [ ] `./mvnw verify` passes.
- [ ] Commit: `feat: show multi-specialty doctors in the demo data`.

---

## Web tasks (`agendoc-web`)

Components live in `src/features/clinic-panel/` (panel) and `src/features/{scheduling,appointments}/` (patient). Use `lucide-react` icons, never emojis. Use the tokens from `src/styles/tokens.css`. `:global(.ad-field)` is needed inside modules. In tests, user-event cannot type into time or datetime inputs, so use `fireEvent.change`.

### Task 6: Contracts and mock

**Branch:** `feat/panel-v2-06-contracts`

**Files:**
- Modify:
  - `src/features/clinic-panel/types.ts` and `api.ts`;
  - `src/features/scheduling/types.ts` and `api.ts`;
  - `src/features/appointments/types.ts` and `api.ts`;
  - `src/lib/api/types.ts`, `src/lib/api/errors.ts` and `src/mocks/support.ts` (`SPECIALTY_UNAVAILABLE`);
  - `src/mocks/scheduling.ts`, `src/mocks/panel.ts` and `src/mocks/db.ts`.
- Test: the existing component tests stay green; add `src/mocks/panel.test.ts`.

**Interfaces:**
- **Consumes:** the JSON contracts of Tasks 2 to 4.
- **Produces:**
  - TS types that mirror them exactly (`PanelProfessional.specialties`, `Block.category`, `Agenda.periods`, `PanelAppointment.respondBy/late/specialty/privatePriceCents`, `Appointment.respondBy/late/privatePriceCents`, `Slot.specialty`, `Conflicts.appointments`).
  - API helpers: `getBlocks(scope)` and `book(..., specialty)`.
  - The mock implements the same rules:
    - deadline and late state (a simplified working-day calculation is fine, but `respondBy` and `late` must be present);
    - specialty grid and overlap;
    - categories, scopes and the conflict list.
  - The mock clinic gets one doctor with two specialties, one late request and one block of each kind of scope.
  - Existing screens keep compiling. The minimal adaptations needed (for example `expiresAt` → `respondBy`) are done here, without visual redesign.

- [ ] Write `panel.test.ts`:
  - `lateRequestComesFirst`;
  - `conflictsListPatients`;
  - `pastScopeOnlyEndedBlocks`.
- [ ] Run: it fails.
- [ ] Update the types, API and mock.
- [ ] `npm run ci` passes.
- [ ] Commit: `feat: align web contracts and mock with panel v2 API`.

### Task 7: Patient side: specialty and deadline

**Branch:** `feat/panel-v2-07-patient`

**Files:**
- Modify: `features/scheduling/ClinicView.tsx`, `ClinicCard.tsx` and `SlotPicker.tsx`, and `features/appointments/ConfirmView.tsx`, `AppointmentView.tsx`, `AppointmentList.tsx` and `status.ts`, with their tests.

**Interfaces:**
- **Consumes:** Task 6.
- **Behavior:**
  - The searched specialty's price and duration are shown, and `specialty` is sent when booking.
  - "Médico · Especialidade" appears everywhere.
  - `PENDING` and not late: "Aguardando a clínica · responde até {seg 18:00}".
  - `PENDING` and late: "A clínica ainda não respondeu", with the button "Escolher outro horário", which cancels the request and opens the clinic's slots for the same specialty.
    - For a pending **reschedule**, it cancels only the request and opens the reschedule of the original.
  - `EXPIRED` keeps the current notice.

- [ ] Tests first:
  - `ConfirmView`: `sendsSearchedSpecialty`;
  - `AppointmentView`:
    - `showsRespondByWhilePending`;
    - `lateRequestOffersAnotherTime`;
    - `lateRescheduleKeepsOriginal`;
  - `ClinicView`: `showsPriceOfSearchedSpecialty`.
- [ ] Run: they fail.
- [ ] Implement.
- [ ] `npm run ci` passes.
- [ ] Commit: `feat: show specialty and clinic deadline to patients`.

### Task 8: Panel foundations

**Branch:** `feat/panel-v2-08-foundations`

**Files:**
- Create in `features/clinic-panel/`:
  - `colors.ts`: `doctorColor(professionals, id)` with a palette of 8 colors with AA contrast, assigned by `createdAt`/`id` order; inactive is gray.
  - `DoctorChips.tsx`: single-select, with an optional "Todos os médicos" chip and a color dot; inactive is dimmed.
  - `ui.module.css`: shared buttons with hover, active and focus states, segmented control, card section header, and the icon-plus-text button alignment.
- Modify: `PanelShell.tsx` and `Panel.module.css` (spacing, page header with a right-side action and a 20px gap below it).

**Interfaces:**
- **Produces:**
  - `doctorColor(list: PanelProfessional[], id: string): string` (a CSS color);
  - `<DoctorChips professionals value onChange allowAll? />`, where `value: string | null` (null = all);
  - the `PageHeader` slot in `PanelPage` (`title`, `lead`, `action?`).

- [ ] Tests first:
  - `colors.test.ts`:
    - `stableByCreationOrder`;
    - `inactiveIsGray`;
  - `DoctorChips.test.tsx`:
    - `allIsSelectedWhenValueNull`;
    - `selectsDoctor`.
- [ ] Run: they fail.
- [ ] Implement.
- [ ] `npm run ci` passes.
- [ ] Commit: `feat: add doctor colors, chips and shared panel controls`.

### Task 9: Agenda and appointment detail

**Branch:** `feat/panel-v2-09-agenda`

**Files:**
- Rewrite: `WeekAgenda.tsx` and `AppointmentPanel.tsx`.
- Create: `TimeGrid.tsx` (columns, hour lines, open periods, block bands, event layout side by side) and `EventCard.tsx`.
- Update: `Panel.module.css`.
- Tests: `WeekAgenda.test.tsx` and `AppointmentPanel.test.tsx`.

**Interfaces:**
- **Consumes:** Tasks 6 and 8.
- **Behavior (reference `panel-agenda.html`, final v3):**
  - Toolbar:
    - the week is written out in full ("9 – 15 de novembro · 2026"), with ‹ › and "Hoje";
    - while the week loads, show a visible loading state on the grid;
    - `DoctorChips` with "Todos os médicos" as the default;
    - a link "N pedidos aguardando resposta" to `/clinica/solicitacoes`.
  - Legend at the top of the calendar card.
  - Time grid:
    - the hours span from the first to the last period of the visible doctors (minimum 08–18);
    - open time comes from `agenda.periods`;
    - a day with no periods gets a narrow column;
    - blocks are hatched bands with "doctor/Clínica inteira · reason";
    - overlapping events go side by side.
  - Event cards follow the Global Constraints UI rules and use the doctor's color. A pending card is dashed with a `?` icon, and a late one shows `!`.
  - Detail panel:
    - `position: sticky`, with its top aligned with the calendar card;
    - avatar initials, status badge and the deadline banner for pending;
    - icon rows: date and duration, doctor · specialty, payment with price or insurer and card, phone with "Ligar" (`tel:`) and "WhatsApp" (`https://wa.me/55…`);
    - a triage card with urgency and the AI notice;
    - the actions in a footer.
  - The "Bloqueios nesta semana" list is removed.
  - Mobile: the grid scrolls horizontally and the detail goes below it.

- [ ] Tests first (adapt the existing ones):
  - `defaultsToAllDoctors`;
  - `weekArrowsLoadNextWeek` (shows the loading state, then the new range text);
  - `cardsStartWithPatientName`;
  - `cardsCarryFullTextInLabel`;
  - `lateRequestShowsAlert`;
  - `blockRendersInsideDay`;
  - `keepsShowingAnAppointmentAfterTheClinicCancelsIt` (existing);
  - `detailShowsDeadlineAndContactActions`.
- [ ] Run: they fail.
- [ ] Implement.
- [ ] `npm run ci` passes.
- [ ] Commit: `feat: redesign clinic agenda as a time grid`.

### Task 10: Requests

**Branch:** `feat/panel-v2-10-requests`

**Files:**
- Modify: `Requests.tsx` and its test. Reuse the detail pieces from Task 9 (`AppointmentPanel`).

**Behavior:**
- The list is ordered by the API: late first.
- Each item shows the patient, doctor · specialty, date and time, and "Responder até…" or a red "prazo vencido".
- The explanatory copy describes the new rule: "Responda até o fim do próximo dia de atendimento do médico. Depois disso o pedido fica atrasado, mas continua valendo até 2 h antes da consulta."
- Accepting or declining refreshes the sidebar badge (`notifyRequestsChanged`).

- [ ] Tests first:
  - `lateRequestsFirstWithAlert`;
  - `explainsDeadlineRule`;
  - `acceptRefreshesBadge`.
- [ ] Run: they fail.
- [ ] Implement.
- [ ] `npm run ci` passes.
- [ ] Commit: `feat: show request deadlines in the clinic panel`.

### Task 11: Doctors

**Branch:** `feat/panel-v2-11-doctors`

**Files:**
- Rewrite: `Professionals.tsx`.
- Create: `SpecialtiesEditor.tsx`.
- Tests: `Professionals.test.tsx` and `SpecialtiesEditor.test.tsx`.

**Behavior (reference `panel-doctors-settings.html`, top part):**
- A card grid (avatar initials in the doctor's color, CRM-UF, status badge, one row per specialty "name · N min · R$ X", the weekly summary computed from the rules) and a sticky side editor.
- The editor has name, CRM, UF, Ativo/Inativo (segmented) and the specialties table (select from the catalog, duration select 10–120 step 5, price in R$, remove ×, "+ Adicionar especialidade").
- Validation messages follow the pinned values.
- The header action "+ Novo médico" (with the icon) has a 20px gap above the content.

- [ ] Tests first:
  - `createsDoctorWithTwoSpecialties`;
  - `blocksDuplicateSpecialty`;
  - `requiresAtLeastOneSpecialty`;
  - `showsWeeklySummary`.
- [ ] Run: they fail.
- [ ] Implement.
- [ ] `npm run ci` passes.
- [ ] Commit: `feat: manage doctors with several specialties`.

### Task 12: Weekly hours

**Branch:** `feat/panel-v2-12-hours`

**Files:**
- Rewrite: `ScheduleRules.tsx`, keeping `validateWeek` exported and extended for the limit of 4 periods.
- Create: `TimeSelect.tsx` (24 h, 15-min steps, 00:00–23:45 for the start, 00:15–23:59 for the end).
- Tests: `ScheduleRules.test.tsx` and `TimeSelect.test.tsx`.

**Behavior (reference `panel-hours.html`, final v3):**
- `DoctorChips` without "all".
- A table with Dia | 1º período | 2º período | (+ período) | Visão do dia.
  - The number of period columns is the maximum over the days (minimum 2).
  - Day tiles show the abbreviation in the doctor's color with ✓ when on, dashed when off; clicking toggles.
  - Period editing happens inside the cell.
  - Errors show on the day row, and the bad period turns red.
- "Copiar segunda para os dias úteis" and "Limpar semana".
- Below the table: the summary (days, hours per week, first and last time) and the doctor's specialties with duration and price.
- A sticky save bar with "Alterações não salvas", "Descartar" and "Salvar horários".
- Switching doctor with unsaved changes asks for confirmation inside the UI (no `window.confirm`).

- [ ] Tests first:
  - `togglesDayWithTile`;
  - `editsPeriodInsideCell`;
  - `copiesMondayToWeekdays`;
  - `flagsOverlapOnRow`;
  - `asksBeforeDiscardingUnsavedHours`;
  - `noAmPmInTimeSelect`;
  - the existing save test, adapted.
- [ ] Run: they fail.
- [ ] Implement.
- [ ] `npm run ci` passes.
- [ ] Commit: `feat: redesign weekly hours editor`.

### Task 13: Blocks

**Branch:** `feat/panel-v2-13-blocks`

**Files:**
- Rewrite: `Blocks.tsx`.
- Create: `BlockForm.tsx`.
- Test: `Blocks.test.tsx`.

**Behavior (reference `panel-blocks.html`, final v2):**
- Tabs "Próximos · N" and "Passados" (`scope`), and `DoctorChips` plus a "Clínica inteira" filter.
- The list is grouped by month (label of fixed height). Each item has the three zones:
  - **when:** a date tile, written "SEG–QUA / 09–11 / NOV" for ranges with no clipping, plus the bold time and duration, or "Dia inteiro";
  - **who:** a chip in the doctor's color, or the dark "Clínica inteira";
  - **why:** the category tag plus the text, or "Sem motivo informado".
  - Each item also has a relative chip ("começa amanhã", "em 3 dias", "agora") and a delete button.
- The sticky side form starts at the same height as the first item (offset by the month label).
  - It has "Clínica inteira / Um médico", "Dia inteiro", dates and times, category presets that also fill the editable text, and the conflict warning listing the patients, with "Cancelar e avisar os pacientes" and a required reason.
- Cancelling conflicts refreshes the requests badge.

- [ ] Tests first:
  - `groupsByMonthWithThreeZones`;
  - `pastTabLoadsPastScope`;
  - `presetSetsCategoryAndText`;
  - `wholeDayBlock`;
  - `listsConflictingPatientsAndCancelsWithReason` (adapt the existing test).
- [ ] Run: they fail.
- [ ] Implement.
- [ ] `npm run ci` passes.
- [ ] Commit: `feat: redesign schedule blocks`.

### Task 14: Clinic settings

**Branch:** `feat/panel-v2-14-settings`

**Files:**
- Modify: `Settings.tsx` and its test.
- Create: `InsurerPicker.tsx` and `GuidelinesEditor.tsx`.

**Behavior (reference `panel-doctors-settings.html`, bottom part):**
- Insurer cards:
  - a monogram in our palette and a ✓ when selected;
  - "Particular" is locked as always accepted;
  - the counter reads "N de 7 convênios";
  - the cards are real checkboxes for accessibility.
- Confirmation mode as two radio cards. The MANUAL copy mentions the deadline rule.
- The guidelines are a numbered list:
  - drag handle plus "Subir" and "Descer" buttons;
  - remove and add;
  - at most 10 items of 200 characters.
- Phone and WhatsApp keep their fields in the same card style.

- [ ] Tests first:
  - `togglesInsurerCard`;
  - `particularIsLocked`;
  - `reordersGuidelineWithButtons`;
  - `savesConfirmationMode`.
- [ ] Run: they fail.
- [ ] Implement.
- [ ] `npm run ci` passes.
- [ ] Commit: `feat: redesign clinic settings`.

### Task 15: Browser pass and test data

No branch of its own; fixes found here go on `fix/panel-v2-*` branches.

- [ ] Restart the API (8080) and the web (3000) on `feat/clinics-scheduling`.
- [ ] Update the local test clinic data (scratchpad `seed_test_clinic.py`) so that:
  - Camila also has Clínico Geral at 20 min / R$ 200;
  - a few blocks have categories;
  - there is at least one late request.
- [ ] In Chrome, as the clinic, walk through every screen at desktop width and at 390px.
  - Check the agenda arrows, the "Todos" default, unclipped cards, the sticky detail, blocks in the grid, the late request, the hours editor (no AM/PM), the block form with conflicts, doctors with two specialties, and the settings cards.
  - Check that there are no console errors.
- [ ] As a patient: search a second specialty, book it (MANUAL clinic: the first slot respects the 4 working hours), and see "responde até…".
- [ ] Gates on both repos.
- [ ] Ledger entry and report to the user.

---

## Final review

After Task 15, run one whole-branch review of both repos against the spec, as the execution skill describes. Then report to the user and wait for milestone approval before merging into `develop`.
