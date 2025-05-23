# Ponderada de Arquitetura SOA e Requisitos Não Funcionais
## 1 - Construa uma arquitetura SOA simples utilizando diagrama de componentes que leve em conta os blocos principais do sistema assim como a integração com os componentes de serviços da arquitetura SOA - Justifique os pontos da sua arquitetura.

### Resposta:
A seguir, está a arquitetura SOA que pensei para o sistema de reservas de passagens aéreas. As linhas tracejadas representam comunicação com APIs externas ao sistema, como a API de geolocalização e a API de pagamentos (Stripe). As linhas contínuas representam comunicação entre os componentes do sistema.
Fiz uma tabela com a descrição de cada componente, seu papel principal no domínio e a justificativa de importância no contexto do sistema.
<div align="center">
<img src="./Arquitetura SOA - Semana 04.png" alt="Arquitetura SOA" /> <br>
<sup>Fonte: Material produzido pelos autores (2025).</sup>
</div>


| #  | Componente                          | Papel principal no domínio                                                                                                          | Justificativa de design                                                                                                |
| -- | ----------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| 1  | **Cliente (Web/Mobile)**            | Interface ou aplicativo que permite pesquisar, reservar e pagar.                                                         | Mantém lógica de apresentação isolada; qualquer canal novo (quiosque, chatbot) reutiliza as mesmas APIs.               |
| 2  | **API Gateway**                     | Termina TLS, valida JWT com o Auth Service, aplica *rate-limit*, faz cache leve e roteia para o micro-serviço alvo.                 | Ponto único de entrada → simplifica configuração de CORS, logging e políticas de segurança; esconde topologia interna. |
| 3  | **Auth Service + DB**               | Registro, login social e emissão/validação de tokens JWT.                                                                           | Centraliza segurança e LGPD; demais serviços apenas conferem o token → mantém‐se *stateless*.                          |
| 4  | **Flight Service + DB**             | Detém inventário de voos, horários, assentos e regras de tarifa / classe.                                                           | Otimizado para leitura (picos de busca); atualização limitada a Booking/Admin evita contenção de escrita.              |
| 5  | **Booking Service + DB**            | “Orquestrador” da reserva: bloqueia assentos no Flight Service, inicia pagamento e muda estado (pendente → confirmada / cancelada). | Implementa padrão *Saga*; desacoplado de UI e de processamento de pagamento.                                           |
| 6  | **Payment Service**                 | Converte ordem interna em cobrança via **Stripe API** (cartão, Pix, wallet) e consome *web-hooks* de sucesso/estorno.               | Isola escopo PCI-DSS; trocar adquirente exige mexer somente aqui.                                                      |
| 7  | **Notification Service + DB**       | Enfileira e envia e-mails/SMS (via SendGrid, SES etc.) de confirmação ou alerta de falha.                                           | Comunicação assíncrona evita que erros de e-mail derrubem reservas; permite reenvio e auditoria.                       |
| 8  | **Admin Service**                   | Interface interna para funcionários gerenciarem voos, preços e disponibilidade de assentos (operações CRUD).                        | Separa domínios de usuário final e back-office, aplicando políticas de autorização mais restritivas.                   |
| 9  | **API de Geolocalização** (externa) | Traduz IP/GPS em cidade de origem sugerida ao usuário.                                                                              | “Pay as you go” evita manter base de dados IP↔localidade e melhora UX sem custo fixo.                                  |
| 10 | **Stripe API** (externa)            | Processa pagamentos, devolve tokens antifraude, regula chargebacks.                                                                 | Reduz time-to-market e garante compliance financeiro.                                                                  |

---
## 2 - Liste pelo menos 5 requisitos não funcionais deste sistema justificando a necessidade deste requisito assim como a sua relação ou necessidade dentro da arquitetura SOA

### Resposta:
Abaixo está uma tabela com 5 requisitos não funcionais (NFRs) que são importantes pro sistema de reservas de passagens aéreas, pensando em problemas que podem acontecer e como a arquitetura ajuda a evitar esses problemas.


| NFR                                                                      | Por que importa                                                                      | Como a arquitetura ajuda                                                                                                                                            |
| ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **1. Alta disponibilidade (99,9 % ou mais)**                             | Site de companhia aérea não pode cair: cada minuto parado significa vendas perdidas. | Os serviços rodam em **cluster** (várias cópias). Se um nó falha, outro assume. Dados de reserva e pagamento têm réplica em tempo real.                             |
| **2. Escalabilidade elástica**                                           | Promoções, férias e outras épocas do ano geram picos de acesso.                                           | Como cada parte (Flight, Booking, etc.) é um **micro-serviço**, dá para subir mais contêineres só do que está sobrecarregado, sem mexer no resto. Assim, eles conseguem dar conta da sobrecarga.                  |
| **3. Desempenho de resposta (até 2 s na busca, 5 s para fechar compra)** | Páginas lentas impactam muito negativamente na experiência dos usuários.                          | O **API Gateway** guarda respostas rápidas em cache; os serviços conversam por HTTP direto, sem etapas extras, e o fluxo de reserva é curto.                        |
| **4. Segurança e conformidade (LGPD e outras legislações)**                          | Lidamos com cartão e dados pessoais; multas e reputação em jogo.                     | **Auth Service** gera **JWT** assinado; só o **Payment Service** toca dados de cartão e fala com a Stripe (Ou outra API de pagamento); todo o tráfego é HTTPS e os bancos ficam criptografados. |
| **5. Observabilidade (logs, métricas, tracing)**                         | Precisamos achar erros rápido e provar que o SLA está ok.                            | Logs e métricas de todos os serviços ficam salvas em seus respectivos bancos; o **API Gateway** agrega tudo e manda para um sistema de monitoramento (ex: ELK, Prometheus). Assim, dá pra ver o que está acontecendo em tempo real. |