# Decisões técnicas

## 1. Transação Serializable para impedir agendamento duplicado

**Problema.** Dois clientes abrem a mesma agenda e clicam no mesmo horário quase ao mesmo tempo. Uma checagem simples ("o horário está livre?") seguida de uma gravação deixa uma janela em que os dois passam pela checagem e os dois gravam.

**Decisão.** Criar e remarcar agendamentos dentro de uma transação interativa com isolamento `Serializable`. Dentro dela, o sistema relê os agendamentos do profissional naquele dia, verifica sobreposição de intervalo e só então grava. Se duas transações concorrentes entram em conflito, o Postgres aborta uma delas, e esse erro é traduzido para o usuário como "esse horário acabou de ser reservado, escolha outro".

**Por quê.** Com `Serializable`, a garantia vem do banco, e não de uma disciplina no código. É a opção mais simples e segura para uma regra de negócio central do produto.

## 2. Driver da Neon em modo WebSocket

**Contexto.** O driver serverless da Neon tem dois modos. O modo HTTP é mais leve, mas só executa lotes de statements prontos, sem transações interativas. A decisão anterior exige transação interativa.

**Decisão.** Usar o adaptador da Neon para Prisma em modo WebSocket. Em desenvolvimento e no CI, onde o banco é um Postgres comum em container, o adaptador não é usado.

**Lição de operação.** Houve uma instabilidade em produção que, num primeiro momento, pareceu ser culpa do adaptador. Voltar ao driver padrão não resolveu, e a investigação mostrou que a causa real eram URLs de conexão desatualizadas na configuração do deploy. Depois de corrigir, o adaptador voltou, agora com uma chave de desligamento por variável de ambiente. Se ele der problema de novo, dá para desativá-lo com um redeploy, sem precisar reverter código às pressas.

## 3. Fuso horário tratado como regra, não como detalhe

**Problema.** Servidores na nuvem rodam em UTC. Às 22h em São Paulo já é o dia seguinte em UTC, e um agendamento feito à noite pode cair no dia errado.

**Decisão.** Todas as datas de agenda são formatadas explicitamente no fuso `America/Sao_Paulo`. Para impedir regressões, um teste percorre o código-fonte e falha se encontrar `timeZone: "UTC"` sem um comentário de justificativa na mesma linha.

**Por quê.** É um erro fácil de reintroduzir sem perceber, e difícil de notar em testes manuais durante o dia. Transformar a regra em teste automatizado tira a responsabilidade da memória de quem está codando.

## 4. Notificações fora do caminho crítico

**Decisão.** Depois que o agendamento é gravado, os avisos ao dono e ao cliente são disparados de forma assíncrona. Falhas são registradas, mas não afetam a resposta ao cliente.

**Por quê.** Um serviço externo fora do ar (Telegram, push, WhatsApp) não pode impedir alguém de agendar.

## 5. Isolamento entre tenants garantido por teste

**Problema.** Todos os estabelecimentos dividem o mesmo banco. O que os separa é a coluna do tenant, e basta uma consulta sem esse filtro para um estabelecimento ver dados de outro. Revisão de código não pega isso de forma confiável.

**Decisão.** Um teste automatizado lê o schema, descobre quais tabelas têm a coluna do tenant e percorre o código-fonte. Ele falha se uma consulta a essas tabelas não citar o tenant no próprio `where`, inclusive em `update` e `delete` por id, para que uma busca anterior removida por engano não vire vazamento. Casos que operam entre tenants de propósito, como webhooks e rotinas diárias, precisam estar numa lista explícita ou ter um comentário com a justificativa.

**Por quê.** A regra de segurança mais importante de um SaaS multi-tenant passa a ser verificada em toda PR, em vez de depender da atenção de quem escreve ou revisa.

## 6. Pagamentos que se corrigem sozinhos

**Contexto.** Webhooks chegam fora de ordem, chegam repetidos e às vezes não chegam. Dinheiro não tolera nenhum dos três.

**Decisão.** Três camadas:

- Cada evento é gravado com unicidade por provedor e id do evento, então reprocessar é seguro.
- Rotinas agendadas consultam o provedor e fecham o que o webhook perdeu, como pagamentos pendentes e assinaturas fora de sincronia.
- Quando algo realmente falha, o sistema não esconde o erro: cria um registro para o dono do estabelecimento, avisa por e-mail e à equipe, e envia o evento ao Sentry.

**Por quê.** O objetivo não é que nunca falhe, e sim que toda falha seja detectada, seja corrigível sem intervenção manual quando possível, e fique visível quando não for.

## 7. Contas de terceiros conectadas por OAuth

**Contexto.** O pagamento do cliente final precisa cair na conta do próprio estabelecimento, sem que a plataforma segure o dinheiro nem peça credenciais.

**Decisão.** Cada estabelecimento autoriza a plataforma por OAuth no Mercado Pago. Os tokens são guardados criptografados (AES-256-GCM, com IV aleatório e tag de autenticação). Quando o acesso é revogado ou recusado, o sistema detecta, avisa o dono e oferece o fluxo de reconexão, em vez de falhar em silêncio no pagamento do cliente. Há também um modo de teste com pagador fictício e um script para conectar uma conta de teste localmente.

**Por quê.** Reduz o risco e a responsabilidade da plataforma e deixa a revogação do acesso como um estado previsto do sistema, não como um erro inesperado.

## 8. Dinheiro como inteiro e devolução por serviço

**Problema.** Uma compra pode juntar vários serviços pagos de uma vez, e o cliente pode cancelar só um deles, ou a compra inteira, com alguns horários já realizados.

**Decisão.** Valores são sempre inteiros em centavos, nunca ponto flutuante. A compra é um grupo de pagamentos com total e subtotais, e a devolução é calculada por serviço. Ao cancelar a compra inteira, serviços cujo horário já passou ficam de fora do estorno.

**Por quê.** Evita erro de arredondamento e deixa a regra de estorno explícita e testável, em vez de espalhada pelas telas.
