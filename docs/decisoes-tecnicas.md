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
