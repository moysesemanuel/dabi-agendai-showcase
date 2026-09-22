# Arquitetura

## Visão geral

Uma única aplicação Next.js atende todos os estabelecimentos. O que muda de um para outro é o **tenant**, identificado pelo domínio da requisição. Cada estabelecimento tem seu próprio site, sua agenda, seus clientes, sua configuração e sua assinatura, sem enxergar os dados dos outros.

## Multi-tenancy

- **Resolução por domínio.** O host da requisição é comparado com o domínio cadastrado do tenant. Tenants inativos não são servidos.
- **Isolamento nos dados.** Todas as entidades de negócio (profissionais, serviços, produtos, clientes, agendamentos, folgas, datas fechadas, notificações) carregam o identificador do tenant.
- **Unicidade por tenant.** Restrições únicas combinam tenant e campo, como `(tenant, telefone)` e `(tenant, nome do serviço)`. O mesmo cliente pode existir em estabelecimentos diferentes sem conflito.
- **Índices pensados para a agenda.** A busca de conflitos usa um índice composto por tenant, profissional e intervalo de horário.
- **Previews funcionais.** Deploys de preview da Vercel têm domínio aleatório. Um fallback controlado, que só existe em preview, permite testar cada PR sem cadastrar um tenant manualmente.

## Três perfis de acesso

| Perfil | Onde atua | Sessão |
|---|---|---|
| Cliente final | Site, agendamento e área do cliente do estabelecimento | Sessão do tenant |
| Dono / administrador | Backoffice do próprio estabelecimento | Sessão do tenant, com papel de admin |
| Admin da plataforma | Console que opera todos os tenants | Sessão própria, separada da dos tenants |

A proteção das rotas administrativas acontece no middleware, que roda no Edge. Como o Edge não acessa o banco, o middleware valida apenas a sessão assinada, e as verificações que dependem de dados ficam nas camadas que rodam em Node.

## Agendamento

1. O cliente escolhe serviço, profissional, data e horário.
2. Os horários disponíveis consideram o horário de funcionamento configurado pelo estabelecimento, a duração do serviço, folgas do profissional, datas fechadas e agendamentos existentes.
3. A gravação acontece em transação `Serializable`, como descrito em [decisoes-tecnicas.md](decisoes-tecnicas.md).
4. Depois de gravar, as notificações ao dono e ao cliente são disparadas de forma assíncrona. Uma falha de notificação nunca desfaz o agendamento.

## Cobrança

- Assinatura recorrente por estabelecimento, com planos definidos no produto.
- Eventos do Mercado Pago são registrados com unicidade por provedor e ID do evento, o que garante idempotência no processamento de webhooks.
- O backoffice é bloqueado automaticamente quando a assinatura não está ativa.

## Notificações

| Canal | Uso |
|---|---|
| Web Push | Alerta de novo agendamento para o dono, mesmo com o painel fechado |
| Telegram | Alerta de novo agendamento, com vínculo da conta por código |
| WhatsApp | Lembretes automáticos aos clientes, disparados por um job agendado |
| E-mail (Resend) | Recuperação de senha e mensagens transacionais |
