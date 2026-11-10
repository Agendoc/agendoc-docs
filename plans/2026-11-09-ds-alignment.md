# Alinhamento do web ao design system

Spec: o design system AgenDoc (https://claude.ai/artifact/XkY9gE2GgJdWnzRc3GPcJ4) e a auditoria de 09/11/2026, que achou cinco grupos de divergência no app do paciente. Os componentes sem tela no app (TriageQuestion, SearchField, CompareTable, ReminderItem, TopBar brand) ficam de fora: são funcionalidade nova. O painel da clínica (`/clinica/*`) também fica de fora.

Repositório: `agendoc-web` só. Integração em `feat/ds-alignment` (a partir da `develop`), uma branch por tarefa (`feat/ds-NN-*`).

## Global Constraints

- O DS é a referência: classes `ad-*` e valores do `bundle.css`, markup e props do `bundle.js`/`index.d.ts`, regras do `README.md` e de cada `components/<Nome>/README.md`. Cópia local em `DS_DIR` (informada no dispatch).
- Componente do DS entra como componente React em `src/components/ui/<Nome>.tsx`, com o CSS do `bundle.css` copiado sem mudanças para `src/styles/components.css` (como já foi feito com Button, Notice etc.). Nada de recriar o visual num CSS module de feature.
- Só tokens de `src/styles/tokens.css`. Nada de hex, raio ou tamanho soltos.
- Texto secundário em `--ink-muted`; `--ink-subtle` só em placeholder e desabilitado.
- Ícones do `lucide-react`. Sem caracteres "★".
- Formatos: `R$ 180,00` (sempre com centavos), `1,2 km`, `4,8`, `ter, 10 nov`, `07:00`.
- Rodapé: BottomNav ou botão primário fixo, nunca os dois.
- Comportamento e rotas atuais continuam iguais, salvo o que a tarefa muda. O mock MSW continua funcionando.
- Gates: `npm run ci` verde. Testes de componente para todo componente novo e para os comportamentos alterados. Nunca rodar `npx prettier`.
- Código em inglês, interface em pt-BR, commits semânticos curtos em inglês com o trailer `Co-Authored-By`.

## Task 1: Componentes do DS que faltam

Portar para `src/components/ui` (TSX + CSS do bundle), com testes: `Chip`, `SegmentedControl`, `Switch`, `Rating`, `ProgressSteps`, `SpecialtyCard`, `DateStrip`, `TimeSlotPicker`, `ClinicCard`, `AppointmentCard`. `Badge` ganha o tom `brand`. `BottomNav` passa a seguir o DS (pílula `blue-soft` no item ativo, ícone `blue-strong`, rótulo em negrito, inativos `ink-muted`), mantendo os 3 destinos atuais (Conversas, Consultas, Perfil) e seus ícones.

- Props como em `index.d.ts`, adaptadas ao app: ícones como `LucideIcon` (igual ao `Badge` atual); `ClinicCard` e `AppointmentCard` aceitam conteúdo de ação (`ReactNode`) em vez de callbacks fixos, para as telas passarem `Link`s.
- Este passo só cria os componentes; as telas trocam nas tarefas seguintes.

## Task 2: Formatos, cores e selos

- `formatPrice` sempre com centavos (`R$ 130,00`); ajustar testes.
- Datas curtas do DS (`ter, 10 nov`) no botão "Continuar" da clínica, na confirmação e na remarcação, no lugar do dia por extenso.
- Trocar `--ink-subtle` por `--ink-muted` nos textos secundários dos CSS modules do paciente (`features/scheduling`, `appointments`, `reviews`, `profile`, `chat`).
- Estrelas: nota exibida usa `Rating`; as estrelas de dar nota (`features/reviews/StarRating.tsx`) e os marcadores do mapa usam o ícone `Star` do Lucide, não "★".
- Selos feitos à mão viram `Badge`: "Dados de demonstração" (`DemoBadge`), "em breve" e a urgência do `TriageCard`.
- Selects nativos (`.ad-field__input` em `<select>` e `.selectChip select`): seta própria (`ChevronDown`) e o anel de foco do DS.

## Task 3: Busca de clínicas

`SearchView`/`features/scheduling/ClinicCard.tsx` passam a usar o `ClinicCard` do DS: iniciais em navy, `Rating` abaixo do nome, distância e endereço com ícones, "Próximo horário: <primeiro horário>" em `teal-ink`, convênios como `Badge` neutro, preço "a partir de" em destaque e um botão "Agendar" que leva à página da clínica. Sem "Comparar". A clínica escolhida no mapa usa a borda do estado selecionado. Os chips de filtro passam a usar `Chip`. `ProgressSteps` no topo: passo 2 de 4, "Clínica".

## Task 4: Minhas consultas

`AppointmentList` usa `SegmentedControl` (Próximas/Anteriores) e `AppointmentCard`: bloco de data, `Badge` de status, especialidade e hora, médico e endereço com ícones. Consultas futuras não canceladas mostram "Como chegar" (mesmas opções de rota do detalhe) e "Remarcar"; o nome da especialidade é o link para o detalhe (o card deixa de ser um link inteiro). A caixa "Nova consulta" vira uma lista de `SpecialtyCard` (ícones do README do DS: cardiologia `HeartPulse`, neurologia `Brain`, oftalmologia `Eye`, ortopedia `Bone`, pediatria `Baby`, dermatologia `ScanFace`, demais `Stethoscope`; descrição curta em linguagem do dia a dia) que leva à busca com a especialidade. O detalhe da consulta usa o mesmo card.

## Task 5: Agendamento e conversas

- Página da clínica: `DateStrip` e `TimeSlotPicker` do DS no lugar de `DayStrip`/`SlotPicker` (mantendo os grupos por médico); `ProgressSteps` passo 3 de 4, "Horário".
- Confirmação e remarcação: `ProgressSteps` passo 4 de 4, "Confirmar" (só na confirmação); lembrete por e-mail com `Switch`; WhatsApp como `Switch` desabilitado com `Badge` "Em breve".
- `/chat`: sai o botão fixo "Nova conversa" do rodapé; entra um `IconButton` "+" (rótulo "Nova conversa") na TopBar. O EmptyState continua com sua ação.
