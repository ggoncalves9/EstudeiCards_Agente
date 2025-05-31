Plano Detalhado do Projeto EstudeiCards Agentes
Visão Geral do Projeto
O EstudeiCards Agentes é um sistema de gamificação de conhecimento DevOps que envia aos usuários cadastrados três perguntas diárias sobre temas do ecossistema DevOps. As perguntas têm três níveis de dificuldade – Fácil, Pleno e Expert – estimulando aprendizado progressivo. As questões podem ser enviadas via WhatsApp (utilizando a plataforma Zaia) e, se viável, via mensagem direta no LinkedIn. O diferencial é que as perguntas e correções serão geradas dinamicamente com apoio de modelos de LLM (como OpenAI GPT-4 ou Google Gemini), usando uma base de dados de referência para garantir acurácia. O objetivo do projeto é engajar gratuitamente a comunidade DevOps User Group Brazil (DOUG BR) em uma competição saudável, 100% automatizada, que incentiva estudo contínuo através de quizzes diários e rankings semanais.
Arquitetura e Componentes do Sistema
O sistema integra diversos componentes de forma orquestrada, conforme ilustrado no fluxo abaixo:
Orquestrador N8N: Responsável por coordenar todos os fluxos – agendamento diário/semanal, envio de perguntas, processamento de respostas, atualização do banco e acionamento de APIs externas. O N8N é uma plataforma de automação de fluxos de trabalho altamente versátil, permitindo conectar "qualquer coisa a tudo" de forma visual e programável
reddit.com
. Nele serão configurados os workflows que automatizam o quiz diário e o ranking semanal.
Agente Zaia (WhatsApp): Chatbot configurado na plataforma Zaia para interagir com os usuários via WhatsApp. O Zaia facilita a conexão do agente de IA ao WhatsApp através de integrações nativas e API
zaia.app
. Este agente envia as perguntas (“cards” de estudo) diariamente, recebe as respostas dos usuários e encaminha essas respostas para o N8N, além de devolver as correções e feedback automaticamente. Observação: O Zaia suporta integrações via Make (Integromat) e APIs, o que utilizaremos para comunicá-lo com o N8N.
Canal LinkedIn (Opcional): Caso seja possível utilizar a API do LinkedIn para mensagens, o sistema também enviaria perguntas via inbox do LinkedIn. Entretanto, é importante verificar a viabilidade: a API pública do LinkedIn não permite envio de mensagens privadas a usuários comuns, a menos que se trate de uma integração aprovada e restrita
community.make.com
. Portanto, assumiremos que as interações com usuários acontecerão principalmente via WhatsApp, utilizando LinkedIn apenas para divulgação do ranking semanal público.
Banco de Dados PostgreSQL: Armazena persistentemente os dados do sistema. Em containers Docker (local ou AWS) teremos o PostgreSQL com tabelas para Usuários, Perguntas, Respostas/Logs, Pontuações e Rankings. Sempre que necessário, o N8N consulta ou atualiza esse banco: por exemplo, para obter as próximas perguntas, gravar a resposta do usuário e calcular pontuações. Usar um banco relacional garante consistência na gamificação (evita perda de dados entre reinícios do fluxo) e facilita consultas agregadas (como calcular o Top 10 semanal).
LLM (OpenAI GPT-4 ou Google Gemini): O modelo de linguagem entra em cena em dois momentos: (1) para geração dinâmica de perguntas a partir da base de conhecimento de DevOps, e (2) para correção inteligente de respostas abertas e geração de feedback. A base de dados interna servirá de referência (contexto) para o LLM, possivelmente via técnica de Retrieval-Augmented Generation (RAG)
reddit.com
 – ou seja, o LLM acessa informações atualizadas do banco (como gabaritos, explicações ou documentação DevOps) para criar perguntas pertinentes e avaliar respostas com precisão. Isso garante que as perguntas sejam relevantes e atualizadas, e que a correção automática seja confiável. Exemplo: Poderíamos fornecer ao LLM um trecho de documentação sobre Kubernetes do nosso banco e pedir que ele formule uma questão de nível “Expert” sobre aquele conteúdo, garantindo variedade de perguntas além do banco estático.
Integração LinkedIn (Postagens): Além do agente conversacional, utilizaremos a API do LinkedIn (ou integração via N8N) para postar automaticamente o ranking semanal na página ou grupo do DOUG BR no LinkedIn. O N8N fará uma requisição à API do LinkedIn para criar uma postagem contendo os Top 10 usuários da semana e suas pontuações. Se possível, tentaremos mencionar cada usuário (usando handles/IDs do LinkedIn) na postagem – embora isso dependa das permissões da API. Em último caso, os nomes serão listados sem tag. (Nota: A postagem de conteúdo no LinkedIn via API é viável mediante autenticação OAuth e permissões de publishing
n8n.io
, já a mensagem direta não é permitida conforme citado).
Diagrama de Fluxo (Visão Geral)
Embora a interface aqui não permita exibir imagens diretamente, podemos descrever visualmente o fluxo principal:
[Cron Diário – N8N] ➜ [Zaia API – WhatsApp] ➜ [Usuário]: O N8N, via um gatilho agendado diário, inicia o envio das perguntas. Ele comunica-se com a API do agente Zaia
zaia.app
 para que o bot do WhatsApp envie a primeira pergunta do dia ao usuário.
[Usuário ➜ Zaia – WhatsApp] ➜ [N8N]: O usuário recebe a pergunta no WhatsApp e responde. A resposta é capturada pelo agente Zaia e enviada para o N8N através de um webhook (o agente é configurado para fazer uma requisição HTTP ao N8N com a resposta do usuário e seu identificador).
[N8N ➜ PostgreSQL & LLM]: O N8N recebe a resposta do usuário e então executa a lógica de correção. Primeiro, consulta no PostgreSQL o gabarito ou informações da pergunta respondida (por exemplo, a resposta correta esperada, ou dados técnicos relevantes). Se a resposta for objetiva (ex: múltipla escolha), o N8N simplesmente compara com o gabarito. Se for uma resposta em texto livre ou necessitar avaliação semântica, o N8N pode invocar o LLM (OpenAI/Gemini) para avaliar se a resposta do usuário está correta ou próxima do esperado. O LLM pode ser passado junto com a resposta correta como referência para julgar a equivalência. Além disso, o LLM pode gerar um feedback explicativo para o usuário, enriquecendo a experiência (por exemplo: “Correto! Você acertou porque de fato no Kubernetes o etcd armazena os dados de estado do cluster...” ou “Não foi dessa vez. A resposta esperada era X, pois...”). Essa combinação de banco + IA garante precisão e personalização na correção.
[N8N ➜ Atualização do Banco]: Concomitantemente, o N8N atualiza as tabelas de resultados: marca aquela pergunta como respondida pelo usuário, registra se ele acertou, e incrementa sua pontuação. A pontuação pode considerar o nível da pergunta – por exemplo, 1 ponto para Fácil, 2 para Pleno, 3 para Expert, ou outra escala – incentivando o usuário a buscar desafios maiores. (A definição exata da pontuação por nível pode ser ajustada conforme feedback da comunidade.)
[N8N ➜ Zaia ➜ Usuário]: O N8N então retorna ao agente Zaia uma resposta formatada para o usuário contendo o feedback da questão atual e, se ainda houver perguntas pendentes, já a próxima pergunta. O agente Zaia envia essa mensagem ao usuário no WhatsApp. O ciclo então se repete: o usuário responde à próxima pergunta, o agente encaminha ao N8N, e assim por diante. Ao final da 3ª pergunta do dia, o N8N formula uma mensagem final de conclusão daquele dia para o usuário.
[Resumo Diário ➜ Usuário]: Após a terceira pergunta, o usuário recebe do agente um resumo com sua performance diária: quantas respostas corretas de 3, quantos pontos ganhou no dia, e possivelmente sua pontuação acumulada na semana até aquele dia. Esse feedback final diário serve de estímulo e transparência, mostrando ao participante seu progresso no ranking.
[Cron Semanal – N8N] ➜ [PostgreSQL]: Paralelamente ao fluxo diário, existe um fluxo semanal. Toda segunda-feira (em horário definido, ex: madrugada de domingo para segunda), um gatilho agendado no N8N irá calcular o ranking da semana anterior. O N8N faz uma consulta agregada no PostgreSQL para ordenar os usuários por pontuação acumulada nos últimos 7 dias (que no caso serão resetados semanalmente). Ele então extrai os Top 10 usuários.
[N8N ➜ LinkedIn]: De posse do ranking, o N8N monta um conteúdo de postagem para o LinkedIn da comunidade DOUG BR. Essa postagem contém, por exemplo, um título comemorativo (“🏆 Ranking Semanal DevOps User Group – Parabéns aos Top 10!”) e uma lista enumerada do 1º ao 10º colocados com seus nomes e pontuações. Se for possível tecnicamente, o N8N inclui menções (@) aos perfis do LinkedIn desses usuários – desde que previamente tenhamos armazenado o ID ou URL do perfil de cada um. (Lembrando: mencionar perfis via API pode ser limitado; como fallback, podemos listar nomes e eventualmente um link para o perfil.) Em seguida, o N8N realiza a chamada à API do LinkedIn para criar a postagem na página do grupo ou perfil autorizado
n8n.io
.
[Reset de Pontuações]: Ainda no fluxo semanal, depois de postar o ranking, o N8N deve reiniciar as pontuações dos usuários no banco, preparando o terreno para a próxima competição semanal. Isso pode ser feito zerando um campo de pontuação semanal na tabela de usuários ou movendo os dados para um histórico. Reiniciar semanalmente mantém a competição justa e motivadora – ninguém fica eternamente na frente, pois toda semana todos começam do zero, dando chance para novos participantes se destacarem
nudgenow.com
. Essa estratégia de reset periódico é recomendada em gamificação para manter o engajamento alto e a competição “fresca”
nudgenow.com
.
Importante: Todo o projeto é desenhado para ser 100% automatizado. Uma vez configurados os fluxos no N8N e no agente Zaia, a ideia é que não haja intervenções manuais: diariamente as perguntas saem e são corrigidas automaticamente, e semanalmente o ranking é publicado automaticamente. A moderação humana pode se restringir a revisar eventuais falhas ou ajustar/expandir a base de perguntas conforme necessário.
Fluxo de Perguntas Diárias – Detalhamento
Esta seção descreve passo a passo o fluxo diário desde a seleção das perguntas até o feedback ao usuário, integrando N8N, Zaia/WhatsApp, Banco de Dados e LLM:
Agendamento Diário: O trigger inicial parte de um nó Cron no N8N configurado para disparar todos os dias úteis (ou todos os dias da semana, conforme decidirmos) em um determinado horário – por exemplo, às 9h da manhã. Esse Cron aciona o workflow responsável pelas perguntas do dia.
Seleção de Usuários e Perguntas: O primeiro passo do workflow diário é obter do banco de dados a lista de usuários ativos que devem receber o quiz. Cada usuário pode ter preferências registradas (por exemplo, talvez alguns prefiram receber via WhatsApp, outros via LinkedIn – mas dado o cenário de limitação da API do LinkedIn, provavelmente todos receberão pelo WhatsApp). Para cada usuário, o N8N pode selecionar ou gerar três perguntas:
Se optarmos por perguntas pré-cadastradas no banco: selecionamos 3 questões de categorias ou dificuldades variadas (podemos estabelecer um rodízio: ex. 1 fácil, 1 pleno, 1 expert por dia para cada usuário). Marcar também quais questões já foram enviadas a esse usuário em dias anteriores para evitar repetição frequente.
Se optarmos por geração dinâmica via LLM: podemos armazenar no banco apenas temas ou tópicos de conhecimento e pedir ao LLM para gerar questões inéditas baseadas nesses tópicos. Por exemplo, escolher 3 tópicos (um básico, um intermediário, um avançado) e usar a API do OpenAI/Gemini para criar perguntas nessas categorias. Nesse caso, o banco de dados fornece a referência (tópico e talvez um texto base ou link), e o LLM cria a pergunta e possivelmente o gabarito esperado. Essa abordagem aumenta a diversidade de perguntas, mas demanda cautela: precisamos validar que o gabarito fornecido pelo LLM está correto. Uma forma segura é: armazenar previamente no banco o gabarito ou explicação do tópico, e ao usar o LLM, incluí-lo no prompt para garantir que a resposta conhecida seja usada como base da pergunta. (Em suma, o LLM não “inventa” conhecimento novo, apenas formula a pergunta baseado no conteúdo fornecido.)
Envio da Primeira Pergunta (WhatsApp via Zaia): Com a pergunta #1 definida para um determinado usuário, o N8N fará o envio via WhatsApp. Para isso, existem dois cenários:
Usando API do Zaia: O Zaia oferece uma API para enviar mensagens através do agente (similar a como faria com a API oficial do WhatsApp Business). O N8N executa uma requisição HTTP (ou usa um nó próprio, caso disponível) endereçada ao endpoint do agente Zaia contendo o identificador do usuário (número de telefone WhatsApp) e o conteúdo da pergunta. O agente Zaia então encaminha essa mensagem ao usuário no WhatsApp.
zaia.app
Via Integração N8N->WhatsApp Cloud API: Alternativamente, se o Zaia expuser diretamente a instância do WhatsApp Business API, o N8N poderia usar um nó dedicado (por ex. WhatsApp Business Cloud node) para enviar a mensagem. Contudo, como temos um agente configurado, o ideal é passar por ele para manter contexto da conversa.
A mensagem enviada ao usuário contém a pergunta claramente formulada, possivelmente com opções se for múltipla escolha. Exemplo de mensagem: “Pergunta 1 (Fácil): No Git, qual comando é usado para criar uma nova branch?”.
O agente Zaia lida com o envio e garante a entrega via WhatsApp (lembrando que o número do WhatsApp do agente deve estar configurado e autorizado para envio de mensagens para os usuários opt-in).
Interação do Usuário – Recebimento e Resposta: O usuário recebe a pergunta no aplicativo do WhatsApp e responde normalmente, escrevendo sua resposta ou escolhendo uma opção. Por exemplo, o usuário poderia responder “git checkout -b” para a pergunta acima.
Captação da Resposta (Zaia) e Webhook para N8N: O agente Zaia, ao receber a mensagem de resposta do usuário, deve acionAR um Webhook HTTP previamente configurado que aponta para o N8N. Na configuração do agente (ver seção de configuração do agente Zaia abaixo), haverá um passo do tipo “Enviar para webhook/API” que envia os dados relevantes da mensagem para o endpoint do N8N:
Dados enviados: identificação do usuário (pode ser um ID interno ou o número de telefone), identificação da pergunta (podemos incluir um código da pergunta na mensagem original ou manter contexto no agente), e o texto da resposta dada.
Esse webhook ativa no N8N o fluxo de processamento da resposta do usuário.
Processamento da Resposta no N8N: O N8N recebe a chamada do webhook com os dados. Ele identifica a qual pergunta essa resposta corresponde (por ID ou contexto armazenado). Em seguida, executa a lógica de correção:
Busca no PostgreSQL o gabarito da pergunta. O gabarito pode ser uma resposta exata (para perguntas objetivas) ou um conjunto de palavras-chave/descrição esperada (para perguntas dissertativas).
Se for necessário, invoca o LLM para avaliar. Por exemplo, para respostas textuais: envia ao LLM um prompt contendo a pergunta original, a resposta do usuário e a resposta correta esperada, pedindo uma avaliação binária (“está certo ou não?”) ou uma correção percentual. O LLM também pode ser solicitado a gerar uma breve explicação de 1-2 frases sobre a resposta correta.
Determinação de acerto: Com o resultado da avaliação, o N8N marca se o usuário acertou ou errou. Caso usemos apenas comparação simples (exato/igual), essa etapa é direta. Se usamos LLM, interpretamos a saída dele. Ex.: O LLM pode responder algo como “Correto. A resposta do usuário corresponde exatamente ao comando git para criar branch.” ou “Incorreto – o usuário mencionou X, mas o comando correto é Y.”. O N8N extrai daí a flag de acerto e o feedback.
Atualização de Pontuação e Log: O N8N então atualiza o banco: incrementa os pontos do usuário naquela semana (ex.: +1, +2 ou +3 dependendo da dificuldade da pergunta) e salva um registro da resposta (em uma tabela de Respostas ou Logs) com detalhes: user_id, question_id, resposta dada, correto/errado, timestamp. Esse log permite também auditar mais tarde e dar feedback acumulado.
Geração de Feedback Imediato: Em posse do resultado, o N8N prepara uma mensagem de retorno para o usuário contendo:
Uma indicação de acerto ou erro. (Ex.: ✅ "Correto!" ou ❌ "Resposta incorreta." – usar emojis torna amigável).
A resposta certa ou explicação. (Ex.: “O comando correto é git checkout -b, que cria e já muda para a nova branch.”).
Pontuação parcial: Opcionalmente, informar “+X pontos” ganhos ou simplesmente o número de acertos até agora no dia.
Próxima pergunta: Se ainda não foi a 3ª, já incluir a próxima pergunta na sequência, para manter o usuário engajado em continuar respondendo. Assim evitamos múltiplas mensagens fragmentadas – o usuário receberia em uma única mensagem tanto o feedback quanto a próxima questão. (No agente Zaia, também poderíamos separar essas etapas em duas mensagens distintas – mas integrar em uma só pode ser mais dinâmico.)
Envio de Feedback e Próxima Pergunta: O N8N então envia essa resposta formatada de volta ao usuário via o agente Zaia (novamente utilizando a API ou retorno do webhook):
O agente Zaia recebe a resposta do N8N e a posta no WhatsApp do usuário. Por exemplo: "✅ Correto! O comando git checkout -b cria uma nova branch e muda para ela. Você ganhou 1 ponto.\n\nPergunta 2 (Pleno): ...".
O usuário visualiza o feedback e já a segunda pergunta em sequência.
Iteração das Questões 2 e 3: O fluxo repete os passos 4 a 9 para a segunda e terceira perguntas do dia:
Usuário responde ➜ Zaia encaminha ➜ N8N corrige (consulta DB/LLM) ➜ N8N retorna feedback/próxima ➜ Zaia envia ao usuário.
Em cada iteração, os pontos vão sendo acumulados e o feedback imediato fornecido.
O agente Zaia precisa conseguir distinguir em qual pergunta o usuário está. Podemos gerenciar isso via contexto no N8N: cada webhook de resposta carrega um identificador da pergunta atual ou uma etapa. Como alternativa, o agente Zaia pode manter estado do diálogo para saber “próxima etapa”, mas é mais simples deixar o N8N gerenciar isso através, por exemplo, de um ID de sessão ou marcadores nas mensagens.
Conclusão do Quiz Diário: Após o usuário responder à terceira pergunta, o N8N computa o resultado final do dia para aquele usuário:
Conta quantas de 3 ele acertou e os pontos obtidos no dia.
Atualiza novamente a base (pode armazenar um registro de conclusão de quiz diário, se desejado).
Prepara uma mensagem de fechamento, parabenizando pela participação. Exemplo de mensagem final: "Fim do quiz de hoje! Você acertou 2 de 3 perguntas e fez 4 pontos hoje. Sua pontuação total na semana é 10 pontos. Continue assim! 💪\n\n*(Se quiser, nos vemos amanhã para mais perguntas!)*".
Envia essa mensagem final via Zaia para o usuário.
O agente Zaia pode então opcionalmente encerrar ou aguardar até o próximo dia.
Engajamento adicional: Podemos considerar funcionalidades extras para aumentar o engajamento:
Se o usuário ficar muito tempo sem responder, o agente poderia mandar uma mensagem de lembrete ou dica. (Ex.: “Está aí? 🙂 Não se preocupe, responda quando puder. Se precisar de ajuda, tente lembrar que... [dica]”).
Se o usuário responde algo como “pula” ou “não sei”, podemos permitir pular a pergunta (talvez sem pontuar) e já mostrar a resposta correta para aprendizado.
Registrar a participação diária para eventualmente premiar assiduidade (badges semanais de “100% presença” etc., embora isso não esteja nas especificações, é uma ideia de gamificação adicional).
Suporte a LinkedIn (se aplicável): Caso fosse possível enviar via LinkedIn, o fluxo seria semelhante, mas com diferenças técnicas:
Ao invés do agente Zaia, usaríamos a API do LinkedIn (ou um agente próprio) para enviar as perguntas via mensagem direta. Cada usuário teria fornecido seu LinkedIn ID. Contudo, reforçando, a API do LinkedIn não disponibiliza envio de mensagens diretas na API aberta
community.make.com
, então esta via provavelmente será descartada.
Em lugar disso, poderemos no máximo enviar as perguntas via mensagem do LinkedIn Pages se existisse essa permissão (por exemplo, páginas empresariais podem mandar mensagem para quem já interagiu via chat da página, conforme APIs de “LinkedIn Pages Messaging”). Ainda assim, isso exigiria que o usuário tivesse iniciado uma conversa com a página DOUG BR no LinkedIn.
Devido a essas restrições, o plano principal assume o WhatsApp como canal único de entrega das perguntas. O LinkedIn, então, ficará restrito à divulgação do ranking e talvez conteúdos semanais, não para o quiz diário individual.
Em resumo, o fluxo diário combina automação via N8N, interatividade via WhatsApp/Zaia e inteligência via LLM para oferecer aos usuários uma experiência de micro-aprendizado diária, com correção instantânea e feedback personalizado.
Banco de Dados: Modelo de Dados e Estrutura da Base de Perguntas
Um esquema sugerido em PostgreSQL para suportar o EstudeiCards poderia incluir as seguintes tabelas principais:
Usuários: Armazena os participantes registrados.
Campos: user_id (chave primária), nome, telefone_whatsapp (ou ID do contato no Zaia), linkedin_profile (URL ou ID do perfil LinkedIn, se fornecido, para menções no ranking), pontuacao_semana (pontuação acumulada na semana atual), pontuacao_total (opcional, se quiser histórico total), data_cadastro, etc.
Poderíamos guardar também a preferência de idioma ou canal aqui (caso no futuro houvesse envio por e-mail, Telegram, etc).
Perguntas: Contém o banco de perguntas de referência.
Campos: pergunta_id, nivel (Fácil/Pleno/Expert), enunciado (texto da pergunta; pode incluir placeholders se gerada dinamicamente), opcoes (se for múltipla escolha, armazenar talvez como JSON ou em tabela relacionada de opções), resposta_correta (pode ser a letra da opção correta ou o texto esperado), explicacao (um texto explicando a resposta, para feedback; esse campo pode ser usado pelo LLM ou enviado diretamente ao usuário), tema (categoria/assunto DevOps, e.g. “Docker”, “Kubernetes”, “CI/CD”), fonte_referencia (um link ou referência na base de conhecimento, útil para montar prompts do LLM ou para enviar ao usuário que quiser ler mais).
Essa tabela serve de repositório de conhecimento. Quando quisermos que o LLM gere perguntas, podemos armazenar aqui conteúdos que ele usará. Por exemplo, ter um registro com nível Expert, tema “Kubernetes etcd”, sem enunciado fixo mas com uma explicação do conceito, permitindo que a pergunta seja criada dinamicamente a partir disso.
Alternativamente, podemos ter uma tabela separada Temas/Curadoria, e uma tabela PerguntasGeradas que é preenchida on-the-fly pelo LLM e armazenada se quiser manter histórico das perguntas feitas.
RespostasUsuarios: Registro de respostas enviadas e resultados, para fins de auditoria e possivelmente refinar o modelo.
Campos: resposta_id, user_id (FK para Usuários), pergunta_id (FK para Perguntas, ou um identificador da pergunta do dia), resposta_texto (o conteúdo que o usuário enviou), correto (booleano), pontos_obtidos (inteiro), data_hora_resposta.
Essa tabela vai armazenando cada interação. Poderíamos também armazenar um campo sessao_id ou data_quiz, para agrupar as 3 perguntas de um mesmo dia.
RankingSemanal: (Opcional, pois podemos calcular on the fly)
Podemos não ter uma tabela fixa de ranking, em vez disso calculamos via consulta na tabela de usuários (ordenando por pontuacao_semana). Porém, podemos querer guardar historicamente os top 10 de cada semana para análise futura. Nesse caso, uma tabela ranking_semanal com campos: semana_id, inicio_semana (data), fim_semana, user_id, pontuacao naquela semana, posicao.
Toda segunda-feira salvaríamos os top N ali. Isso é opcional se quisermos algum histórico de evolução ou caso queiramos premiar o “campeão do mês” no futuro, por exemplo.
Configurações/Parâmetros: Uma tabela simples para armazenar parâmetros globais, como horário de envio, texto customizado de mensagens, etc., que permita mudar sem alterar o fluxo. Ex.: horario_envio_quiz, mensagem_boas_vindas, flags para “LinkedIn_on/off”. Embora não estritamente necessário, dá flexibilidade sem mexer em código.
Relações e Considerações:
Usuários -> RespostasUsuarios é 1:N (um usuário tem muitas respostas registradas).
Perguntas -> RespostasUsuarios também é 1:N (cada pergunta pode ter sido respondida por vários usuários ao longo do tempo).
As perguntas podem ter uma relação de muitos para muitos com usuários se quisermos impedir repetição: ex., uma tabela PerguntasEnviadas mapeando user_id e pergunta_id para marcar que aquela já foi enviada àquele user. Assim evitamos repetir questão para a mesma pessoa. Ou podemos simplesmente marcar na RespostasUsuarios e checar lá.
Performance: O número de usuários do quiz pode crescer, então projetamos consultas simples. A seleção de top 10 por semana em PostgreSQL é trivial com índice em pontuacao_semana. Zerar pontuações semanalmente é um update em massa (cuidado com lock se forem muitos usuários, mas provavelmente são dezenas ou centenas, não milhões – perfeitamente viável).
Podemos usar views ou funções no Postgres para facilitar, ex: uma view que já traga o ranking atual.
Exemplo de funcionamento dos dados: Digamos que o usuário Alice (ID 1) responde corretamente 2 das 3 perguntas hoje (segunda-feira). No banco: tabela RespostasUsuarios terá 3 linhas novas para Alice (com campo correto true/false conforme), e na tabela Usuários a Alice ficará com pontuacao_semana = 2 (se cada acerto =1 ponto simples) ou outra pontuação se ponderado. No dia seguinte, mais 3 respostas, acumulando. Na sexta-feira supomos Alice tenha 10 pontos total. Outro usuário Bob fez 12 pontos. Na segunda, o N8N calcula ranking: Bob em 1º, Alice em 2º etc., posta no LinkedIn e depois reseta pontuacao_semana = 0 para todos.
Configuração do Agente Zaia e Integração com N8N
Para que o agente no Zaia colabore eficientemente com o N8N, iremos configurar uma sequência de steps (passos) no painel do Zaia, usando possivelmente um Template de Agente adequado (a documentação da Zaia fornece modelos prontos que podem ser adaptados
instagram.com
). Abaixo sugerimos a configuração:
Step de Início (Opcional – Boas-vindas): Quando um novo usuário envia “oi” ou quando iniciamos pela primeira vez, o agente pode enviar uma mensagem de boas-vindas explicando rapidamente o funcionamento: "Olá! Eu sou o EstudeiCards, seu agente de estudo diário de DevOps. Todo dia enviarei 3 perguntas para você responder. Vamos começar!". Este passo pode ser útil caso usuários entrem em contato fora do fluxo programado (por exemplo, mandando mensagens avulsas). O agente pode reconhecer algumas palavras-chave como “quiz”, “start” etc., e responder com orientação.
Trigger de Envio Diário: O envio da primeira pergunta do dia, no nosso caso, é iniciado pelo N8N via API (push). Ou seja, não é exatamente o agente que inicia sozinho (a não ser que programássemos uma automação interna do Zaia, mas é mais fácil deixar o N8N gerenciar o agendamento). Assim, o Zaia deve expor uma forma de receber do N8N uma chamada para enviar mensagem. Normalmente isso ocorre via API do WhatsApp Business associada ao agente. Então aqui não é um “step” manual, mas sim a integração N8N->Zaia já descrita: uma requisição do N8N que, ao chegar no Zaia, aciona o envio de determinada mensagem template. Precisaremos configurar no Zaia um Template de mensagem ou formatação que o N8N possa invocar. A documentação da Zaia sugere que há como usar agentes em conjunto com ferramentas de automação (Make, Zapier, etc.), então provavelmente o Zaia fornecerá um endpoint webhook ou uma credencial/API token para permitir isso
zaia.app
.
Step de Recebimento de Resposta (Entrada do Usuário): Este é crucial. No construtor de agentes da Zaia, configuraremos que toda mensagem recebida do usuário durante o quiz seja tratada por um step que:
Capture o texto da resposta.
Chame uma ação de webhook/HTTP para enviar os dados ao N8N. A configuração envolverá o URL do webhook do N8N (que teremos após publicar o fluxo ou usando o tunnel se local), o método (POST) e os campos a enviar. Incluiremos identificadores necessários – possivelmente usando as variáveis do agente para pegar o telefone do usuário e o conteúdo da mensagem.
É importante que este step espere pela resposta do N8N antes de prosseguir. Ou seja, deve ser uma chamada síncrona cujo resultado (payload de retorno) possa ser utilizado nos passos seguintes. Se o Zaia permitir, faremos o webhook e armazenaremos a resposta em alguma variável do agente.
Possivelmente o Zaia tem um template específico chamado “Webhook Step” ou “API Integration Step” conforme a documentação de Templates de Agentes. Usaremos esse recurso aqui.
Step de Envio de Feedback (Saída do Agente): Assim que o N8N retornar com a avaliação, o agente Zaia passa para o próximo step que envia a mensagem de feedback ao usuário. Existem duas abordagens:
Agente apenas repassa texto pronto: A forma mais simples é o N8N já devolver exatamente o texto que o agente deve enviar (incluindo já a próxima pergunta se houver). Nesse caso, o step de saída pega o conteúdo da resposta do webhook (por ex., campo message do JSON retornado) e usa como mensagem ao usuário. Assim, toda lógica de formatação fica no N8N e o agente é “bobo”, só transmite.
Agente com lógica de decisão: Alternativamente, o N8N poderia retornar dados estruturados, tipo { "correct": true, "feedback": "Correto!... ", "nextQuestion": "Pergunta 2: ...", "end": false }. A configuração do agente poderia então ter um step que verifica a variável correct e decide enviar uma mensagem diferente para acerto ou erro (permitindo por exemplo mudar tom de emoji). Porém, isso pode ser overengineering – mais simples deixar a frase pronta. Ainda assim, poderíamos fazer: Step 4A para acerto e 4B para erro, cada um com uma mensagem template preenchida com os detalhes (como a explicação e próxima pergunta). Essa lógica seria definida usando condições no builder do Zaia.
O step de envio efetivamente coloca na conversa do WhatsApp o conteúdo de feedback. Após este envio, o agente pode então decidir se houve fim de fluxo ou se volta a esperar nova resposta.
Loop das Perguntas: Precisamos garantir que o agente consiga lidar com até 3 perguntas em sequência no mesmo chat:
Se usamos a abordagem de mensagem composta (feedback + próxima pergunta juntos), então após o Step 4 de envio, o agente automaticamente ficará pronto para novamente receber a resposta do usuário (ele retorna ao step 3 de receber input). Isso forma um loop 3 vezes.
Podemos controlar a saída do loop verificando se já foram feitas 3 perguntas. Como o agente não “sabe” sozinho quantas perguntas já se passaram (a não ser que mantenhamos um contador no contexto do agente), provavelmente delegamos isso ao N8N. O N8N ao retornar após a terceira pergunta indicaria algo como "end": true. O agente Zaia poderia ter uma condição: se end == true, ao invés de voltar ao step de input, vai para um step final.
Step Final: Esse step final simplesmente agradece ou encerra: pode ser uma mensagem “Obrigado por participar hoje! 🚀 Até amanhã.”. Contudo, essa mensagem já estará inclusa no feedback final enviado pelo N8N, então pode não ser necessário um step separado. De todo modo, garantindo que após a terceira pergunta o agente não fique aguardando mais respostas para aquele fluxo.
Tratamento de Exceções (Fallbacks): Precisamos também configurar comportamentos para situações excepcionais no agente:
Se o usuário envia algo fora do contexto (ex: manda uma pergunta aleatória ou uma saudação fora do horário do quiz): o agente pode responder com um lembrete do funcionamento ou uma mensagem padrão (“No momento estou ativo apenas para enviar questões diárias. Aguarde o próximo quiz ou digite 'ajuda' para instruções.”).
Se o webhook para N8N falhar (ex: N8N fora do ar): o agente deve ter um fallback do tipo “Desculpe, estou passando por um problema técnico. Por favor, tente novamente mais tarde.”. Isso evita o bot ficar mudo caso ocorra um erro. Podemos implementar isso usando as opções de timeout/erro do step de webhook no Zaia (geralmente dá para definir uma mensagem de erro padrão se a API não responder).
Se o tempo de resposta do usuário exceder um limite (ex: usuário some após 1ª pergunta): O agente pode encerrar a sessão após algumas horas de inatividade para não ficar aguardando indefinidamente. Ainda assim, se o usuário responder tardiamente no mesmo dia, poderíamos optar por continuar de onde parou, ou informar que aquele quiz expirou. Para simplificar, podemos aceitar respostas atrasadas dentro do mesmo dia.
Integração com N8N (Make/N8N Connection): Segundo a documentação da Zaia, há templates demonstrando integração com o Make. No nosso caso, faremos com N8N, mas o princípio é o mesmo: API calls. Citando a documentação: "Conecte a Zaia ao... WhatsApp... ou qualquer outro canal por meio de integração via API
zaia.app
*.“ Ou seja, configuraremos a Zaia de forma que:
Tenhamos as credenciais de API necessárias (token, URL base).
No N8N, usaremos possivelmente um nó HTTP Request para mandar mensagens via Zaia.
No Zaia, configuramos os steps de webhook para apontar para N8N. Essa colaboração é o coração da solução, unindo o poder de fluxo do N8N com a interface conversacional da Zaia.
Em resumo, a configuração do agente Zaia será relativamente simples, pois a maior parte da lógica está no N8N. O agente atua como intermediário de mensagens:
Enviando o que o N8N manda,
Recolhendo o que o usuário responde e repassando para o N8N.
Essa separação é benéfica: se amanhã quisermos trocar o canal (por exemplo, usar Telegram), a lógica no N8N de perguntas/pontuação/feedback pode ser a mesma, só trocaríamos o canal de envio.
Gamificação Semanal e Ranking
A gamificação é central para engajar os participantes continuamente. Eis como será aplicada:
Sistema de Pontos: Cada resposta correta rende pontos. Conforme sugerido, adotaremos pontuação distinta por nível de dificuldade para valorizar conhecimentos avançados. Exemplo: Fácil = 10 pontos, Pleno = 20 pontos, Expert = 30 pontos (ajustável). Assim, se num dia o usuário acerta 1 fácil e 1 expert (erra uma plena), ele faria 40 pontos. Pontos incentivam o usuário a buscar acertar, mas mesmo erros trazem aprendizado pelo feedback imediato.
Ranking Semanal: Os pontos acumulam ao longo da semana (segunda a domingo, ou podemos contar segunda a domingo e divulgar na segunda-feira seguinte). Toda semana, todos os participantes competem pelos melhores scores. O Top 10 de cada semana ganha reconhecimento especial na comunidade:
Na segunda-feira, conforme detalhado, o N8N computa quem foram os 10 maiores pontuadores.
Esse ranking é então publicado no LinkedIn do DOUG BR para celebrar os vencedores. A publicação atua como quadro de honra, motivando participantes. Se possível, mencionaremos (tag) os usuários – o que dá ainda mais recompensa social: eles terão seus nomes em destaque para toda a comunidade ver, o que é uma grande motivação extrínseca (especialmente para profissionais querendo visibilidade). Caso a API não permita menções automáticas, pelo menos os nomes serão escritos, e um administrador poderia manualmente editar a postagem depois para marcar os perfis.
Exemplo de conteúdo: "📢 Ranking Semanal - DevOps User Group BR 📢 \n1️⃣ @João Silva – 150 pts\n2️⃣ @Maria Pereira – 140 pts\n... \n🏅 Parabéns aos destaques da semana! Continue participando do DevOps EstudeiCards e melhore suas skills enquanto se diverte. Toda segunda-feira zera o placar, então todos têm chance de aparecer aqui.*".
Reinício (Reset) Semanal: Como já mencionado, o reset semanal dos pontos é fundamental. Isso mantém a competição justa e emocionante
nudgenow.com
. Participantes novos não se sentem desmotivados por chegar tarde – toda semana é uma nova oportunidade. E participantes recorrentes têm um estímulo constante para voltarem toda semana e defenderem seu lugar. Essa mecânica, inspirada em muitas plataformas (como leaderboards de cursos, Duolingo, etc.), evita que os mesmos sempre liderem indefinidamente
nudgenow.com
.
Premiação e Reconhecimento: Embora o projeto seja gratuito e não envolva prêmios materiais, o reconhecimento dos top performers já é um elemento de premiação. Além disso, podemos:
Conceder badges virtuais semanais: por exemplo, um pequeno certificado digital ou selo “Vencedor da semana X do EstudeiCards” que o usuário pode compartilhar no LinkedIn. Isso poderia ser gerado automaticamente (talvez usando uma imagem composta com o nome do vencedor, via alguma API gráfica, e enviado individualmente – seria um plus não obrigatório).
Mencionar os vencedores também em grupos do WhatsApp ou no próprio grupo DOUG BR (caso exista fora do LinkedIn), para prestígio perante os peers.
Feedback Contínuo: A gamificação também está presente no feedback diário. O usuário não fica no escuro – ele sabe a cada resposta se pontuou ou não, e sabe ao final do dia como foi. Essa transparência imediata é importante para a motivação diária. Além disso, o agente pode eventualmente enviar a posição parcial do usuário no ranking durante a semana. Ex.: no resumo diário de sexta, poderíamos dizer “Você está em 3º lugar até agora!”. Isso cria uma competição saudável e encoraja a pessoa a se esforçar nas próximas perguntas. (Esse recurso é opcional; podemos adicionar se o volume de usuários não for muito grande, pois requer calcular ranking parcial, mas é simples: no momento do resumo diário, fazer uma query para ver quantos têm pontuação maior que a do usuário e derivar posição.)
Participação e Inclusão: Para que a gamificação seja positiva, cuidaremos da linguagem para ser sempre encorajadora. Mesmo usuários com baixo desempenho devem se sentir motivados a continuar, pelo tom amistoso e pelo reset semanal que dá “nova vida” regularmente. A ideia é que ninguém desanime por ficar atrás, e sim tente novamente na semana seguinte – exatamente o que o reset proporciona
nudgenow.com
.
Ciclo de Engajamento: A cada semana, o processo se reinicia. Podemos enviar toda segunda-feira, junto com as perguntas do dia, uma mensagem curta: "Boa sorte na nova semana do EstudeiCards! 💥 Você tem uma nova chance de chegar ao topo do ranking." Isso marca o novo ciclo e lembra todos que agora é hora de competir novamente.
Em termos de implementação, o fluxo semanal no N8N cobrirá:
Gatilho (cron) segunda-feira cedo.
Função para calcular ranking (ou diretamente um nó de banco que ordena e limita 10).
Nó de LinkedIn: formata e publica o post
n8n.io
.
Nó de banco para resetar pontos (um simples UPDATE usuarios SET pontuacao_semana = 0).
(Opcional) Loop para enviar mensagem individual via WhatsApp aos Top 10 avisando que eles apareceram no ranking, parabenizando-os. Isso seria um mimo extra – ex.: "Parabéns, você ficou em 2º lugar no ranking da semana passada! 🎉 Confira a postagem no LinkedIn da comunidade." Essa etapa não é estritamente pedida, mas é uma possibilidade de extensão.
Viabilidade de Mensagens via LinkedIn e Plano de Contingência
Conforme já destacado, existe dúvida sobre a possibilidade de envio de mensagens inbox via LinkedIn API pública. Recapitulando os achados: a menos que o projeto tenha acesso a APIs específicas (geralmente restritas a parceiros e casos de uso como recrutamento ou vendas via LinkedIn), não há endpoint oficial para enviar mensagem privada a um usuário do LinkedIn por meio de apps de terceiros
community.make.com
. Dito isso:
Validação Técnica: Tentaremos usar a API do LinkedIn Pages Messages (se o DOUG BR tiver uma página). Essa API permite que uma página envie mensagem para usuários que iniciaram contato com ela
linkedin.com
. Porém, demandaria que os membros da comunidade primeiro mandassem uma mensagem para a página (o que não é garantido/escala). Logo, isso não serve ao propósito de envio pró-ativo diário.
Conclusão: As perguntas diárias não serão enviadas via LinkedIn inbox, focando exclusivamente no WhatsApp (que é um canal mais direto e ubiquo, com opt-in dos participantes via número de telefone).
Fallback 100% WhatsApp: Todo o sistema funcionará integralmente via WhatsApp caso o LinkedIn não permita automação nas mensagens privadas. Isso significa:
O agente Zaia no WhatsApp será o único meio de entrega e interação do quiz.
Os usuários precisam fornecer e confirmar seu número de WhatsApp ao se cadastrarem.
A comunicação one-to-one permanece no WhatsApp, que tem a vantagem de ser instantânea e já muito utilizada pelo público alvo.
O LinkedIn, então, será utilizado apenas para comunicação one-to-many (divulgação do ranking semanal na página do grupo), o que está de acordo com as políticas da plataforma e é realizável via API de posts.
Caso desejássemos alguma integração com LinkedIn para efeito de login/cadastro (ex: validar perfis), poderíamos explorar a API de autenticação do LinkedIn, mas isso foge do escopo. Provavelmente um simples formulário de cadastro capturando nome, WhatsApp e LinkedIn (opcional) já basta para montar a base de usuários.
Possibilidade de Expansão: Se no futuro o LinkedIn abrir as portas para bots (ou se DOUG BR obtiver alguma parceria), poderíamos retomar a ideia de enviar quizzes por lá ou em grupos do LinkedIn. Até lá, manteremos a simplicidade com WhatsApp.
Em suma, o plano de contingência é na verdade o plano principal: assumir WhatsApp como canal único do agente. Todas as funcionalidades descritas (envio diário, respostas, feedback, etc.) estão garantidas via WhatsApp/Zaia. A única funcionalidade ligada ao LinkedIn que mantemos é a postagem do ranking semanal – que é menos interativa e mais de alcance comunitário.
Conclusão: Visão Integrada do Projeto
Em termos visuais, imaginemos o fluxo completo integrando tudo:
O N8N age como cérebro do sistema, programando quando disparar quizzes, calculando pontuações e decidindo o conteúdo a enviar. Ele conversa com:
O Banco de Dados PostgreSQL para ler/escrever usuários, perguntas e resultados.
O LLM (OpenAI/Gemini) para turbinar a geração de perguntas e avaliação de respostas, garantindo dinamicidade e profundidade no conteúdo
reddit.com
.
O Agente Zaia (WhatsApp) para entregar mensagens aos usuários e receber respostas, usando integrações API
zaia.app
.
A API do LinkedIn para publicar os resultados semanais e engajar a comunidade mais ampla no reconhecimento dos participantes
n8n.io
.
O Agente Zaia age como voz do sistema, na figura de um chatbot amigável no WhatsApp:
Envia diariamente 3 EstudeiCards (perguntas) por texto, possivelmente com formatação Markdown simples para destacar comandos ou opções.
Recebe as respostas e comunica de volta os acertos/erros quase instantaneamente, graças ao processamento automatizado do N8N.
Mantém o usuário engajado com uma experiência de chat fluida, como se estivesse conversando com um tutor pessoal de DevOps.
A Comunidade DOUG BR é alimentada por essa dinâmica:
Os membros aprendem e revisam conhecimentos DevOps constantemente em pequenas doses.
O aspecto de competição saudável os incentiva a participar todos os dias – afinal, cada dia é chance de somar pontos.
Semanalmente, há o ápice do jogo: o anúncio dos campeões, o que gera reconhecimento e possivelmente conversa nas redes (ex: pessoas comemorando no LinkedIn, motivando outros a entrar no quiz também).
A comunidade cresce em conhecimento compartilhado e em interação entre si, fortalecendo o grupo.
Este projeto, inteiramente open-source/free para os usuários, também demonstra na prática o poder de integrações modernas:
Orquestração de workflows com N8N, que permite conectar diferentes serviços de forma transparente
reddit.com
.
Uso de AI (LLM) junto com base de conhecimento estruturada, para criar um conteúdo educativo interessante e avaliar respostas de forma inteligente.
Utilização de canais populares (WhatsApp) de maneira automatizada, através de plataformas como Zaia que facilitam bots multicanais.
Divulgação em redes sociais (LinkedIn) para ampliar o alcance e engajamento, fechando o ciclo entre o aprendizado individual e o reconhecimento público.
Por fim, ao implementar o EstudeiCards Agentes, estaremos fornecendo à comunidade DOUG BR uma ferramenta contínua de aprendizado e engajamento. O plano acima cobre desde a concepção até os detalhes técnicos e de gamificação, garantindo um sistema robusto e envolvente. A cada semana, todos terão uma nova chance de competir e aprender, mantendo o ecossistema DevOps sempre vivo e em evolução 🚀. Referências Utilizadas: N8N e integrações
reddit.com
zaia.app
; Limitações da API do LinkedIn
community.make.com
; Geração de perguntas via LLM/RAG
reddit.com
; Boas práticas de leaderboards semanais
nudgenow.com
; Fluxos de automação para LinkedIn posts
n8n.io
.
