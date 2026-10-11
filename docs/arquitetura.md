# Arquitetura

## Visão geral

Uma única aplicação Next.js atende todos os estabelecimentos. O que muda de um para outro é o **tenant**, identificado pelo domínio da requisição. Cada estabelecimento tem seu próprio site, sua agenda, seus clientes, sua configuração e sua assinatura, sem enxergar os dados dos outros.

## Multi-tenancy

- **Resolução por domínio.** O host da requisição é comparado com o domínio cadastrado do tenant. Tenants inativos não são servidos.
- **Isolamento nos dados.** Todas as entidades de negócio (profissionais, serviços, produtos, clientes, agendamentos, pagamentos, folgas, datas fechadas, notificações) carregam o identificador do tenant. Um teste automatizado impede consultas sem esse filtro.
- **Unicidade por tenant.** Restrições únicas combinam tenant e campo, como `(tenant, telefone)` e `(tenant, nome do serviço)`. O mesmo cliente pode existir em estabelecimentos diferentes sem conflito.
- **Índices pensados para a agenda.** A busca de conflitos usa um índice composto por tenant, profissional e intervalo de horário.
- **Previews funcionais.** Deploys de preview da Vercel têm domínio aleatório. Um fallback controlado, que só existe em preview, permite testar cada PR sem cadastrar um tenant manualmente.

## Perfis de acesso

| Perfil | Onde atua | Sessão |
|---|---|---|
| Cliente final | Site, agendamento e área do cliente do estabelecimento | Sessão do tenant |
| Equipe do estabelecimento | Backoffice, com telas e ações limitadas pelo cargo (Dono, Gerente, Recepção, Profissional) | Sessão do tenant, com cargo |
| Admin da plataforma | Console que opera todos os tenants | Sessão própria, separada da dos tenants |

As permissões ficam em um catálogo único que define o que cada cargo vê e faz, e a mesma regra vale para menus, páginas e rotas de API. Testes travam essa superfície por cargo.

A proteção das rotas administrativas acontece no middleware, que roda no Edge. Como o Edge não acessa o banco, o middleware valida apenas a sessão assinada, e as verificações que dependem de dados ficam nas camadas que rodam em Node.

## Agendamento

1. O cliente escolhe serviço, profissional, data e horário, podendo montar um carrinho com vários serviços.
2. Os horários disponíveis consideram o horário de funcionamento configurado pelo estabelecimento, a duração do serviço, folgas do profissional, datas fechadas e agendamentos existentes.
3. A gravação acontece em transação `Serializable`, como descrito em [decisoes-tecnicas.md](decisoes-tecnicas.md).
4. Depois de gravar, as notificações ao dono e ao cliente são disparadas de forma assíncrona. Uma falha de notificação nunca desfaz o agendamento.

## Pagamentos

Há duas camadas independentes, ambas com Mercado Pago:

**Assinatura da plataforma.** Cada estabelecimento tem uma assinatura recorrente (mensal ou anual) com período de teste. Quando a assinatura não está ativa, o backoffice é bloqueado automaticamente. Uma rotina de reconciliação compara o estado local com o do provedor.

**Pagamento do cliente final.** Cada estabelecimento conecta a própria conta do Mercado Pago por OAuth, e o dinheiro cai direto nela. O cliente paga por Pix ou cartão no agendamento, na compra de pacotes e na mensalidade do clube.

- Os tokens de cada estabelecimento ficam criptografados no banco (AES-256-GCM), e a revogação do acesso dispara um fluxo de reconexão.
- Valores são sempre inteiros em centavos.
- Eventos do provedor são registrados com unicidade por provedor e id do evento, o que garante idempotência no processamento de webhooks.
- Compras com vários serviços são agrupadas, com total e subtotais, e a devolução é feita por serviço.
- Pagamentos com problema viram um registro para o dono, com e-mail, notificação à equipe e evento no Sentry.
- A comanda abate o que já foi pago online, e os relatórios separam a parte online da recebida no balcão.

## Notificações

| Canal | Uso |
|---|---|
| Web Push | Alerta de novo agendamento para o dono, mesmo com o painel fechado |
| Telegram | Alerta de novo agendamento, com vínculo da conta por código |
| WhatsApp | Lembretes e avisos aos clientes, disparados por jobs agendados, com controle de créditos por estabelecimento |
| E-mail (Resend) | Recuperação de senha e mensagens transacionais |

**Rotinas agendadas.** Lembretes, renovações, recorrências, fila de espera, reconciliação e relatório mensal rodam em workflows agendados do GitHub Actions, que chamam rotas protegidas por token.
