# EV ChargeOps: Plataforma de Gestão de Recarga em Infraestrutura Compartilhada

## Equipe Crew Olympus

| Nome | RM |
|------|-----|
| Gabriel Tavares Martins de Oliveira | RM573718 |
| Lucas Araujo de Carvalho | RM571060 |
| Matheus Henrique Pedrozo Traiba | RM571817 |
| Miguel Monteiro Moreira | RM572904 |
| Pedro Henrique de Lima Costa | RM573008 |

---

# Sobre o Projeto

O EV ChargeOps é o produto proposto no **Challenge 2026 da FIAP em parceria com a GoodWe**. O objetivo é criar uma plataforma que transforma sessões de recarga de veículos elétricos em infraestrutura compartilhada (condomínios, edifícios corporativos, etc) em dados estruturados, faturamento individual por usuário e inteligência operacional.

O problema que motivou o projeto é prático: quando vários moradores ou colaboradores usam o mesmo carregador, não há como saber quem consumiu o quê. O custo da energia vai para a conta coletiva e é dividido entre todos, independente do uso. A proposta é construir a plataforma sobre o carregador **GoodWe HCA G2**, lendo os dados de cada sessão via Modbus TCP, identificando o usuário pelo cartão RFID e calculando o valor a cobrar com base na energia efetivamente consumida.

---

# Frente 1: Contexto e Problema

Aprofundamentos escolhidos: A (análise de mercado) e C (dados públicos de EV no Brasil)

## Crescimento dos veículos elétricos no Brasil

Os números da ABVE mostram que a adoção de veículos elétricos no Brasil cresceu de forma consistente nos últimos anos e acelerou em 2025 e 2026:

- Frota eletrificada acumulada entre 2012 e janeiro de 2026: **645.407 veículos**
- Vendas em 2025: **223.912 unidades** (26% de crescimento frente a 2024)
- Emplacamentos em janeiro e fevereiro de 2026: **48.591 unidades**, 90% a mais que no mesmo período do ano anterior
- Pontos públicos e semipúblicos de recarga em maio de 2026: **25.429**, crescimento de 20,7% em três meses
- São Paulo lidera com 181.305 EVs, seguido por DF (48.502) e Rio de Janeiro (39.295)

Desse total de pontos instalados, 66% ainda são carregadores lentos AC e 34% são rápidos DC. Isso reforça que a recarga em casa e em condomínios, feita em corrente alternada, é e vai continuar sendo a forma mais comum de abastecer um veículo elétrico no Brasil.

## O problema nos condomínios e edifícios

Quando um ponto de recarga é instalado em área comum, surgem quatro problemas que precisam ser resolvidos para que a operação seja viável:

**Medição individual:** sem um medidor dedicado por sessão, o consumo vai para a conta coletiva do edifício e é rateado entre todos os condôminos, mesmo os que não têm veículo elétrico.

**Controle de acesso:** sem autenticação, qualquer pessoa com acesso ao local pode usar o carregador, sem nenhuma rastreabilidade.

**Gestão de carga:** com vários carregadores ligados ao mesmo tempo, o consumo agregado pode ultrapassar a demanda contratada do edifício, gerando multas ou quedas de energia.

**Exigência legal:** São Paulo aprovou a **Lei 18.403/2026**, que garante ao condômino o direito de instalar ponto de recarga na própria vaga e exige que o consumo seja medido individualmente para fins de cobrança. Ou seja, o condomínio precisa de um sistema assim para estar em conformidade com a lei.

## Como funciona uma sessão de recarga

Uma sessão de recarga passa por estados bem definidos, monitoráveis em tempo real pelo registrador 10017 do protocolo Modbus TCP do HCA G2:

1. **Ocioso sem plugue (estado 0):** o carregador está disponível e aguarda conexão.
2. **Plugue conectado (estado 1):** o veículo foi conectado fisicamente, mas a sessão ainda não iniciou.
3. **Handshaking (estado 2):** o carregador e o veículo trocam informações sobre capacidade de carga e autenticam o usuário (por RFID, app ou início automático).
4. **Carregando (estado 3):** a energia começa a fluir. O carregador registra em tempo real a potência entregue (kW), a corrente e tensão por fase e a energia acumulada na sessão (kWh).
5. **Sessão concluída (estado 4):** o veículo atingiu a carga completa ou o usuário encerrou manualmente. O carregador registra o horário de fim, a energia total entregue e a leitura final do medidor MID.

Durante esse fluxo, os principais dados gerados são:

| Dado | Registrador Modbus | Descrição |
|---|---|---|
| Horário de início | 10158 a 10160 | Ano/mês, dia/hora, minuto/segundo |
| Horário de fim | 10162 a 10164 | Ano/mês, dia/hora, minuto/segundo |
| Energia entregue na sessão | 10016 | kWh (ganho 10) |
| Leitura MID antes | 10170 | kWh com precisão de 0,01 |
| Leitura MID depois | 10172 | kWh com precisão de 0,01 |
| Duração total | 10166 | Segundos |
| Modo de início | 10076 | RFID, app, automático, etc. |

A diferença entre os registradores 10170 e 10172 (medidor MID) é a base mais confiável para faturamento, pois tem rastreabilidade metrológica. O registrador 10016 serve para monitoramento em tempo real durante a sessão.

## Modelos de negócio para recarga compartilhada

Existem diferentes formas de estruturar a cobrança em infraestruturas compartilhadas. Cada modelo tem implicações distintas para o gestor e para o usuário:

**Recarga gratuita:** o custo da energia é absorvido pelo condomínio ou empresa e dividido entre todos os condôminos ou funcionários, independente de quem carregou. É o modelo mais simples de operar, mas gera subsídio cruzado e tende a ser abandonado à medida que a frota cresce.

**Cobrança por kWh:** o usuário paga pelo volume exato de energia consumida na sessão. É o modelo mais justo e transparente, mas exige medição individual confiável — o que o medidor MID do HCA G2 viabiliza. É o modelo adotado pelo EV ChargeOps.

**Cobrança por tempo:** o usuário paga pela duração da sessão, independente da energia entregue. É mais simples de implementar, mas penaliza veículos com maior eficiência de carga e não reflete o custo real da energia.

**Assinatura mensal:** o usuário paga uma taxa fixa por mês que garante um volume de recargas ou um número de horas de uso. Facilita o planejamento financeiro do usuário, mas pode gerar ociosidade ou sobreuso dependendo do perfil de cada um.

**Rateio condominial:** o custo total da energia consumida por todos os carregadores no mês é dividido proporcionalmente entre os usuários, com base no número de sessões ou na energia individual de cada um. Exige controle de uso por sessão, mas não necessariamente um medidor MID por carregador.

No contexto do EV ChargeOps, o modelo adotado é a **cobrança por kWh**, por ser o mais alinhado à exigência da Lei 18.403/2026 (medição individual) e o que oferece maior transparência para o usuário e menor risco de conflito para o gestor do condomínio.

## Análise de mercado: o que já existe

Pesquisamos as principais soluções disponíveis para entender o que o mercado oferece e onde estão as lacunas, especialmente para o contexto brasileiro.

**Wallbox Pulsar Plus (Espanha):** carregador AC de 7,4 a 22 kW com balanceamento dinâmico de carga entre múltiplos pontos (Power Sharing), integração com energia solar e suporte ao protocolo OCPP. Não tem homologação ANATEL, o que complica a regularização no Brasil.

**Zaptec Go (Noruega):** 22 kW, com algoritmo de distribuição de carga entre dezenas de carregadores simultâneos e integração com sistemas de automação predial. Também sem homologação para o Brasil.

**ChargePoint (EUA):** plataforma SaaS consolidada para gestão corporativa de frotas, com autenticação por RFID ou app e relatórios detalhados. Não tem hardware distribuído no Brasil.

**Neocharge (Brasil):** é o principal concorrente nacional, com wallboxes de 3,7 a 22 kW e plataforma de gestão própria. Tem suporte técnico local, mas não tem integração nativa com sistemas fotovoltaicos e não expõe protocolo Modbus para integrações externas.

**GoodWe HCA G2 + EV ChargeOps:** é a única combinação com homologação ANATEL (nº 06795-24-02673), integração direta com inversores fotovoltaicos GoodWe via RS485, controle dinâmico de carga e protocolo Modbus TCP aberto para que a plataforma acesse os dados de sessão.

| Critério | Wallbox | Zaptec | ChargePoint | Neocharge | GoodWe HCA G2 + EV ChargeOps |
|---|---|---|---|---|---|
| Homologação Brasil | Não | Não | Não | Sim | Sim |
| Integração fotovoltaica | Sim | Sim | Não | Parcial | Sim |
| Faturamento individual | Sim | Sim | Sim | Sim | Sim |
| Protocolo aberto (Modbus/OCPP) | Sim | Parcial | Sim | Não | Sim |
| Gestão dinâmica de carga | Sim | Sim | Sim | Não | Sim |
| Ecossistema solar GoodWe | Não | Não | Não | Não | Sim |

---

# Frente 2: Base Regulatória e Técnica

Aprofundamentos escolhidos: B (exploração do portal SEMS+ e dados disponíveis) e C (APIs complementares: Open Charge Map e Google Places)

## ANEEL Resolução Normativa 1.000/2021

A RN 1.000/2021 consolidou os direitos e deveres dos consumidores de energia elétrica no Brasil e incorporou as regras da Resolução 819/2018 sobre recarga de veículos elétricos. Para o EV ChargeOps, os pontos mais relevantes são:

- Qualquer pessoa jurídica pode vender energia para recarga de veículos elétricos sem precisar de autorização específica da ANEEL.
- O preço cobrado ao usuário final não é tabelado: é negociado livremente pelo operador do ponto de recarga.
- As tarifas e condições precisam ser transparentes e visíveis para o consumidor antes de iniciar a sessão.
- A instalação deve seguir as normas ABNT NBR 5410 e NBR 16722.
- Para que o faturamento tenha validade legal e rastreabilidade metrológica, é recomendado o uso de medidor com certificação MID (Measuring Instruments Directive).

Na prática, isso significa que o EV ChargeOps pode operar a cobrança por consumo em condomínios sem burocracia adicional, desde que use um medidor MID e deixe as tarifas visíveis. O HCA G2 já suporta o medidor MID via RS485, o que atende esse requisito.

## GoodWe HCA G2: o hardware da solução

O HCA G2 é o carregador AC da GoodWe para uso residencial e compartilhado. Está disponível em três modelos:

| Modelo | Potência | Fases |
|---|---|---|
| GW7K-HCA-20 | 7 kW | Monofásico |
| GW11K-HCA-20 | 11 kW | Trifásico |
| GW22K-HCA-20 | 22 kW | Trifásico |

Tensão de operação: 220 V (monofásico) e 380 V (trifásico), padrão brasileiro. Proteção IP66 no carregador e IP55 no plugue. Conector Tipo 2 (IEC 62196-2).

**Interfaces de comunicação:**

| Interface | Uso | Alcance |
|---|---|---|
| RS485_A1/B1 | Comunicação com inversor fotovoltaico GoodWe | 100 m |
| RS485_A2/B2 | Comunicação com medidor MID | 100 m |
| LAN (RJ45) | Conexão com roteador e SEMS+ | 100 m |
| Wi-Fi (2,4 GHz) | Conexão sem fio com roteador | ~15 m |
| Bluetooth | Configuração local via app SolarGo | ~10 m |
| RFID (13,56 MHz) | Autenticação por cartão (ISO 14443A/B) | Leitura por contato |

**Modos de carregamento disponíveis:** Rápido, Prioridade FV, FV + Bateria, Agendado e Controle Dinâmico de Carga. O Controle Dinâmico ajusta a potência de carregamento com base na corrente disponível no quadro do edifício, evitando ultrapassagem de demanda.

**Formas de iniciar uma sessão:** cartão RFID (suporta até 10 cartões por carregador), app SEMS+ ou SolarGo, ou início automático ao conectar o veículo (sem autenticação).

**Estados possíveis do carregador (registrador Modbus 10017):**

| Código | Estado |
|---|---|
| 0 | Ocioso, sem plugue |
| 1 | Ocioso, plugue conectado |
| 2 | Handshaking com o veículo |
| 3 | Carregando |
| 4 | Sessão concluída |
| 5 | Alarme |
| 6 | Pré-agendado |
| 7 | Manutenção |
| 8 | Falha ao iniciar |
| 9 | Atualizando firmware |
| 10 | Interrompido |

## Integração via Modbus TCP

A comunicação entre a plataforma e o carregador é feita pelo protocolo Modbus TCP na rede local (protocolo RS485: 9600 bps, 8N1). Os registradores mais importantes para o EV ChargeOps:

**Monitoramento da sessão em andamento:**

| Registrador | Descrição | Unidade |
|---|---|---|
| 10009 a 10011 | Tensão por fase (A, B, C) | V (ganho 10) |
| 10012 a 10014 | Corrente por fase (A, B, C) | A (ganho 10) |
| 10015 | Potência de carregamento | kW (ganho 10) |
| 10016 | Energia acumulada nesta sessão | kWh (ganho 10) |
| 10017 | Status do carregador | 0 a 10 |
| 10063 | Duração da sessão em andamento | segundos |
| 10075 | Status da conexão do veículo | 0/1/2 |
| 10076 | Modo de início da sessão | 0 a 7 |

**Histórico e faturamento:**

| Registrador | Descrição | Unidade |
|---|---|---|
| 10065 | Energia histórica acumulada | kWh |
| 10103 | Energia de origem fotovoltaica | kWh |
| 10105 | Energia comprada da rede | kWh |
| 10158 a 10160 | Horário de início da sessão | ano/mês, dia/hora, min/seg |
| 10162 a 10164 | Horário de fim da sessão | ano/mês, dia/hora, min/seg |
| 10170 | Leitura do medidor MID antes da sessão | kWh (ganho 100) |
| 10172 | Leitura do medidor MID após a sessão | kWh (ganho 100) |

**Controle remoto:**

| Registrador | Descrição |
|---|---|
| 10060 | Liga/desliga recarga (1=desliga, 2=liga) |
| 10019 | Plug & Charge (0=desativado, 1=ativado) |
| 10500 a 10521 | Gerenciamento de cartões RFID |
 
## Exploração do portal SEMS+ e dados disponíveis

O SEMS+ (semsplus.goodwe.com) é a plataforma de nuvem da GoodWe onde ficam os dados dos inversores e carregadores. A equipe tem acesso via login compartilhado à planta **LAB FIAP Eco Smart Home**, que é o ambiente de laboratório disponibilizado pela FIAP para o projeto. Não há acesso a uma API programática: a exploração é feita diretamente no portal.

**Infraestrutura disponível na planta LAB FIAP:**

| Dispositivo | Modelo | SN | Status |
|---|---|---|---|
| EV Charger | Carregador EV | 57000HPA247L0002 | Ocioso |
| ES LD | Inversor de armazenamento (7,5 kW) | 97500NAP25BL0008 | Offline |
| 53600ERN238W0001 | Inversor de armazenamento (3,6 kW) | 53600ERN238W0001 | Offline |
| Dongle 14 / Dongle 16 | Dongles de comunicação | — | Offline |
| Third-party Inverter 1 | Inversor de terceiros | VD1009097500NAP25BL0008 | Offline |

**Dados disponíveis no painel da planta:**

O portal expõe um dashboard em tempo real com geração de energia (kWh), energia comprada da rede, energia injetada, receita de rede e consumo da carga. O fluxo de energia é representado graficamente com fontes (Solar, Bateria, Rede, Gerador) e destinos (Bateria, Uso da carga, Rede), além de curvas de potência ao longo do dia.

**Dados disponíveis por sessão de carregamento (Registo de carregamento do EV Charger):**

| Campo | Exemplo |
|---|---|
| Horário de início | 19/06/2026 19:27:28 |
| Horário de fim | 20/06/2026 00:22:24 |
| Duração | 4 Horas 56 Minutos |
| Energia carregada | 11,49 kWh |
| Autonomia estimada | 57,45 km |
| ID do cartão RFID utilizado | 57000HPA247L0002 |
| Porta de carregamento | 1 |

Esses dados são o que o EV ChargeOps precisará ler para calcular o rateio por usuário. Na planta de laboratório já há sessões registradas, incluindo uma sessão de 37 minutos com 0,00 kWh, que representa um caso de sessão iniciada mas sem carga efetiva, caso relevante para o modelo de rateio.

Durante a pesquisa da Sprint 1, foram identificadas e documentadas as APIs disponíveis para integração programática com a plataforma SEMS.

**GoodWe Open API (oficial — openapi.goodwe.com)**

A GoodWe disponibiliza uma API REST oficial, documentada em `openapi.goodwe.com`, com quatro grupos de endpoints (todos via `POST`):

| Grupo | Endpoints |
|---|---|
| Basic Information Query | Query Station Information, Query Device List Under Station, Query Device Attribute Information |
| Running Data Monitoring | Query Device Real-time Telemetry Data, Query Station Real-time Telemetry Data, Query Device Statistics Data, Query Station Statistics Data |
| Remote Dispatch Management | Create Control Task, Query Control Result, Create Control Parameter Read Task, Query Control Parameter Read Result |
| Alarm Detection Management | Query Device Alarms |

O carregador EV é suportado como `deviceType: 5`. Os campos de telemetria disponíveis para o EV Charger são:

| Campo | Tipo | Descrição | Unidade |
|---|---|---|---|
| vehConnectStatus | integer | Status da conexão: 0=desconectado, 1=plugado sem carga, 2=plugado e carregando | — |
| currentChargeE | double | Energia carregada na sessão atual | kWh |
| currentChargeTime | integer | Duração da sessão atual | s |
| evChargerCharge | double | Energia acumulada total do carregador | kWh |
| activePower | double | Potência de carregamento em tempo real | kW |
| voltage1/2/3 | double | Tensão por fase | V |
| current1/2/3 | double | Corrente por fase | A |

O campo `currentChargeE` é o dado central para o cálculo de rateio por sessão. O servidor para o Brasil é o International Server: `https://hk-gateway.semsportal.com`. O acesso à Open API requer conta organizacional no SEMS e deve ser solicitado à equipe GoodWe (academy@goodwe.com).

**API não oficial do SEMS Portal (comunidade)**

Paralelamente, a API não documentada do `semsportal.com` é real e ativamente utilizada — a integração open source [goodwe-sems-home-assistant](https://github.com/timsoethout/goodwe-sems-home-assistant) consome essa API via `POST /api/v1/Common/CrossLogin` para autenticação e `POST /api/v1/PowerStation/GetMonitorDetailByPowerstationId` para dados da planta. Suporta conta visitante read-only.

## APIs complementares

**OCPP (Open Charge Point Protocol)**

É o protocolo padrão de comunicação entre carregadores e sistemas de gestão. O OCPP 1.6 cobre autenticação, controle de sessão e relatórios; o 2.0.1 adiciona smart charging e segurança TLS. É o protocolo mais adotado no mercado e será avaliado como camada de integração do EV ChargeOps com sistemas de gestão externos.

**OCPI (Open Charge Point Interface v2.2.1)**

Define como diferentes operadores de rede trocam informações entre si: disponibilidade de pontos, autorização de sessões entre redes e registros de sessão (CDR) para faturamento cruzado. Relevante para cenários onde o condomínio queira abrir o ponto de recarga para visitantes de outras redes.

**Open Charge Map API**

Base de dados comunitária com mais de 55.000 pontos de recarga no mundo. Usada para enriquecer o mapa de pontos disponíveis na região e identificar oportunidades de expansão.
```
GET https://api.openchargemap.io/v3/poi/?output=json&countrycode=BR&maxresults=100
```

**Google Places API (evChargeOptions)**

Retorna informações sobre carregadores cadastrados em estabelecimentos no Google Maps: número de conectores, tipos e potência máxima disponível.
```
GET https://places.googleapis.com/v1/places/{place_id}
Headers: { "X-Goog-FieldMask": "evChargeOptions" }
```

---

# Frente 3: Arquitetura e Inteligência

Aprofundamentos escolhidos: B (papel da IA na solução) e C (esquema do banco de dados)

## Diagrama de Arquitetura

O EV ChargeOps adota uma estrutura em camadas bem definidas, garantindo o desacoplamento de responsabilidades desde o hardware em campo até as interfaces de ponta, permitindo manutenção simplificada e robustez no processamento de sessões.

<img width="1172" height="779" alt="image" src="https://github.com/user-attachments/assets/c2c4c7be-ff59-45a2-9e03-d901ef2ead6d" />

## Estrutura e Camadas do Sistema

* **Camada Física (Edge Device):** Centrada no carregador **GoodWe HCA G2**, responsável pela medição de energia (kWh), controle de início/fim de sessão, leitura do status de recarga e autenticação local via cartões RFID.
* **Camada de Comunicação:** Transmissão dos dados do carregador para o ecossistema backend utilizando o protocolo **Modbus TCP** estruturado sobre a infraestrutura de rede local (Wi-Fi ou LAN/Ethernet).
* **Camada de Backend (Core System):** O coração da plataforma, operando como o orquestrador que consome as transmissões Modbus. É responsável pelo recebimento de sessões de recarga, identificação de usuários por RFID, processamento do consumo energético, aplicação das regras do motor de rateio, integração direta com os modelos de IA e exposição de APIs REST para o frontend.
* **Banco de Dados:** Centralizado em tecnologia relacional **PostgreSQL**, garantindo consistência ácida (ACID) e integridade referencial para as entidades fundamentais do sistema: *Usuários, Veículos, Carregadores, Sessões de recarga e Faturas*.
* **Camada de Frontend:** Interfaces segregadas por perfil de acesso:
  * **Usuário:** Aplicativo/Portal focado em histórico de recargas, consumo mensal detalhado e visualização transparente de faturas.
  * **Administrador:** Dashboard de gestão contendo visão geral do sistema, monitoramento em tempo real das sessões ativas, relatórios de consumo total do condomínio e central de alertas emitidos pela IA.
* **Integrações Externas (Secundárias):** Sincronização via **SEMS+ API (GoodWe)** para consolidação de dados de nuvem do fabricante e **Open Charge Map** para contextualização geográfica opcional.

---

## Modelo de Rateio (Motor de Rateio)

O motor de rateio é o módulo responsável pelo cálculo de cobrança individualizada, estruturado para incentivar o uso eficiente da infraestrutura compartilhada e mitigar conflitos em vagas comuns (como o bloqueio de carregadores por veículos já carregados).

O cálculo do custo total da sessão ($C_{total}$) baseia-se na seguinte formulação:

$$C_{total} = (E_{consumida} \times T_{energetica}) + C_{ociosidade}$$

Onde:
* **$E_{consumida}$:** Energia efetivamente consumida na sessão (em kWh), extraída dos registradores do HCA G2.
* **$T_{energetica}$:** Tarifa de energia aplicada (R$/kWh), configurada conforme a concessionária local ou acordos do condomínio.
* **$C_{ociosidade}$:** Taxa adicional aplicada caso o veículo permaneça conectado à vaga após atingir 100% da carga. É calculada pelo tempo de ociosidade ($t_{ocioso}$) multiplicado por uma taxa de penalidade horária ($T_{ociosa}$):

$$C_{ociosidade} = t_{ocioso} \times T_{ociosa}$$

---

## Papel da IA na Solução

A inteligência do sistema opera na camada analítica por meio de dois componentes focados em otimização operacional e segurança:

* **IA 1 — Previsão de Pico de Demanda:** Utilizando técnicas de regressão linear ou análise de média temporal móvel, este modelo estuda o histórico de sessões do condomínio para prever os horários de maior criticidade no consumo elétrico. O resultado subsidia diretamente o gerenciamento de carga, sugerindo ou automatizando o controle para evitar sobrecargas no quadro geral.
* **IA 2 — Detecção de Anomalias:** Baseada em regras heurísticas de consumo ou algoritmos de isolamento (*Isolation Forest*), atua na varredura de comportamento para identificar padrões de consumo irregular (desvios abruptos de corrente/tensão), detectar falhas mecânicas/elétricas do hardware, mitigar uso indevido (fraudes de autenticação) e sinalizar veículos ociosos travando a vaga.

---

## Plano para a Sprint 02

* **Hardware & Comunicação:** Estruturação do ambiente de testes Modbus TCP para leitura simulada dos registradores do GoodWe HCA G2.
* **Backend & Banco de Dados:** Modelagem física e criação do schema do banco PostgreSQL contendo as tabelas do Core System. Desenvolvimento dos endpoints REST iniciais para ingestão de sessões.
* **Inteligência Artificial:** Desenvolvimento do protótipo da IA 1 (Média Temporal) em Python para modelagem preditiva de carga baseada em dados históricos sintéticos.
* **Frontend:** Criação das telas iniciais do fluxo do usuário (histórico e faturas) utilizando o framework escolhido para a interface.

---

## Links do Projeto

Utilize os acessos abaixo para visualizar as especificações técnicas, designs e o gerenciamento do projeto:

* **Diagrama da Arquitetura:** https://excalidraw.com/#json=hAJkKcG6Jo_vsugTp-sp8,aXthfg0HEz-wcBrLT4KSFw
* **Documentação da Arquitetura:** https://docs.google.com/document/d/1XkIU3JqrH9_mNOLK5a058u5zLWTZaQYw4Zvs8c6QDMQ/edit?usp=sharing
* **Quadro de Tarefas (Gerenciamento):** https://app.notion.com/p/Quadro-T-cnico-Sistema-de-Recarga-GoodWe-10a027a3111743848737baff63a2f214?source=copy_link
* **Protótipo de Interface (Figma):** https://www.figma.com/make/B0gsqv8mz5QLgzeufeLwgn/EV-Charging-Management-Dashboard?t=qHdKOFcLpgRUOViv-20&fullscreen=1

# Fontes Consultadas
 
- ABVE. Frota de eletrificados no Brasil se aproxima dos 650 mil veículos. Disponível em: https://smabc.org.br/frota-de-eletrificados-no-brasil-se-aproxima-dos-650-mil-veiculos/
- ABVE. Infraestrutura de recarga avança e já está em 25% dos municípios brasileiros. Disponível em: https://abve.org.br/infraestrutura-de-recarga-avanca-e-ja-esta-em-25-dos-municipios-brasileiros/
- O Tempo. Brasil alcança 25 mil pontos de recarga para carros elétricos. Disponível em: https://www.otempo.com.br/autotempo/2026/6/17/brasil-chega-a-25-mil-pontos-de-recarga-para-carros-eletricos
- ANEEL. Resolução Normativa nº 1.000, de 07/12/2021. Disponível em: https://www2.aneel.gov.br/cedoc/ren20211000.html
- ANEEL. Veículos Elétricos. Disponível em: https://www.gov.br/aneel/pt-br/assuntos/veiculos-eletricos
- VoltBras. Legislação Brasileira sobre eletropostos. Disponível em: https://voltbras.com/normas-tecnicas-e-legislacao/legislacao-brasileira-sobre-eletropostos-o-que-saber-antes-de-investir/
- GoodWe. GW_HCA-G2 Datasheet (PT). Documento técnico oficial, 2024.
- GoodWe. GW_HCA-G2 Manual do Usuário (PT). Documento técnico oficial, 2024.
- GoodWe. Mapa MODBUS HCA G2. Documento técnico oficial, 2024.
- GoodWe. SEMS+ Portal. Disponível em: https://semsplus.goodwe.com
- GoodWe. SEMS Plus 2 APP User Manual v1.0. Documento técnico oficial, novembro de 2025.
- GoodWe. API Introduction (SA-E-20221031-001). Documento técnico oficial, 2022.
- GoodWe. Developer Platform (OpenAPI). Disponível em: https://openapi.goodwe.com
- Karunanayake, B. Accessing the GoodWe SEMS Portal API: A Comprehensive Guide. Medium, 2023. Disponível em: https://binodmx.medium.com/accessing-the-goodwe-sems-portal-api-a-comprehensive-guide-296e0431c285
- Soethout, T. goodwe-sems-home-assistant: integração open source da API SEMS com Home Assistant. GitHub, 2026. Disponível em: https://github.com/timsoethout/goodwe-sems-home-assistant
- NeoCharge. Carregador para carro elétrico em prédios e condomínios. Disponível em: https://www.neocharge.com.br/tudo-sobre/carregador-carro-eletrico-predio-condominio-instalacao
- Open Charge Alliance. Open Charge Point Protocol (OCPP). Disponível em: https://openchargealliance.org/protocols/open-charge-point-protocol/
- Virta Global. OCPI protocol explained. Disponível em: https://www.virta.global/blog/ocpi-protocol-explained-the-backbone-of-ev-charging-interoperability
- Open Charge Map. API Documentation. Disponível em: https://api.openchargemap.io/v3
- Google. Places API, evChargeOptions field. Disponível em: https://developers.google.com/maps/documentation/places/web-service/reference/rest/v1/places