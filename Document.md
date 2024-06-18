

# Decisões Arquiteturais

## Introdução

A Twitch é uma das maiores plataformas de transmissão de vídeo ao vivo do mundo, com uma infraestrutura complexa que suporta uma variedade de serviços essenciais. A plataforma enfrenta desafios significativos devido à sua escala e ao rápido crescimento, exigindo decisões arquiteturais cuidadosas.

A Twitch.tv foi lançado em 6 de junho de 2011, após seu início como Justin.tv em 2006. Foi construída como um monolito usando Ruby on Rails. Ter tudo junto em uma única base de código é uma ótima maneira de lançar uma startup e, inicialmente, funcionou bem para o Twitch. Isso permitiu um desenvolvimento rápido enquanto usava práticas de codificação de ponta.

No entanto, à medida que as organizações crescem, existem vários motivos para dividir o código em partes menores.

## Grandes Gargalos de desempenho Pós Monolito

À medida que a base de usuários do Twitch aumentava, diferentes partes do sistema começaram a atingir seus limites. API, vídeo, banco de dados, chat, busca, descoberta, comércio e outros sistemas internos precisavam de melhorias. Para ilustrar a situação, vamos usar o sistema de chat inicial como exemplo.

Na época, o sistema de chat funcionava em um conjunto de 8 máquinas, com canais de chat distribuídos aleatoriamente entre essas máquinas. Essa arquitetura funcionava bem até que eventos de jogos começaram a explodir em popularidade. Grandes canais chegavam a ter a (então) inacreditável quantidade de 20.000 espectadores! Na época (2010), esses números eram incríveis. Hoje em dia, nem tanto.

De qualquer forma, grandes canais desaceleravam a máquina que hospedava o sistema de chat. Em alguns casos, demorava mais de um minuto para estabelecer uma conexão. Além disso, o problema se espalhava para todos os outros canais atribuídos ao mesmo host. Com apenas 8 máquinas, isso significa 1/8 de todos os canais! Investir mais dinheiro no problema (adicionando mais máquinas) reduziria o impacto, mas não resolveria o problema principal.

As primeiras tentativas de construir microserviços foram feitas para eliminar gargalos como esses. Os engenheiros trabalharam contra o relógio para encontrar novas soluções que pudessem escalar. Houve muita experimentação, e a maioria das tentativas nem chegaram à produção. Por exemplo, a equipe que trabalhava no sistema de chat inicialmente tentou melhorar a arquitetura existente usando Sidekiq em um backend RabbitMQ. Claro, essa implementação estava dentro do Rails e só tornou o monólito maior. Em seguida, tentaram extrair o sistema de chat para um serviço NodeJS, mas, infelizmente, o conceito inicial não conseguiu escalar devido a um novo bug encontrado no núcleo do NodeJS (ainda era uma tecnologia jovem). Procurando outras opções, encontraram Python com Tornado, que parecia funcionar tão bem quanto, mas com menos surpresas. Finalmente, a implementação em Python se tornou o novo serviço de chat. Embora essa extração inicial não qualificasse como um microserviço moderno, foi um passo na direção certa.

## Primeiros Microserviços

O primeiro código em Go a rodar em produção foi um pequeno servidor Pubsub (também conhecido como troca de mensagens). Era uma peça pequena, mas importante, do sistema de chat. Com apenas cerca de 100 linhas de código, ele manipulava todas as mensagens no Twitch através de uma única thread.

Essa foi uma excelente oportunidade para testar o Go e, após uma prova de conceito inicial, a equipe substituiu o servidor Pubsub por uma nova implementação em Go. Foi uma vitória fácil. O desempenho por thread dobrou em relação à implementação anterior em Python e poderia ser modificado para rodar em múltiplas threads usando goroutines.

O novo servidor Pubsub adicionou impulso à linguagem de programação Go no Twitch. Outras equipes também estavam experimentando com ela, e logo o próximo microserviço importante em Go foi criado. A equipe de vídeo nomeou o novo serviço de Jax.

Jax foi projetado para listar as principais transmissões ao vivo de cada categoria na página inicial do Twitch. O desafio era indexar dados ao vivo em tempo quase real, mantendo a fonte de dados devidamente organizada, para que pudesse ser gerenciada por diferentes equipes. O sistema de vídeo sabe sobre o número de espectadores ao vivo, mas não se importa com metadados de categoria. Construir o novo serviço Jax fora do Rails e do sistema de vídeo permitiu uma separação adequada de responsabilidades. Depois disso, o Jax passou por várias iterações até que o esforço inicial pudesse valer a pena: primeiro usando Postgres, depois índices em memória, e finalmente Elasticsearch.

## Migração Massiva para Go

Com exceção de alguns microserviços como o Jax, a grande maioria do site ainda funcionava dentro do monólito Rails. Os desenvolvedores destacavam vários desafios ao trabalhar com a base de código existente. Por exemplo:

- Uso de um único pipeline de deploy para todos
- Espera por builds cada vez mais lentos
- Dificuldade para rastrear erros até seus responsáveis
- Roteamento lento devido a muitos endpoints de API
- Consultas de banco de dados lentas
- Constante enfrentamento de conflitos de merge

Talvez o esforço mais importante foi a adição de uma camada extra na frente da API backend: um proxy reverso NGINX, usado para redirecionar uma porcentagem do tráfego para uma nova borda de API. A nova borda de API foi escrita em Go, usando middleware comum para validação, autorização e limitação de taxa, mas delegando toda a lógica de negócios para os microserviços subjacentes.

Com o novo proxy em vigor, o processo para migrar endpoints foi definido:

- Desenvolva seu novo microserviço.
- Replique o endpoint antigo na nova borda.
- Atualize o arquivo de configuração do NGINX para enviar uma porcentagem do tráfego para o novo endpoint.
- Verifique as métricas e continue aumentando o tráfego até 100%.

Chamar um módulo dentro do monólito significa chamar uma função ou método, mas chamar outro serviço requer muito mais: tratamento de erros, monitoramento, circuit breakers, rate limiting, rastreamento de solicitações, versionamento de API, segurança s2s, autorização restrita, testes de integração confiáveis e muito mais. Quando alguém descobria algo, sua solução era frequentemente copiada a uma taxa alarmante, replicando problemas mais rápido do que poderiam ser corrigidos. Houve muita duplicação.

Outras funcionalidades permaneciam sem dono, funcionando como esperado do ponto de vista do negócio, mas ficando para trás nas práticas de segurança e qualidade. Para lidar com a longa cauda, alguns funcionários formaram um grupo de trabalho interno com o objetivo específico de monitorar e migrar esses últimos serviços.

Endpoints remanescentes ainda recebiam tráfego de fontes desconhecidas, talvez bots ou scripts, e não estava claro o que esses endpoints deveriam fazer. A única maneira de saber se era seguro desativá-los era forçar 10 minutos de inatividade e ver se alguém reclamava.

No final de 2018, quando todo o tráfego foi desviado do monólito, finalmente foi hora de parar todas as instâncias EC2 na conta principal da AWS. Ainda levaria mais um ano para limpar completamente a conta de dependências e papéis de acesso (trabalho que foi concluído através de auditorias de segurança em toda a empresa e atualizações de tecnologia), mas para todos os efeitos, a campanha Wexit foi concluída.

Hoje em dia, diferentes equipes de produto podem operar de forma independente, gerenciando sua própria infraestrutura e ajustando seus serviços conforme necessário. Equipes fundamentais fortes fornecem frameworks e ferramentas específicas para ajudar a padronizar a pilha de engenharia e as práticas operacionais.

## Detalhamento e estratégias para Live Video Stream em Escala Global

Para fornecer serviços de streaming de alta qualidade e baixa latência para criadores e espectadores, o Twitch mantém quase uma centena de pontos de presença (PoPs) em diferentes regiões geográficas ao redor do mundo. Cada PoP está conectado à nossa rede backbone privada, e as transmissões de vídeo ao vivo recebidas nos PoPs são então enviadas para os data centers de origem para processamento e distribuição.

Os data centers de origem hospedam os recursos necessários para fornecer transformações de mídia computacionalmente intensivas, como transcodificação de vídeo. O Twitch tem uma comunidade global de criadores e espectadores, então configuramos data centers de origem em todo o mundo para processar as transmissões ao vivo dos nossos criadores e depois distribuí-las aos espectadores via rede backbone. Usar rede backbone para entregar tráfego de vídeo ao vivo permite alcançar uma alta disponibilidade, baixa latência e uma alta experiência de Qualidade de Serviço (QoS) para os usuários. Isso permite superar a instabilidade da Internet pública, especialmente ao entregar dados sensíveis ao tempo, como vídeo ao vivo. Um resumo de alto nível da infraestrutura é mostrado no diagrama a seguir.

![image](https://github.com/Gabrielcefetzada/tp-como-twitch-lida-com-requests/assets/63877012/15021c27-5319-48ca-a2b1-f237ebdfacb3)

## Desafios

Cada PoP ainda rodava HAProxy e enviava streams de vídeo ao vivo estaticamente para apenas uma das origens para processamento. Como as atividades de broadcast são altamente cíclicas (com um padrão diário), isso resultou em uma utilização ineficiente dos recursos de computação e rede. Por exemplo, uma origem operava com carga total durante as horas de pico de uma determinada região geográfica, mas a utilização se tornava mínima fora desse período. A figura a seguir ilustra como diferentes regiões geográficas podem impactar a utilização da nossa infraestrutura.

![image](https://github.com/Gabrielcefetzada/tp-como-twitch-lida-com-requests/assets/63877012/2b7725f4-91a7-4f7a-8f3d-9d26d2fbeb7b)

Os broadcasters nas regiões A e B entravam ao vivo em horários diferentes, e em cada região os PoPs enviavam streams de vídeo ao vivo para apenas uma das origens. Isso resultou em uma utilização ineficiente da infraestrutura ao longo do dia. Em resumo, cada uma das origens precisaria ser dimensionada para os picos de tráfego "regionais" em vez da capacidade total em todas as origens ser dimensionada para o pico de tráfego "global"; este último sendo muito menor que a soma de todos os picos regionais.

Dada a natureza relativamente estática da configuração do HAProxy, era difícil lidar com um aumento inesperado de tráfego de vídeo ao vivo (por exemplo, devido a um evento popular) ou reagir às flutuações do sistema (como a perda de capacidade em uma origem).

Cada origem possui uma quantidade heterogênea de recursos de computação e rede. Era difícil configurar o HAProxy para que os PoPs enviassem a quantidade "certa" de tráfego de vídeo ao vivo para as origens. Além disso, a demanda de tráfego de ingestão pode ser muito volátil de tempos em tempos devido a eventos de transmissão populares.

## Solução: Intelligest

Para superar os desafios mencionados anteriormente, apoiar melhor a crescente demanda por tráfego de vídeo ao vivo e possibilitar novos casos de uso e serviços para os broadcasters, foi preciso reformular a arquitetura de software dos PoPs e aposentar completamente o HAProxy. Veio o Intelligest, um sistema proprietário de roteamento de ingestão para distribuir inteligentemente o tráfego de ingestão de vídeo ao vivo dos PoPs para as origens.

A arquitetura do Intelligest consiste em dois componentes: o proxy de mídia Intelligest rodando em cada PoP e o Serviço de Roteamento Intelligest (IRS) rodando na AWS. O diagrama a seguir mostra um resumo de alto nível da arquitetura do Intelligest.

![image](https://github.com/Gabrielcefetzada/tp-como-twitch-lida-com-requests/assets/63877012/1749b97f-9c4e-45ea-a027-960d96208b5d)

Proxy de Mídia Intelligest

O proxy de mídia Intelligest é um componente do plano de dados e serve como o "gateway de ingestão" para a infraestrutura da Twitch. O proxy de mídia roda em todos os PoPs, coletivamente chamados de "borda Intelligest." O proxy de mídia termina as transmissões de vídeo ao vivo dos broadcasters, extrai as propriedades relevantes dessas transmissões e usa o IRS para determinar para quais origens as transmissões devem ser enviadas. Em geral, o proxy de mídia pode terminar qualquer número de protocolos de transporte de mídia (por exemplo, RTMP ou WebRTC), permitindo-nos usar um protocolo "canônico" de nossa escolha dentro da infraestrutura backbone do Twitch.

Serviço de Roteamento Intelligest (IRS)

O Serviço de Roteamento Intelligest (IRS) é o controlador de roteamento de ingestão e é responsável por tomar decisões de roteamento para as transmissões de vídeo ao vivo. Note que a arquitetura Intelligest não controla ou gerencia o roteamento L3 da rede backbone. O IRS é um serviço com estado e pode ser configurado para suportar roteamento baseado em regras para satisfazer diferentes casos de uso. Por exemplo, pode-se configurar o IRS de forma que transmissões ao vivo associadas a certos canais de vídeo sejam sempre encaminhadas para uma origem específica, independentemente de quais PoPs as transmissões de vídeo cheguem. Isso é útil no caso de haver canais com necessidades de processamento especiais que só podem ser atendidas em uma origem específica. Como outro exemplo, o IRS também pode rotear várias transmissões relacionadas para a mesma origem para processamento - tal capacidade pode ser útil em um cenário de transmissão premium onde há uma transmissão principal e uma de backup.

O diagrama acima ilustra um exemplo simples. Os broadcasters enviam suas transmissões de vídeo ao vivo para um PoP onde o proxy de mídia Intelligest está rodando. O proxy de mídia Intelligest consulta o serviço IRS para obter as decisões de roteamento para as transmissões e, em seguida, as encaminha para as origens correspondentes. O objetivo do serviço IRS é calcular decisões de roteamento para otimizar um objetivo do sistema.

## Aprimoramento do Intelligest - Capacitor e The Well

Embora essa abordagem funcione razoavelmente bem, ainda existem algumas limitações:

A solução de roteamento computada é estática e incapaz de lidar com padrões de tráfego inesperados e flutuações do sistema em tempo real. Por exemplo, uma perda repentina de capacidade de transcodificação de vídeo em um data center de origem precisa ser acompanhada por uma redução correspondente na quantidade de tráfego de ingestão nessa origem.

Precisava-se atualizar o roteamento de tempos em tempos, resolvendo o problema de otimização com a demanda de tráfego mais recente para capturar novos padrões de tráfego da população de broadcasters.

Para abordar as limitações discutidas anteriormente, foram desenvolvidos dois subsistemas que podem, respectivamente, rastrear a utilização dos recursos computacionais e de rede em toda a plataforma Twitch. Especificamente, o IRS agora é capaz de consultar dois serviços, nomeadamente Capacitor e The Well. Ambos os serviços são implementados na AWS. O Capacitor monitora os recursos computacionais em cada origem e pode detectar quaisquer flutuações de capacidade (por exemplo, alguns recursos computacionais estão offline devido a manutenção), para que possa saber quanta capacidade de processamento está disponível em cada origem a qualquer momento. O The Well monitora a rede backbone e fornece informações sobre o status dos links de rede, como utilização e disponibilidade a qualquer momento.

![image](https://github.com/Gabrielcefetzada/tp-como-twitch-lida-com-requests/assets/63877012/0bab803b-f8e7-4d91-8292-01d06aefcde1)

Com as informações de ambos os serviços, o IRS tem uma visão em tempo real da infraestrutura. O IRS usa um algoritmo guloso randomizado para calcular decisões de roteamento para resolver o problema de utilização de recursos.

Uma situação em que isso tem sido muito benéfico, é nos cenários de falha de origem. Normalmente, esses cenários incluem perda de capacidade computacional ou de rede em uma origem e são refletidos nos sistemas Capacitor ou The Well. Como resultado, o IRS pode direcionar automaticamente novas transmissões de vídeo ao vivo para uma origem não afetada para alcançar alta disponibilidade para nossos broadcasters. A arquitetura também é extensível para monitorar outros cenários de falha, como uma interrupção no plano de controle regional, de modo que o IRS será capaz de detectar e responder a uma variedade de situações importantes de falha.

## Considerações Finais

A arquitetura da Twitch é um exemplo de como gerenciar uma plataforma em larga escala, com diversos componentes interconectados e desafios únicos de escalabilidade, complexidade e crescimento. Essas informações são cruciais para entender as decisões arquiteturais necessárias para suportar uma plataforma de streaming de vídeo ao vivo de grande porte.


# Referências bibliográficas

[Twitch. Twitch Engineering: An Introduction and Overview. Blog da Twitch, 18 dez. 2015. Disponível em: https://blog.twitch.tv/twitch-engineering-an-introduction-and-overview-24b4ed4a1842. Acesso em: 18 maio 2024.](https://blog.twitch.tv/en/2022/04/26/ingesting-live-video-streams-at-global-scale/)

https://blog.twitch.tv/en/2022/03/30/breaking-the-monolith-at-twitch/?utm_source=Twitch+Blog&utm_campaign=Blog

https://blog.twitch.tv/en/2022/04/12/breaking-the-monolith-at-twitch-part-2/

