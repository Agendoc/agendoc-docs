# Roadmap

Status: `em andamento` · `próximo` · `ideia` · `feito`

| Item | Status | Onde |
|---|---|---|
| Parte 1: Clínicas e agendamento (busca, marcar, minhas consultas, painel da clínica, avaliações, emergência, .ics) | feito | [spec](specs/2026-11-01-clinics-scheduling-design.md) · [plano](plans/2026-11-01-clinics-scheduling.md) · [progresso](#progresso-da-parte-1) · [backlog](backlog-parte-1.md) |
| Parte 2: E-mail: confirmação de e-mail, recuperação de senha, resumo da triagem, lembretes | próximo | — |
| Parte 3: WhatsApp: estudar integração, depois resumo e lembretes | próximo | — |
| Chat entre clínica e paciente dentro do app | ideia | — |
| Chatbot de ajuda na tela de busca de clínicas | ideia | — |
| Vários usuários por clínica / login por médico | ideia | — |
| Exclusão de conta (LGPD) | ideia | — |

## Progresso da parte 1

Os marcos aprovados entraram na branch `develop` de `agendoc-api` e `agendoc-web`. No fim da parte, `develop` foi para a `main` (PR #3 nos dois repositórios), e o deploy foi feito em 09/11/2026.

| Marco | Conteúdo | Status |
|---|---|---|
| M1 | Modelo e seed, busca com mapa e filtros, página da clínica e horários, marcar e confirmar, emergência | feito e aprovado (07/11/2026), na `develop` |
| M2 | Minhas consultas, cancelar, Perfil com geocodificação, remarcar | feito e aprovado (07/11/2026), na `develop` |
| M3 | Painel da clínica (com o redesenho v2: várias especialidades por médico, prazo de resposta, categorias de bloqueio), endpoint admin, expiração dos pendentes | feito e aprovado (08/11/2026), na `develop` |
| M4 | Avaliações, documentação final, revisão e deploy | feito e aprovado (09/11/2026), no ar desde 09/11/2026 |
