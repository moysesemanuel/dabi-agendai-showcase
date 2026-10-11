# DaBi Agendaí

Sistema de agendamento multi-tenant para prestadores de serviço com atendimento por horário — barbearias, salões, estúdios e clínicas. Cada estabelecimento tem seu próprio subdomínio, com site institucional, agendamento online, área do cliente, cobrança recorrente e um backoffice completo.

**Stack:** Next.js 16 (App Router) · React 19 · TypeScript · Prisma · PostgreSQL · Vercel

---

## O problema

Quem atende por horário marcado normalmente opera no WhatsApp e no caderno. O custo disso aparece como conflito de agenda, tempo perdido confirmando horário por mensagem, cliente que não aparece e nenhuma visão do próprio negócio.

O DaBi Agendaí estrutura essa operação: o cliente agenda sozinho, o estabelecimento vê a agenda em tempo real, o histórico fica registrado e o dono é avisado na hora de cada novo agendamento — sem precisar ficar de olho no WhatsApp.

---

## Módulos

| Módulo | O que faz |
|---|---|
| **Planos** | Básico, Pro e Max — recursos liberados progressivamente (mini site → site completo, produtos, fidelidade). Mensal ou anual, com upgrade/downgrade self-service pelo próprio painel |
| **Site institucional** | Home com serviços, equipe, prova social e contato — um subdomínio por barbearia. Mini site no Básico, site completo a partir do Pro |
| **Cadastro self-service** | A barbearia se cadastra sozinha e já entra no ar com **15 dias de teste grátis, sem cartão** — escolhe assinar (ou não) só depois de usar |
| **Agendamento** | Serviço → profissional → data → horário, com bloqueio de horário ocupado e carrinho para múltiplos serviços |
| **Área do cliente** | Cadastro, login, histórico, cancelamento, remarcação e recuperação de senha por e-mail |
| **Fidelidade** | Pontuação por atendimento, níveis e recompensas configuráveis — exclusivo do plano Max |
| **Backoffice** | Agenda em grade (dia/3 dias/semana), catálogo de serviços e produtos, equipe, datas bloqueadas, dados de clientes/receita e FAQ |
| **Cobrança** | Assinatura recorrente via Mercado Pago, com bloqueio automático de acesso em caso de inadimplência |
| **Notificações** | Aviso automático de novo agendamento por Web Push, Telegram e WhatsApp, mesmo com o painel fechado |
| **App instalável (PWA)** | Painel e site público podem ser adicionados à tela inicial (Android e iPhone) e abrir como app, com ícone e cor da própria barbearia |

---

## O que já funciona hoje

| Área | Estado |
|---|---|
| Multi-tenant real (subdomínio e dados isolados por barbearia) | ✅ |
| Autenticação e autorização (admin e cliente, por tenant) | ✅ |
| 3 planos (Básico/Pro/Max) com recursos liberados por plano | ✅ |
| Cadastro self-service com 15 dias de teste grátis, sem cartão | ✅ |
| Cobrança recorrente via Mercado Pago, com bloqueio por inadimplência | ✅ |
| Troca de plano e de ciclo (mensal/anual) pelo próprio painel | ✅ |
| Notificação automática de novo agendamento (Web Push, Telegram, WhatsApp) | ✅ |
| App instalável (PWA) no painel e no site público | ✅ |
| Configuração do site persistida no banco, por tenant | ✅ |
| Testes automatizados (`yarn test:unit`) | ✅ |
| Testes end-to-end (`yarn test:e2e`) | ⛔ configurado, cobertura ainda mínima |

---

## Rodando localmente

Pré-requisitos: Node.js 20+, Yarn 1.x, PostgreSQL.

```bash
yarn install
cp .env.example .env   # preencha as variáveis (banco, sessão, Mercado Pago, Resend, VAPID, Telegram, WhatsApp)
yarn db:push
yarn dev
```

O schema não usa `prisma migrate` — mudanças de schema são aplicadas com `yarn db:push` (dev) e via SQL direto em produção.

---

## Roadmap

Plano de expansão dividido em ondas — detalhes em cada issue do [board do projeto](https://github.com/users/moysesemanuel/projects/6).

- **Onda 2 — Gestão do dia a dia**: base de clientes ligada a todo agendamento (feito), ficha de cliente com histórico e busca, comandas (serviços + produtos, desconto, fechamento), comissão por profissional, acesso próprio por profissional, dashboard financeiro
- **Onda 3 — Receita recorrente**: pagamento online pelo cliente final, pacotes de serviços, assinaturas de clientes (clube com cobrança recorrente)
- **Onda 4 — Fazer o cliente voltar**: campanhas automáticas e mensagens prontas por WhatsApp, fila de espera, avaliações pós-atendimento, agendamento recorrente
- Ampliar cobertura de testes end-to-end
- Adaptar copy e FAQ para outros tipos de negócio além de barbearia

---

## Licença e contato

Projeto proprietário da DaBi Tech — Digital Solutions (Moyses Emanuel - Desenvolvedor Full-Stack).

- GitHub: https://github.com/moysesemanuel
- LinkedIn: https://www.linkedin.com/in/moysesemanuel
