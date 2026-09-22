# Testes e CI

## Testes

**Unitários (Vitest):** cerca de 45 casos em 9 suítes, cobrindo:

- regras de agendamento e cálculo de horários;
- serviço de assinatura e integração com o Mercado Pago;
- a guarda de fuso horário, que varre o código-fonte;
- formatadores e máscaras de moeda usados no backoffice e na área do cliente.

**End-to-end (Playwright):**

- *smoke test* dos fluxos principais;
- fluxo completo de cancelamento de agendamento.

## Pipeline

O pipeline tem dois jobs em sequência, disparados em toda PR e a cada push na `main`:

**1. Checks:** instalação com lockfile congelado, lint, typecheck e testes unitários.

**2. E2E (só roda se os checks passarem):**

1. Sobe um PostgreSQL 16 como serviço do GitHub Actions.
2. Gera credenciais efêmeras para o job (segredo de sessão e senhas de admin), que nunca são segredos reais.
3. Aplica o schema no banco de teste.
4. Cria um estabelecimento de teste pelo mesmo script usado para cadastrar tenants reais.
5. Ativa a assinatura desse estabelecimento, para o backoffice não ficar bloqueado pela cobrança.
6. Gera o build de produção e roda os testes E2E com Playwright no Chromium.
7. Se algo falhar, publica o relatório do Playwright como artefato por 7 dias.

## Jobs agendados

Os lembretes por WhatsApp são disparados por um workflow agendado a cada 15 minutos, que chama uma rota protegida por token.
