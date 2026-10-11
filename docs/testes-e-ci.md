# Testes e CI

## Testes

**Unitários (Vitest):** cerca de 660 casos em 90 arquivos, cobrindo:

- regras de agendamento, cálculo de horários, carrinho e conflitos entre profissionais;
- cobrança da assinatura, troca de plano, renovação e reconciliação com o Mercado Pago;
- pagamento online do cliente final: preço, devolução por serviço, cancelamento de compra e pagamentos com problema;
- clube de assinatura, pacotes, fila de espera e recorrência;
- cargos e permissões, incluindo a superfície de telas e rotas de cada cargo;
- guardas que leem o código-fonte: isolamento entre tenants e fuso horário;
- formatadores, máscaras de moeda e utilitários.

**Testes que fazem varredura do código.** Dois testes não executam funções: eles leem o código-fonte e o schema. Um falha se uma consulta a tabela com tenant não filtrar pelo tenant. O outro falha se alguém usar `UTC` sem justificativa. Servem para impedir regressões que ninguém notaria em revisão.

**End-to-end (Playwright):** *smoke test* dos fluxos principais e fluxo completo de cancelamento de agendamento.

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

## Rotinas agendadas

Nove workflows no GitHub Actions chamam rotas protegidas por token, em intervalos que vão de 5 minutos a uma vez por mês:

- lembretes de agendamento por WhatsApp;
- expiração de pagamentos pendentes;
- renovação e cobrança da mensalidade do clube;
- geração de agendamentos recorrentes;
- avisos da fila de espera;
- reconciliação da assinatura com o Mercado Pago;
- controle de créditos de WhatsApp;
- relatório mensal por e-mail.
