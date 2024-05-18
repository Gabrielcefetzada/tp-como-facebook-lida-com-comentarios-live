

# Decisões Arquiteturais

## Introdução

A Twitch é uma das maiores plataformas de transmissão de vídeo ao vivo do mundo, com uma infraestrutura complexa que suporta uma variedade de serviços essenciais. A plataforma enfrenta desafios significativos devido à sua escala e ao rápido crescimento, exigindo decisões arquiteturais cuidadosas.

## Escalabilidade

 A Twitch lida com mais de 30.000 streamers simultâneos e picos de mais de 2 milhões de streams de vídeo ao mesmo tempo.

 O sistema de chat entrega mais de 10 bilhões de mensagens por dia.

 As APIs web tratam cerca de 50 mil requisições por segundo.

 A arquitetura deve suportar esse volume massivo de dados e acessos.

## Crescimento de Equipe

A equipe de engenharia cresceu 800% nos últimos 3 anos.

A arquitetura precisa ser modular e escalável para acomodar mais desenvolvedores sem comprometer a eficiência.

## Gerenciamento de Falhas

Em sistemas grandes, falhas de hardware e rede são comuns.

A arquitetura precisa incluir planejamento de capacidade e resiliência para mitigar essas falhas.

## Complexidade Adicional

Cada novo sistema adicionado aumenta a complexidade geral.

É necessário um gerenciamento eficiente para manter a coesão e funcionalidade dos sistemas interconectados.

## Infraestrutura Híbrida

Uso de pontos de presença (POPs) próprios para garantir alta qualidade de vídeo.

Migração crescente para Amazon Web Services (AWS) para reduzir sobrecarga operacional e aproveitar a escalabilidade dos serviços da AWS.

## Principais Componentes da Twitch

### Sistema de Vídeo

Ingestão de Vídeo: Recebe streams de vídeo em RTMP e os transporta para o sistema de transcodificação.

Transcodificação: Converte streams RTMP em múltiplos streams HLS, utilizando C/C++ e Go.

Distribuição e Edge: Distribui os streams HLS para pontos de presença (POPs) ao redor do mundo, garantindo alta qualidade de streaming.

Vídeo sob Demanda (VOD): Arquiva os vídeos transmitidos para o sistema de VOD.

### Sistema de Chat

Escalabilidade e Protocolos: Um sistema distribuído escrito em Go, suportando IRC via TCP e WebSockets.

Edge e Pubsub: Distribui mensagens internamente e para os clientes.

Clue: Analisa ações dos usuários e aplica lógica de negócios (ex: bans, comportamento abusivo).

Room: Gerencia e consulta dados de membros nas salas de chat.

## APIs Web e Dados

Gerenciamento de Perfis e Assinaturas: APIs para personalização de perfis.

Busca e Descoberta: Serviços para encontrar streams.

Sistemas de Receita: Gerenciamento de publicidade e assinaturas.

Ferramentas Administrativas: Suporte ao atendimento ao cliente.

Tecnologias Usadas: Mix de Rails, Go e várias aplicações open-source para roteamento, caching e armazenamento de dados.

## Aplicações Web e Cliente

Aplicações Web: Evolução de Rails para Ember.

Aplicativos Nativos: Plataformas móveis (iOS e Android) e consoles (XBox, Playstation).

## Infraestrutura de Ciência de Dados

Pipeline de Dados: Coleta, limpeza e carregamento de eventos para análise.

Agregadores de Streaming: Resumos de métricas em tempo real para os streamers.

## Ferramentas e Infraestrutura Operacional

Teste e QA: Utilização de Jenkins para build e testes automatizados.

Desdobramento e Rollback: Ferramentas caseiras para deploy rápido.

Monitoramento e Alertas: Ferramentas como Ganglia, Graphite e Nagios.

Rastreamento Distribuído: Ferramenta para depuração de sistemas distribuídos.

## Considerações Finais

A arquitetura da Twitch é um exemplo de como gerenciar uma plataforma em larga escala, com diversos componentes interconectados e desafios únicos de escalabilidade, complexidade e crescimento. Essas informações são cruciais para entender as decisões arquiteturais necessárias para suportar uma plataforma de streaming de vídeo ao vivo de grande porte.





