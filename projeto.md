# Plano Detalhado do Projeto EstudeiCards Agentes

## Visão Geral do Projeto

O EstudeiCards Agentes é um sistema de gamificação de conhecimento DevOps que envia aos usuários cadastrados três perguntas diárias sobre temas do ecossistema DevOps. As perguntas têm três níveis de dificuldade – Fácil, Pleno e Expert – estimulando aprendizado progressivo.

As questões podem ser enviadas via WhatsApp (utilizando a plataforma Zaia) e, se viável, via mensagem direta no LinkedIn. O diferencial é que as perguntas e correções serão geradas dinamicamente com apoio de modelos de LLM (como OpenAI GPT-4 ou Google Gemini), usando uma base de dados de referência para garantir acurácia.

O objetivo do projeto é engajar gratuitamente a comunidade DevOps User Group Brazil (DOUG BR) em uma competição saudável, 100% automatizada, que incentiva estudo contínuo através de quizzes diários e rankings semanais.

## Arquitetura e Componentes do Sistema

O sistema integra diversos componentes de forma orquestrada, conforme ilustrado no fluxo abaixo:

- **Orquestrador N8N:** Responsável por coordenar todos os fluxos – agendamento diário/semanal, envio de perguntas, processamento de respostas, atualização do banco e acionamento de APIs externas. O N8N é uma plataforma de automação de fluxos de trabalho altamente versátil, permitindo conectar "qualquer coisa a tudo" de forma visual e programável [reddit.com](https://www.reddit.com). Nele serão configurados os workflows que automatizam o quiz diário e o ranking semanal.
- **Agente Zaia (WhatsApp):** Chatbot configurado na plataforma Zaia para interagir com os usuários via WhatsApp. O Zaia facilita a conexão do agente de IA ao WhatsApp através de integrações nativas e API [zaia.app](https://zaia.app). Este agente envia as perguntas (“cards” de estudo) diariamente, recebe as respostas dos usuários e encaminha essas respostas para o N8N, além de devolver as correções e feedback automaticamente.

  > **Observação:** O Zaia suporta integrações via Make (Integromat) e APIs, o que utilizaremos para comunicá-lo com o N8N.

- **Canal LinkedIn (Opcional):** Caso seja possível utilizar a API do LinkedIn para mensagens, o sistema também enviaria perguntas via inbox do LinkedIn. Entretanto, é importante verificar a viabilidade: a API pública do LinkedIn não permite envio de mensagens privadas a usuários comuns, a menos que se trate de uma integração aprovada e restrita [community.make.com](https://community.make.com). Portanto, assumiremos que as interações com usuários acontecerão principalmente via WhatsApp, utilizando LinkedIn apenas para divulgação do ranking semanal público.
- **Banco de Dados PostgreSQL:** Armazena persistentemente os dados do sistema. Em containers Docker (local ou AWS) teremos o PostgreSQL com tabelas para Usuários, Perguntas, Respostas/Logs, Pontuações e Rankings. Sempre que necessário, o N8N consulta ou atualiza esse banco: por exemplo, para obter as próximas perguntas, gravar a resposta do usuário e calcular pontuações. Usar um banco relacional garante consistência na gamificação (evita perda de dados entre reinícios do fluxo) e facilita consultas agregadas (como calcular o Top 10 semanal).
- **LLM (OpenAI GPT-4 ou Google Gemini):** O modelo de linguagem entra em cena em dois momentos: (1) para geração dinâmica de perguntas a partir da base de conhecimento de DevOps, e (2) para correção inteligente de respostas abertas e geração de feedback. A base de dados interna servirá de referência (contexto) para o LLM, possivelmente via técnica de Retrieval-Augmented Generation (RAG) [reddit.com](https://www.reddit.com) – ou seja, o LLM acessa informações atualizadas do banco (como gabaritos, explicações ou documentação DevOps) para criar perguntas pertinentes e avaliar respostas com precisão.

  Isso garante que as perguntas sejam relevantes e atualizadas, e que a correção automática seja confiável.  
  **Exemplo:** Poderíamos fornecer ao LLM um trecho de documentação sobre Kubernetes do nosso banco e pedir que ele formule uma questão de nível “Expert” sobre aquele conteúdo, garantindo variedade de perguntas além do banco estático.

- **Integração LinkedIn (Postagens):** Além do agente conversacional, utilizaremos a API do LinkedIn (ou integração via N8N) para postar automaticamente o ranking semanal na página ou grupo do DOUG BR no LinkedIn. O N8N fará uma requisição à API do LinkedIn para criar uma postagem contendo os Top 10 usuários da semana e suas pontuações. Se possível, tentaremos mencionar cada usuário (usando handles/IDs do LinkedIn) na postagem – embora isso dependa das permissões da API. Em último caso, os nomes serão listados sem tag.

  > **Nota:** A postagem de conteúdo no LinkedIn via API é viável mediante autenticação OAuth e permissões de publishing [n8n.io](https://n8n.io), já a mensagem direta não é permitida conforme citado.

### Diagrama de Fluxo (Visão Geral)

Embora a interface aqui não permita exibir imagens diretamente, podemos descrever visualmente o fluxo principal:

1. **[Cron Diário – N8N] ➜ [Zaia API – WhatsApp] ➜ [Usuário]:**  
   O N8N, via um gatilho agendado diário, inicia o envio das perguntas. Ele comunica-se com a API do agente Zaia [zaia.app](https://zaia.app) para que o bot do WhatsApp envie a primeira pergunta do dia ao usuário.

2. **[Usuário ➜ Zaia – WhatsApp] ➜ [N8N]:**  
   O usuário recebe a pergunta no WhatsApp e responde. A resposta é capturada pelo agente Zaia e enviada para o N8N através de um webhook (o agente é configurado para fazer uma requisição HTTP ao N8N com a resposta do usuário e seu identificador).

3. **[N8N ➜ PostgreSQL & LLM]:**  
   O N8N recebe a resposta do usuário e então executa a lógica de correção. Primeiro, consulta no PostgreSQL o gabarito ou informações da pergunta respondida (por exemplo, a resposta correta esperada, ou dados técnicos relevantes). Se a resposta for objetiva (ex: múltipla escolha), o N8N simplesmente compara com o gabarito. Se for uma resposta em texto livre ou necessitar avaliação semântica, o N8N pode invocar o LLM (OpenAI/Gemini) para avaliar se a resposta do usuário está correta ou próxima do esperado.

   O LLM pode ser passado junto com a resposta correta como referência para julgar a equivalência. Além disso, o LLM pode gerar um feedback explicativo para o usuário, enriquecendo a experiência (por exemplo: “Correto! Você acertou porque de fato no Kubernetes o etcd armazena os dados de estado do cluster...” ou “Não foi dessa vez. A resposta esperada era X, pois...”). Essa combinação de banco + IA garante precisão e personalização na correção.

4. **[N8N ➜ Atualização do Banco]:**  
   Concomitantemente, o N8N atualiza as tabelas de resultados: marca aquela pergunta como respondida pelo usuário, registra se ele acertou, e incrementa sua pontuação. A pontuação pode considerar o nível da pergunta – por exemplo, 1 ponto para Fácil, 2 para Pleno, 3 para Expert, ou outra escala – incentivando o usuário a buscar desafios maiores.

5. **[N8N ➜ Zaia ➜ Usuário]:**  
   O N8N então retorna ao agente Zaia uma resposta formatada para o usuário contendo o feedback da questão atual e, se ainda houver perguntas pendentes, já a próxima pergunta. O agente Zaia envia essa mensagem ao usuário no WhatsApp.

   O ciclo então se repete: o usuário responde à próxima pergunta, o agente encaminha ao N8N, e assim por diante. Ao final da 3ª pergunta do dia, o N8N formula uma mensagem final de conclusão daquele dia para o usuário.

6. **[Resumo Diário ➜ Usuário]:**  
   Após a terceira pergunta, o usuário recebe do agente um resumo com sua performance diária: quantas respostas corretas de 3, quantos pontos ganhou no dia, e possivelmente sua pontuação acumulada na semana até aquele dia. Esse feedback final diário serve de estímulo e transparência, mostrando ao participante seu progresso no ranking.

7. **[Cron Semanal – N8N] ➜ [PostgreSQL]:**  
   Paralelamente ao fluxo diário, existe um fluxo semanal. Toda segunda-feira (em horário definido, ex: madrugada de domingo para segunda), um gatilho agendado no N8N irá calcular o ranking da semana anterior. O N8N faz uma consulta agregada no PostgreSQL para ordenar os usuários por pontuação acumulada nos últimos 7 dias (que no caso serão resetados semanalmente). Ele então extrai os Top 10 usuários.

8. **[N8N ➜ LinkedIn]:**  
   De posse do ranking, o N8N monta um conteúdo de postagem para o LinkedIn da comunidade DOUG BR. Essa postagem contém, por exemplo, um título comemorativo (“Ranking Semanal DevOps User Group – Parabéns aos Top 10!”) e uma lista enumerada do 1º ao 10º colocados com seus nomes e pontuações. Se for possível tecnicamente, o N8N inclui menções (@) aos perfis do LinkedIn desses usuários – desde que previamente tenhamos armazenado o ID ou URL do perfil de cada um. (Lembrando: mencionar perfis via API pode ser limitado; como fallback, podemos listar nomes e eventualmente um link para o perfil.)

9. **[Reset de Pontuações]:**  
   Ainda no fluxo semanal, depois de postar o ranking, o N8N deve reiniciar as pontuações dos usuários no banco, preparando o terreno para a próxima competição semanal. Isso pode ser feito zerando um campo de pontuação semanal na tabela de usuários ou movendo os dados para um histórico. Reiniciar semanalmente mantém a competição justa e motivadora – ninguém fica eternamente na frente, pois toda semana todos começam do zero, dando chance para novos participantes se destacarem [nudgenow.com](https://nudgenow.com). Essa estratégia de reset periódico é recomendada em gamificação para manter o engajamento alto e a competição “fresca” [nudgenow.com](https://nudgenow.com).

> **Importante:** Todo o projeto é desenhado para ser 100% automatizado. Uma vez configurados os fluxos no N8N e no agente Zaia, a ideia é que não haja intervenções manuais: diariamente as perguntas saem e são corrigidas automaticamente, e semanalmente o ranking é publicado automaticamente.

## Fluxo de Perguntas Diárias – Detalhamento

Esta seção descreve passo a passo o fluxo diário desde a seleção das perguntas até o feedback ao usuário, integrando N8N, Zaia/WhatsApp, Banco de Dados e LLM:

1. **Agendamento Diário:**  
   O trigger inicial parte de um nó Cron no N8N configurado para disparar todos os dias úteis (ou todos os dias da semana, conforme decidirmos) em um determinado horário – por exemplo, às 9h da manhã. Esse Cron aciona o workflow responsável pelas perguntas do dia.

2. **Seleção de Usuários e Perguntas:**  
   O primeiro passo do workflow diário é obter do banco de dados a lista de usuários ativos que devem receber o quiz. Cada usuário pode ter preferências registradas (por exemplo, talvez alguns prefiram receber via WhatsApp, outros via LinkedIn – mas dado o cenário de limitação da API do LinkedIn, provavelmente todos receberão pelo WhatsApp).

   Para cada usuário, o N8N pode selecionar ou gerar três perguntas:
   - **Se optarmos por perguntas pré-cadastradas no banco:** selecionamos 3 questões de categorias ou dificuldades variadas (podemos estabelecer um rodízio: ex. 1 fácil, 1 pleno, 1 expert por dia para cada usuário). Marcar também quais questões já foram enviadas a esse usuário em dias anteriores para evitar repetição frequente.
   - **Se optarmos por geração dinâmica via LLM:** podemos armazenar no banco apenas temas ou tópicos de conhecimento e pedir ao LLM para gerar questões inéditas baseadas nesses tópicos. Por exemplo, escolher 3 tópicos (um básico, um intermediário, um avançado) e usar a API do OpenAI/Gemini para criar perguntas nessas categorias.

   Nesse caso, o banco de dados fornece a referência (tópico e talvez um texto base ou link), e o LLM cria a pergunta e possivelmente o gabarito esperado. Essa abordagem aumenta a diversidade de perguntas, mas demanda cautela: precisamos validar que o gabarito fornecido pelo LLM está correto.

   Uma forma segura é: armazenar previamente no banco o gabarito ou explicação do tópico, e ao usar o LLM, incluí-lo no prompt para garantir que a resposta conhecida seja usada como base da pergunta. (Em suma, o LLM não “inventa” conhecimento novo, apenas formula a pergunta baseado no conteúdo fornecido.)

3. **Envio da Primeira Pergunta (WhatsApp via Zaia):**  
   Com a pergunta #1 definida para um determinado usuário, o N8N fará o envio via WhatsApp. Para isso, existem dois cenários:
   - **Usando API do Zaia:** O Zaia oferece uma API para enviar mensagens através do agente (similar a como faria com a API oficial do WhatsApp Business). O N8N executa uma requisição HTTP (ou usa um nó próprio, caso disponível) endereçada ao endpoint do agente Zaia contendo o identificador do usuário (número de telefone WhatsApp) e o conteúdo da pergunta. O agente Zaia então encaminha essa mensagem ao usuário no WhatsApp.
   - **Via Integração N8N->WhatsApp Cloud API:** Alternativamente, se o Zaia expuser diretamente a instância do WhatsApp Business API, o N8N poderia usar um nó dedicado (por ex. WhatsApp Business Cloud node) para enviar a mensagem. Contudo, como temos um agente configurado, o ideal é passar por ele para manter contexto da conversa.

   A mensagem enviada ao usuário contém a pergunta claramente formulada, possivelmente com opções se for múltipla escolha.  
   **Exemplo de mensagem:** “Pergunta 1 (Fácil): No Git, qual comando é usado para criar uma nova branch?”.

   O agente Zaia lida com o envio e garante a entrega via WhatsApp (lembrando que o número do WhatsApp do agente deve estar configurado e autorizado para envio de mensagens para os usuários opt-in).

4. **Interação do Usuário – Recebimento e Resposta:**  
   O usuário recebe a pergunta no aplicativo do WhatsApp e responde normalmente, escrevendo sua resposta ou escolhendo uma opção. Por exemplo, o usuário poderia responder “git checkout -b” para a pergunta acima.

5. **Captação da Resposta (Zaia) e Webhook para N8N:**  
   O agente Zaia, ao receber a mensagem de resposta do usuário, deve acionar um Webhook HTTP previamente configurado que aponta para o N8N. Na configuração do agente (ver seção de configuração do agente Zaia abaixo), haverá um passo do tipo “Enviar para webhook/API” que envia os dados relevantes da mensagem para o endpoint do N8N.

   **Dados enviados:** identificação do usuário (pode ser um ID interno ou o número de telefone), identificação da pergunta (podemos incluir um código da pergunta na mensagem original ou manter contexto no agente), e o texto da resposta dada.

   Esse webhook ativa no N8N o fluxo de processamento da resposta do usuário.

6. **Processamento da Resposta no N8N:**  
   O N8N recebe a chamada do webhook com os dados. Ele identifica a qual pergunta essa resposta corresponde (por ID ou contexto armazenado). Em seguida, executa a lógica de correção:
   - Busca no PostgreSQL o gabarito da pergunta. O gabarito pode ser uma resposta exata (para perguntas objetivas) ou um conjunto de palavras-chave/descrição esperada (para perguntas dissertativas).
   - Se for necessário, invoca o LLM para avaliar. Por exemplo, para respostas textuais: envia ao LLM um prompt contendo a pergunta original, a resposta do usuário e a resposta correta esperada, pedindo uma avaliação binária (“está certo ou não?”) ou uma correção percentual.
   
   O LLM também pode ser solicitado a gerar uma breve explicação de 1-2 frases sobre a resposta correta.

   O N8N extrai daí a flag de acerto e o feedback.

7. **Atualização de Pontuação e Log:**  
   O N8N então atualiza o banco: incrementa os pontos do usuário naquela semana (ex.: +1, +2 ou +3 dependendo da dificuldade da pergunta) e salva um registro da resposta (em uma tabela de Respostas ou Logs) com detalhes: user_id, question_id, resposta dada, correto/errado, timestamp. Esse log permite também auditar mais tarde e dar feedback acumulado.

8. **Geração de Feedback Imediato:**  
   Em posse do resultado, o N8N prepara uma mensagem de retorno para o usuário contendo:
   - Uma indicação de acerto ou erro. (Ex.: ✅ "Correto!" ou ❌ "Resposta incorreta." – usar emojis torna amigável).
   - A resposta certa ou explicação. (Ex.: “O comando correto é git checkout -b, que cria e já muda para a nova branch.”).
   - Pontuação parcial: Opcionalmente, informar “+X pontos” ganhos ou simplesmente o número de acertos até agora no dia.
   - Próxima pergunta: Se ainda não foi a 3ª, já incluir a próxima pergunta na sequência, para manter o usuário engajado em continuar respondendo.
   
   Assim evitamos múltiplas mensagens fragmentadas – o usuário receberia em uma única mensagem tanto o feedback quanto a próxima questão. (No agente Zaia, também poderíamos separar essas etapas em duas mensagens distintas – mas integrar em uma só pode ser mais dinâmico.)

9. **Envio de Feedback e Próxima Pergunta:**  
   O N8N então envia essa resposta formatada de volta ao usuário via o agente Zaia (novamente utilizando a API ou retorno do webhook):  
   O agente Zaia recebe a resposta do N8N e a posta no WhatsApp do usuário.

   **Exemplo:**  
   "✅ Correto! O comando git checkout -b cria uma nova branch e muda para ela. Você ganhou 1 ponto.

   Pergunta 2 (Pleno): ..." 

   O usuário visualiza o feedback e já a segunda pergunta em sequência.

10. **Iteração das Questões 2 e 3:**  
    O fluxo repete os passos 4 a 9 para a segunda e terceira perguntas do dia: Usuário responde ➜ Zaia encaminha ➜ N8N corrige (consulta DB/LLM) ➜ N8N retorna feedback/próxima ➜ Zaia envia ao usuário. Em cada iteração, os pontos vão sendo acumulados e o feedback imediato fornecido.

    O agente Zaia precisa conseguir distinguir em qual pergunta o usuário está. Podemos gerenciar isso via contexto no N8N: cada webhook de resposta carrega um identificador da pergunta atual ou uma etapa. Como alternativa, o agente Zaia pode manter estado do diálogo para saber “próxima etapa”, mas é mais simples deixar o N8N gerenciar isso através, por exemplo, de um ID de sessão ou marcadores nas mensagens.

11. **Conclusão do Quiz Diário:**  
    Após o usuário responder à terceira pergunta, o N8N computa o resultado final do dia para aquele usuário: Conta quantas de 3 ele acertou e os pontos obtidos no dia. Atualiza novamente a base (pode armazenar um registro de conclusão de quiz diário, se desejado).

    Prepara uma mensagem de fechamento, parabenizando pela participação.  
    **Exemplo de mensagem final:**  
    "Fim do quiz de hoje! Você acertou 2 de 3 perguntas e fez 4 pontos hoje. Sua pontuação total na semana é 10 pontos. Continue assim!

    *(Se quiser, nos vemos amanhã para mais perguntas!)*"

    Envia essa mensagem final via Zaia para o usuário. O agente Zaia pode então opcionalmente encerrar ou aguardar até o próximo dia.

12. **Engajamento Adicional:**  
    Podemos considerar funcionalidades extras para aumentar o engajamento:
    - Se o usuário ficar muito tempo sem responder, o agente poderia mandar uma mensagem de lembrete ou dica. (Ex.: “Está aí? Não se preocupe, responda quando puder. Se precisar de ajuda, tente lembrar que... [dica]”).
    - Se o usuário responde algo como “pula” ou “não sei”, podemos permitir pular a pergunta (talvez sem pontuar) e já mostrar a resposta correta para aprendizado.
    - Registrar a participação diária para eventualmente premiar assiduidade (badges semanais de “100% presença” etc., embora isso não esteja nas especificações, é uma ideia de gamificação adicional).

13. **Suporte a LinkedIn (se aplicável):**  
    Caso fosse possível enviar via LinkedIn, o fluxo seria semelhante, mas com diferenças técnicas:  
    Ao invés do agente Zaia, usaríamos a API do LinkedIn (ou um agente próprio) para enviar as perguntas via mensagem direta. Cada usuário teria fornecido seu LinkedIn ID.  
    Contudo, reforçando, a API do LinkedIn não disponibiliza envio de mensagens diretas na API aberta [community.make.com](https://community.make.com), então esta via provavelmente será descartada.  
    Em lugar disso, poderemos no máximo enviar as perguntas via mensagem do LinkedIn Pages se existisse essa permissão (por exemplo, páginas empresariais podem mandar mensagem para quem já interagiu via chat da página, conforme APIs de “LinkedIn Pages Messaging”). Ainda assim, isso exigiria que o usuário tivesse iniciado uma conversa com a página DOUG BR no LinkedIn.  
    Devido a essas restrições, o plano principal assume o WhatsApp como canal único de entrega das perguntas. O LinkedIn, então, ficará restrito à divulgação do ranking e talvez conteúdos semanais, não para o quiz diário individual.

Em resumo, o fluxo diário combina automação via N8N, interatividade via WhatsApp/Zaia e inteligência via LLM para oferecer aos usuários uma experiência de micro-aprendizado diária, com correção instantânea e feedback personalizado.

## Banco de Dados: Modelo de Dados e Estrutura da Base de Perguntas

Um esquema sugerido em PostgreSQL para suportar o EstudeiCards poderia incluir as seguintes tabelas principais:

- **Usuários:**  
  Armazena os participantes registrados.  
  **Campos:** user_id (chave primária), nome, telefone_whatsapp (ou ID do contato no Zaia), linkedin_profile (URL ou ID do perfil LinkedIn, se fornecido, para menções no ranking), pontuacao_semana (pontuação acumulada na semana atual), pontuacao_total (opcional, se quiser histórico total), data_cadastro, etc.  
  Poderíamos guardar também a preferência de idioma ou canal aqui (caso no futuro houvesse envio por e-mail, Telegram, etc).

- **Perguntas:**  
  Contém o banco de perguntas de referência.  
  **Campos:** pergunta_id, nivel (Fácil/Pleno/Expert), enunciado (texto da pergunta; pode incluir placeholders se gerada dinamicamente), opcoes (se for múltipla escolha, armazenar talvez como JSON ou em tabela relacionada de opções), resposta_correta (pode ser a letra da opção correta ou o texto esperado), explicacao (um texto explicando a resposta, para feedback; esse campo pode ser usado pelo LLM ou enviado diretamente ao usuário), tema (categoria/assunto DevOps, e.g. “Docker”, “Kubernetes”, “CI/CD”), fonte_referencia (um link ou referência na base de conhecimento, útil para montar prompts do LLM ou para enviar ao usuário que quiser ler mais).

  Essa tabela serve de repositório de conhecimento. Quando quisermos que o LLM gere perguntas, podemos armazenar aqui conteúdos que ele usará. Por exemplo, ter um registro com nível Expert, tema “Kubernetes etcd”, sem enunciado fixo mas com uma explicação do conceito, permitindo que a pergunta seja criada dinamicamente a partir disso. Alternativamente, podemos ter uma tabela separada Temas/Curadoria, e uma tabela PerguntasGeradas que é preenchida on-the-fly pelo LLM e armazenada se quiser manter histórico das perguntas feitas.

- **RespostasUsuarios:**  
  Registro de respostas enviadas e resultados, para fins de auditoria e possivelmente refinar o modelo.  
  **Campos:** resposta_id, user_id (FK para Usuários), pergunta_id (FK para Perguntas, ou um identificador da pergunta do dia), resposta_texto (o conteúdo que o usuário enviou), correto (booleano), pontos_obtidos (inteiro), data_hora_resposta.  
  Essa tabela vai armazenando cada interação. Poderíamos também armazenar um campo sessao_id ou data_quiz, para agrupar as 3 perguntas de um mesmo dia.

- **RankingSemanal:**  
  (Opcional, pois podemos calcular on the fly)  
  Podemos não ter uma tabela fixa de ranking, em vez disso calculamos via consulta na tabela de usuários (ordenando por pontuacao_semana). Porém, podemos querer guardar historicamente os top 10 de cada semana para análise futura.

  Nesse caso, uma tabela ranking_semanal com campos: semana_id, inicio_semana (data), fim_semana, user_id, pontuacao naquela semana, posicao. Toda segunda-feira salvaríamos os top N ali.

- **Configurações/Parâmetros:**  
  Uma tabela simples para armazenar parâmetros globais, como horário de envio, texto customizado de mensagens, etc., que permita mudar sem alterar o fluxo.  
  **Exemplo:** horario_envio_quiz, mensagem_boas_vindas, flags para “LinkedIn_on/off”.

  Embora não estritamente necessário, dá flexibilidade sem mexer em código.

### Relações e Considerações

- **Usuários -> RespostasUsuarios** é 1:N (um usuário tem muitas respostas registradas).
- **Perguntas -> RespostasUsuarios** também é 1:N (cada pergunta pode ter sido respondida por vários usuários ao longo do tempo).

As perguntas podem ter uma relação de muitos para muitos com usuários se quisermos impedir repetição: ex., uma tabela PerguntasEnviadas mapeando user_id e pergunta_id para marcar que aquela já foi enviada àquele user. Assim evitamos repetir questão para a mesma pessoa. Ou podemos simplesmente marcar na RespostasUsuarios e checar lá.

**Performance:** O número de usuários do quiz pode crescer, então projetamos consultas simples. A seleção de top 10 por semana em PostgreSQL é trivial com índice em pontuacao_semana. Zerar pontuações semanalmente é um update em massa (cuidado com lock se forem muitos usuários, mas provavelmente são dezenas ou centenas, não milhões – perfeitamente viável). Podemos usar views ou funções no...
