ARQUIVO PDF DA DOCUMENTAÇÃO: [Documentação Técnica_ Algoritmo Keeta - Rota Imune.pdf](https://github.com/user-attachments/files/26305588/Documentacao.Tecnica_.Algoritmo.Keeta.-.Rota.Imune.pdf)

Documentação Técnica: Algoritmo Keeta - Rota Imune
Versão: 1.0.0
Tipo de Arquitetura: Event-Driven (Orientada a Eventos)
Linguagem: HTML, CSS e Javascript

1. Visão Geral
O nosso algoritmo é um motor de roteirização e despacho logístico focado na maximização de margem financeira, retenção de frota e simulação realista, com várias variáveis diferentes em jogo. Diferente de roteirizadores lineares, o sistema opera sob o gerenciamento de Máquinas de Estado Finito (FSM), reavaliando continuamente a malha urbana, as restrições físicas dos veículos e as variáveis caóticas do ambiente antes de consolidar rotas.

2. Máquina de Estados (State Machine)
O ecossistema rastreia o ciclo de vida de cada entidade no sistema através de estados mutáveis em tempo real.

2.1. Ciclo de Vida do Pedido
Os diferentes estados que nossos entregadores poderão variar são:
Aguardando Dispatch: Pedido criado, aguardando formação de liquidez (Holding Time).
JIT (Postergado): Entregador alocado, mas notificação retida na nuvem aguardando o tempo de preparo da cozinha.
Indo ao Restaurante: Entregador acionado e em deslocamento.
Aguardando Preparo: Entregador no raio físico do restaurante.
Em Rota para Entrega: Pedido em posse do entregador.
Aguardando Cliente (No-Show): Entregador na coordenada de destino sem resposta do cliente.
Fila de Emergência: Pedido extraído de um entregador acidentado, com prioridade máxima de reatribuição.

3. Fluxo de Execução do Algoritmo (Pipeline)
A alocação não ocorre de forma imediata à criação do pedido. Ela segue um pipeline sequencial de 4 fases para otimização de matrizes.

Fase 1: Holding Time
Pedidos recebidos são retidos temporariamente na fila (filaDePedidos). O objetivo é criar densidade artificial de dados para permitir ao algoritmo encontrar rotas sobrepostas, evitando a ociosidade do despacho unitário instantâneo.

Fase 2: Varredura Combinatória e Batching
O sistema cruzar as coordenadas dos clientes que realizaram pedidos no mesmo restaurante. O agrupamento (Batching) ocorre somente se as seguintes travas forem superadas:
Trava de Service Level Agreement (SLA): A distância de desvio entre os Clientes A e B não pode exceder 3 Km.
Trava Volumétrica: O volume acumulado dos pacotes não pode exceder a variável de capacidade máxima (capMax) do veículo designado (Moto = 3, Bike = 2).

Fase 3: Modificadores Dinâmicos
Antes de fechar a rota, a matriz também sofre influência de variáveis externas:
Clima (isChovendo): Reduz o multiplicador de velocidade (Motos caem 30%, Bikes 40%) e força a desconexão de parte da frota não-motorizada (isOffline = true).
Gargalo de Produção (pedidosAtivos): O tempo de preparo se altera sob demanda usando a equação: Tempo de Preparo Real = Base + (Pedidos na Fila * 2 min). Sendo a “Base” o tempo médio de preparo do pedido e “Pedidos na Fila” sendo a quantidade de pedidos feitos para aquele restaurante.
Risco e Score (VIP Routing): Se o Gross Merchandise Volume (GMV) do lote for >R$100, a matriz de entregadores é filtrada, aceitando apenas parceiros com a variável Score >= 4.8.

Fase 4: Sincronização Just-in-Time (JIT)
O motor calcula o tempo exato de deslocamento dos entregadores na área até o restaurante.
Tempo Postergado = Tempo Preparo Real - Tempo Viagem Moto
Se o resultado for positivo e o entregador aceitar a corrida, a thread do entregador entra em um estado de espera na nuvem. A notificação de despacho é atrasada propositalmente para mitigar o tempo de espera física na calçada, possibilitando o entregador à trabalhar em outros pedidos e aumentando sua produtividade.

4. Motor Econômico (Unit Economics)
O DRE (Demonstrativo de Resultados) transacional é calculado por pedido, separando a operação em dois modelos de Take Rate por restaurante e manipulando o custo marginal logístico. (Dados adquiridos do próprio sistema financeiro do Ifood e adaptados para a demanda da Keeta).

4.1. Receita (Take Rate)
Marketplace (Frota Própria): Comissão de 12% + 3,2% (Taxa de processamento online).
Full Service (Frota App): Comissão de 25% + 3.2% (Taxa de processamento online).

4.2. Custo Logístico (Remuneração dos Entregadores)
A liquidação do frete para o parceiro baseia-se na identificação do tipo da viagem (Simples/ sem Batching ou com Batching).
Simples/ sem Batching: (R$7,50 + R$1,50 * Distância em Km). 
Com Batching: Entregas agrupadas na mesma bag ignoram a fórmula quilométrica e aplicam um custo marginal travado de R$3,00 por pacote extra. A receita extra do cliente torna-se margem líquida da plataforma.

4.3. Custo de Retenção (Courier Retention Cost - CRC)
Alavancas financeiras financiadas pela margem líquida para proteger o Supply (oferta de entregadores):
Tarifa Dinâmica (Surge Pricing): Bônus variável de R$2,50 a R$5,00 injetado sob alta volumetria na variável filaDePedidos ou chuva, com foco em incentivar os entregadores parceiros à se manterem em nosso aplicativo, atendendo, assim, a grande demanda.
Deadhead Miles (Taxa de Retorno): Subsídio de R$3,00 adicionado se a distância de retorno do cliente final ao centro de atividade anterior for >3Km.
Gamificação (Lock-in): Injeção de R$10,00 a cada corridasCompletadas == 5 para estimular a extrapolação de metas dos entregadores parceiros. No entanto, este ponto ainda está sujeito à mudanças, visto que a Gamificação levará em foco o número médio de entregas feitas por dia por entregador, assim extrapolando essa média e incentivando-o a fazer mais corridas.

5. Tolerância a Falhas (Failover & Recuperação)
O algoritmo possui resiliência contra as ineficiências do mundo físico (Engenharia de Caos):
No-Show: Probabilidade de 5% do cliente não comparecer, travando o vetor de locomoção por 10 minutos adicionais, degradando a margem, mas finalizando a entrega.
Acidentes da Frota: Quando a flag quebrado = true é acionada, o sistema realiza o isolamento da entrega, intercepta a carga do loteAtual, injeta os dados na filaRecuperacao (índice 0, prioridade máxima) e força um redespacho com a frota remanescente, evitando estornos e perdas de inventário.

Parâmetros e Constantes Globais Base
VEL_MOTO_KMH: 30 km/h (Modificável pelo clima)
VEL_BIKE_KMH: 15 km/h (Modificável pelo clima)
RAIO_BATCHING_MAX: 3.0 km
TAXA_NO_SHOW: 5%
CAPACIDADE_MAX_MOTO: 3 volumes
CAPACIDADE_MAX_BIKE: 2 volumes


