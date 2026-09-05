# 02 · Análise regulatória

> **Aviso.** Este documento é análise técnica para orientar decisões de arquitetura e de
> *roadmap*. Não é aconselhamento jurídico. Todos os pontos marcados com ⚖️ devem ser
> confirmados com o Contabilista Certificado responsável e com apoio jurídico antes do
> lançamento.

---

## 1. Exercício da profissão — Ordem dos Contabilistas Certificados

O ponto de partida é que **a contabilidade não pode ser prestada por software**. Compete ao
Contabilista Certificado planear, organizar e coordenar a execução da contabilidade, assumir
a responsabilidade pela regularidade técnica nas áreas contabilística e fiscal, e **assinar as
declarações fiscais** das entidades a seu cargo. A Cifra é, juridicamente, um serviço prestado
sob a responsabilidade do CC, com o software como instrumento.

Consequências diretas para o sistema:

- **Nenhuma submissão automática à AT.** Não deve existir qualquer caminho de código que
  submeta uma declaração sem uma ação humana explícita e registada de um utilizador com
  perfil CC. Isto é um invariante de segurança, verificável por teste.
- **Rastreabilidade da assinatura.** Para cada declaração é preciso saber *quem* aprovou,
  *quando*, *com que dados* e *que versão* do lançamento estava em vigor. Ver
  [04 — auditoria imutável](04-modelo-de-dados.md#6-auditoria-imutável).
- **Contrato de prestação de serviços por cliente**, associado ao CC responsável, com
  registo da aceitação e da eventual cessação. Modelar como entidade de primeira classe.
- ⚖️ **Seguro de responsabilidade civil profissional** e regras da Ordem sobre publicidade
  e sobre a forma societária do prestador — confirmar antes de comunicar comercialmente.
- ⚖️ **Comunicação da carteira à OCC** e regras de transição de cliente entre CCs (a FAQ
  "já tenho contabilista" descreve um processo que tem enquadramento próprio).

**Risco estrutural:** a página nomeia **um** CC. Um único CC responsável é um ponto único
de falha regulatório — se ficar indisponível, nada pode ser assinado nem entregue. Deve
existir um CC suplente contratualizado desde o lançamento e um segundo CC efetivo a partir
de ~250 clientes.

---

## 2. Autoridade Tributária e e-Fatura

### 2.1 O problema central: não há API pública de leitura

Existe *webservice* da AT para **comunicação de faturas emitidas** (SOAP, com certificado
SSL de cliente emitido pela AT e credenciais de sub-utilizador) e para **comunicação de
séries / ATCUD**. Não existe API pública equivalente para **ler as faturas de compra** de
um contribuinte a partir de um terceiro.

Existem três caminhos possíveis, e a escolha condiciona todo o *onboarding*:

| Caminho | Como funciona | Avaliação |
|---|---|---|
| **A — Credenciais do cliente** | O cliente entrega a senha do Portal das Finanças; a plataforma inicia sessão em nome dele | ❌ **Rejeitar.** Guardar credenciais pessoais de acesso a um serviço público é indefensável perante o RGPD (art. 32.º) e perante o cliente. Frágil a mudanças de portal e a MFA. |
| **B — Sub-utilizador delegado ao CC** | O cliente cria, na *Gestão de Utilizadores* do Portal das Finanças, um utilizador com permissões limitadas (consulta/recolha de faturas), atribuído ao contexto da Cifra | ✅ **Recomendado.** É o mecanismo que a própria AT prevê para a relação com o contabilista. Permissões mínimas, revogáveis pelo cliente a qualquer momento, sem partilha da senha pessoal. |
| **C — Carregamento manual / SAF-T** | O cliente ou o CC exporta o ficheiro de faturas ou o SAF-T e carrega na plataforma | ✅ **Obrigatório como alternativa.** Tem de existir sempre, como plano B quando a delegação falha. |

**Decisão de arquitetura:** implementar **B como principal e C como alternativa permanente**,
e desenhar a camada de ingestão de forma a que a origem do documento (`origem`:
`efatura` \| `upload` \| `email` \| `saft` \| `banco`) seja um detalhe da fonte e não afete
o resto do sistema. Se a AT publicar no futuro uma API adequada, entra como mais uma fonte.

O passo de delegação no Portal das Finanças é o momento de maior atrito de todo o
*onboarding*. Deve ter guião passo a passo com capturas de ecrã, verificação automática de
que a delegação ficou ativa, e a possibilidade de o CC fazer a chamada telefónica de apoio.

### 2.2 Obrigações a modelar no motor de prazos

O motor de obrigações é o coração da promessa "a Cifra nunca deixa passar". Não deve ter
datas escritas no código: cada obrigação é uma **regra com vigência temporal**, aplicável
condicionalmente ao enquadramento do cliente (regime de IVA, periodicidade, regime de IRS,
existência de retenções, existência de trabalhadores).

| Obrigação | Periodicidade | Aplicável a |
|---|---|---|
| Emissão de faturas-recibo (recibos verdes) | Mensal | Todos os independentes |
| Validação de faturas no e-Fatura | Mensal | Todos |
| Declaração periódica de IVA + pagamento | Trimestral / mensal | Regime normal |
| Entrega de retenções na fonte | Mensal | Quem retém |
| **Declaração trimestral à Segurança Social** | Trimestral (jan/abr/jul/out) | Independentes com obrigação contributiva |
| Modelo 10 / DMR | Anual / mensal | Quem paga rendimentos sujeitos a retenção |
| Modelo 3 IRS + Anexo B/C | Anual (abr–jun) | Todos |
| IES / Declaração Anual | Anual | Contabilidade organizada |
| Modelo 30 | Mensal | Pagamentos a não residentes |
| Notificações e citações da AT | Evento | Todos |

⚖️ As datas concretas variam com o ano e com alterações legislativas. O calendário deve ser
**mantido pelo Luís e validado pelo Carlos** através de um ecrã de administração, com
histórico de versões — nunca por *deploy* de código.

### 2.3 Faturação certificada — não construir

Emitir faturas em Portugal exige software certificado pela AT (Decreto-Lei n.º 28/2019,
regras de certificação da AT, ATCUD e código QR nos termos da Portaria n.º 195/2020).
Obter certificação implica requisitos técnicos rígidos (assinatura/encadeamento de
documentos, inalterabilidade, SAF-T conforme, exportação, controlo de versões), submissão
de pedido à AT e manutenção continuada.

**Recomendação:** não incluir no âmbito. Integrar por API com um emissor já certificado
(*white-label* ou parceria comercial). A promessa da página — "continuas a faturar como
hoje; a Cifra lê as tuas faturas" — já é a posição correta e deve ser a mensagem principal.

---

## 3. Acesso a dados bancários — PSD2

Ler movimentos de conta bancária é um serviço de informação sobre contas (AIS), atividade
regulada e sujeita a autorização do Banco de Portugal. A Cifra **não deve** procurar
licença própria nesta fase.

**Recomendação:** contratar um agregador AISP licenciado com cobertura da banca portuguesa
(Tink, GoCardless Bank Account Data, Salt Edge ou equivalente), avaliando: cobertura real
das instituições relevantes no Norte (CGD, Millennium, Santander, Novobanco, CTT, Crédito
Agrícola, Montepio), qualidade de categorização, preço por conta ligada, e DPA/subcontratação.

Implicações de produto que devem estar no desenho desde o início:

- O consentimento de acesso tem **validade limitada** e exige renovação com autenticação
  forte. É preciso um fluxo de re-consentimento proativo, com aviso antes da expiração,
  ou a conciliação bancária "silenciosamente" deixa de funcionar.
- Os dados bancários são dados pessoais especialmente reveladores. Devem ter **base legal,
  âmbito e retenção próprios**, e ser opcionais — a Cifra tem de funcionar sem eles.
- A conciliação bancária é **sugestão**, nunca verdade contabilística: o extrato não é
  documento de suporte.

---

## 4. Prevenção do branqueamento de capitais (Lei n.º 83/2017)

Este é o requisito mais frequentemente esquecido em produtos deste tipo. **Os contabilistas
certificados são entidades obrigadas** ao abrigo da Lei n.º 83/2017. Os deveres preventivos
(controlo, identificação e diligência, abstenção, recusa, conservação, exame, colaboração,
não divulgação e formação) aplicam-se à relação com cada cliente.

Isto tem tradução direta em funcionalidades:

| Dever | Funcionalidade |
|---|---|
| Identificação e diligência | Recolha e verificação de identidade no *onboarding*: documento de identificação, NIF, morada, atividade/CAE, e identificação do beneficiário efetivo quando aplicável |
| Recusa / abstenção | O *onboarding* tem de poder ser **bloqueado** por decisão do CC, com registo do motivo |
| Conservação | Retenção dos elementos de identificação por **7 anos** após o fim da relação |
| Exame | Sinalização de operações atípicas face ao perfil declarado do cliente |
| Formação | Registo de formação periódica da equipa |
| Não divulgação | Segregação de acessos: comunicações a autoridades não visíveis para o cliente nem para operadores de 1.ª linha |

⚖️ O detalhe do procedimento e os limiares aplicáveis devem ser definidos pelo CC responsável.
A conclusão para a arquitetura é firme: **KYC é parte do produto e do modelo de dados, não
um passo manual fora do sistema.**

---

## 5. RGPD

### 5.1 Enquadramento

A Cifra é **responsável pelo tratamento** relativamente aos dados dos seus clientes
(subscritores) e, na prática operacional, trata dados de terceiros que constam das faturas
(fornecedores, e por vezes clientes finais do subscritor). Os fornecedores de nuvem, o
agregador bancário, o fornecedor de OCR e o fornecedor de modelo de IA são
**subcontratantes** e exigem contrato nos termos do art. 28.º.

### 5.2 Bases legais por finalidade

| Finalidade | Base legal | Notas |
|---|---|---|
| Prestação do serviço de contabilidade | Execução de contrato (art. 6.º/1/b) | |
| Conservação de documentos contabilísticos e fiscais | Obrigação jurídica (art. 6.º/1/c) | Prazos fiscais sobrepõem-se ao pedido de apagamento |
| Elementos de KYC/AML | Obrigação jurídica (art. 6.º/1/c) | Retenção de 7 anos |
| Lista de espera e comunicações de pré-lançamento | Consentimento (art. 6.º/1/a) | Tem de ser granular e revogável |
| Segurança, registos de auditoria, deteção de fraude | Interesse legítimo (art. 6.º/1/f) | Exige teste de ponderação documentado |
| Melhoria do assistente com dados reais | ⚠️ **Não assumir consentimento tácito** | Ver 5.4 |

### 5.3 Retenção — o conflito a resolver explicitamente

A obrigação fiscal de conservação de documentos e registos (na ordem dos **10 anos**) colide
com o princípio da minimização e com o direito ao apagamento. Resolve-se com uma **política
de retenção por categoria**, implementada como configuração e não como código:

| Categoria | Retenção | Ação no fim |
|---|---|---|
| Documentos e lançamentos contabilísticos | 10 anos (⚖️ confirmar por tipo de imposto) | Apagamento programado |
| Elementos de KYC/AML | 7 anos após fim da relação | Apagamento programado |
| Conversas com o assistente | 24 meses | Anonimização (remoção de identificadores) |
| *Prompts* e respostas do modelo (auditoria de IA) | 24 meses | Anonimização |
| Registos de auditoria de segurança | 12–24 meses | Apagamento |
| Dados bancários agregados | 12 meses após revogação do consentimento | Apagamento |
| Lista de espera / marketing | Até revogação, máx. 24 meses de inatividade | Apagamento |

Um pedido de apagamento executa **apagamento parcial**: sai o que não está coberto por
obrigação legal, permanece o que está, e o cliente recebe explicação clara de porquê.
Este comportamento deve ser implementado e testado, não improvisado no primeiro pedido.

### 5.4 Avaliação de impacto (DPIA) — obrigatória

O tratamento combina: dados financeiros em larga escala, avaliação/classificação apoiada
por tecnologia inovadora (IA), agregação de dados de fontes distintas (AT, banca, cliente)
e tratamento sistemático. Face aos critérios do art. 35.º e às orientações aplicáveis,
**deve considerar-se que a DPIA é obrigatória** e realizar-se **antes** do início do
tratamento, não depois do lançamento.

Elementos mínimos da DPIA: descrição sistemática das operações, avaliação da necessidade e
proporcionalidade, riscos para os titulares (uso indevido, decisão errada com consequência
fiscal, fuga de dados financeiros), e medidas de mitigação — incluindo explicitamente a
**revisão humana obrigatória** como controlo de risco.

### 5.5 Outros pontos operacionais

- **DPO**: ⚖️ provavelmente não obrigatório face à dimensão e à natureza do tratamento, mas
  **recomendado** designar um responsável interno pela proteção de dados e registá-lo.
- **Registo de atividades de tratamento (RoPA)**: obrigatório na prática, dada a natureza
  não ocasional e o volume de dados. Manter em repositório versionado.
- **Localização dos dados**: a página promete alojamento na União Europeia. Isto tem de
  valer **para toda a cadeia**, incluindo o fornecedor de modelo de IA, OCR, envio de email,
  observabilidade e cópias de segurança. É o ponto onde estas promessas mais falham na prática.
- **Transparência**: informar quando os dados são processados por IA, com que finalidade e
  com que salvaguardas.
- **Direitos dos titulares**: portabilidade em formato aberto (o cliente tem de conseguir
  sair com tudo), acesso, retificação e oposição, com prazos de resposta instrumentados.
- **Notificação de violação de dados**: 72 h à CNPD, com comunicação aos titulares quando
  houver risco elevado.

---

## 6. Cibersegurança — NIS2 e Decreto-Lei n.º 125/2025

### 6.1 Estado da transposição

A Diretiva (UE) 2022/2555 (NIS2) foi transposta em Portugal pelo **Decreto-Lei n.º 125/2025**,
publicado a 4 de dezembro de 2025, que aprova o novo **Regime Jurídico da Cibersegurança**,
em vigor desde abril de 2026. O **Regulamento n.º 756/2026, de 22 de junho**, do CNCS,
concretiza o regime — nomeadamente a plataforma eletrónica nacional, os níveis de
conformidade e as medidas mínimas de segurança exigidas.

### 6.2 A Cifra está abrangida?

Análise: o regime aplica-se a entidades **essenciais** e **importantes** de setores
identificados, com um critério de dimensão que, em regra, exclui micro e pequenas empresas
salvo exceções expressas. A prestação de serviços de contabilidade **não consta** dos
setores de alta criticidade nem dos restantes setores críticos da diretiva. A Cifra, como
microempresa prestadora de serviços de contabilidade com um portal próprio, **muito
provavelmente não é entidade essencial nem importante** e não terá obrigação de registo
junto do CNCS.

⚖️ Esta conclusão tem de ser confirmada juridicamente, com atenção a três pontos: (i) a
qualificação do serviço prestado, dado que a Cifra opera uma plataforma digital para
terceiros; (ii) a possibilidade de designação individual por decisão da autoridade
competente; (iii) a evolução da carteira — se vier a servir entidades abrangidas.

### 6.3 Porque se deve cumprir na mesma

O requisito foi expressamente colocado pelos fundadores, e há três razões substantivas
que o justificam independentemente da obrigação legal:

1. **Cadeia de fornecimento.** Entidades abrangidas são obrigadas a impor requisitos de
   segurança aos seus fornecedores. Qualquer cliente Empresa que seja entidade abrangida
   transmitirá esses requisitos à Cifra. Cumprir desde o início evita reengenharia.
2. **Crescimento.** Se a Cifra atingir os limiares de dimensão, o enquadramento pode mudar.
   Construir com os controlos já implementados custa uma fração de os acrescentar depois.
3. **Risco real.** Uma plataforma que concentra NIFs, faturas, extratos bancários e acesso
   delegado ao Portal das Finanças de centenas de contribuintes é um alvo com valor
   económico direto. O padrão NIS2 é, aqui, simplesmente o padrão adequado ao risco.

**Postura adotada:** conformidade material com as medidas de gestão de risco e com os
prazos de notificação de incidentes (alerta precoce em 24 h, notificação em 72 h, relatório
final em 1 mês), **sem** presumir a obrigação de registo. A implementação concreta e a
matriz de controlos estão em
[05 — Segurança e conformidade](05-seguranca-e-conformidade.md).

---

## 7. Regulamento de IA (AI Act)

A funcionalidade central da Cifra é um sistema de IA que interage diretamente com pessoas
singulares e produz orientação com consequências patrimoniais.

**Classificação de risco.** O aconselhamento fiscal a trabalhadores independentes não consta
das utilizações de alto risco do Anexo III (que abrange, por exemplo, avaliação de
solvabilidade e acesso a serviços essenciais). A avaliação preliminar é de **sistema de
risco limitado**, sujeito às obrigações de transparência. ⚖️ Confirmar juridicamente,
sobretudo se no futuro forem introduzidas funcionalidades de pontuação ou de decisão
automatizada sobre o cliente.

**Obrigações de transparência (art. 50.º), aplicáveis desde 2 de agosto de 2026:**

- O utilizador tem de ser informado, **na primeira interação**, de que está a interagir com
  um sistema de IA. A distinção visual entre "assistente" e "contabilista" que a página já
  faz passa a ser requisito, não estética — e deve ser coberta por teste automatizado de UI.
- Conteúdo gerado artificialmente deve ser identificável enquanto tal.

**Obrigações que decorrem do bom senso profissional e reforçam a conformidade:**

- Registo integral de *prompt*, contexto recuperado, resposta, versão do modelo e versão da
  base de conhecimento — sem isto é impossível responder a uma reclamação ou a uma inspeção.
- Política de utilização aceitável e limites explícitos ("orientação preliminar sujeita a
  validação profissional", que a página já contém e deve ser reproduzida em cada resposta).
- Supervisão humana efetiva, com poder real de correção — o que já é o modelo operacional.
- Documentação de avaliação de qualidade do assistente e registo de erros detetados.

---

## 8. Consumidor e contratação à distância

Os clientes-alvo, sendo profissionais, nem sempre são consumidores em sentido estrito, mas
o serviço é vendido à distância, por adesão, com preço mensal:

- ⚖️ **Direito de livre resolução** (14 dias) quando o adquirente for consumidor.
- **Termos e condições** claros quanto a âmbito, exclusões, política de uso justo, prazos
  de resposta e regras de cessação com devolução de documentos.
- **Livro de Reclamações eletrónico** e indicação da entidade de resolução alternativa de
  litígios competente.
- **Faturação certificada da própria Cifra** para cobrar as mensalidades — a Cifra tem de
  emitir as suas próprias faturas em software certificado.

---

## Fontes consultadas

- [CNCS — Diretiva NIS 2](https://www.cncs.gov.pt/pt/diretiva-nis-2/)
- [PwC Portugal — Decreto-Lei n.º 125/2025: Regulamento do Regime Jurídico de Cibersegurança](https://www.pwc.pt/pt/temas-actuais/dl-125-2025.html)
- [Morais Leitão — Novo Regime Jurídico de Cibersegurança (transposição da NIS2)](https://www.mlgts.pt/en/knowledge/legal-alerts/Legal-Alert-New-legal-framework-for-cybersecurity-Transposition-of-NIS2-Directive/26435/)
- [PLMJ — Transposição da Diretiva NIS 2: guia prático](https://www.plmj.com/xms/files/07_Guias_e_Manuais/2025/Transposicao-Diretiva_NIS_2-Ciberseguranca.pdf)
- [ECO — CNCS publica regulamento que concretiza o regime jurídico da cibersegurança](https://eco.sapo.pt/2026/06/22/cncs-publica-regulamento-que-concretiza-regime-juridico-da-ciberseguranca/)
- [OCC — Branqueamento de capitais e financiamento do terrorismo](https://portal.occ.pt/pt-pt/contabilista-certificado/branqueamento-de-capitais-e-financiamento-do-terrorismo)
- [Lei n.º 83/2017, de 18 de agosto](https://diariodarepublica.pt/dr/detalhe/lei/83-2017-108021178)
- [ASAE — Entidades obrigadas e deveres preventivos (BCFT)](https://www.asae.gov.pt/perguntas-frequentes1/prevencao-e-combate-ao-branqueamento-de-capitais-e-ao-financiamento-do-terrorismo/6-quem-sao-as-entidades-obrigadas-no-ambito-da-lei-n-832017-de-18-de-agosto.aspx)
- [Portal das Finanças — e-Fatura: webservice e multidocumento (FAQ)](https://info.portaldasfinancas.gov.pt/pt/apoio_contribuinte/questoes_frequentes/pages/faqs-00996.aspx)
- [OCC — Manual de integração de software: comunicação das faturas à AT](https://www.occ.pt/fotos/editor2/comunicacao_dados_faturas_2013_02_28.pdf)
- [EU AI Act — Transparency rules (art. 50.º)](https://artificialintelligenceact.eu/transparency-rules-article-50/)
- [Segurança Social — Declaração anual e declaração trimestral de trabalhadores independentes](https://www.seg-social.pt/noticias/-/asset_publisher/kBZtOMZgstp3/content/trabalhadores-independentes-entrega-da-declaracao-anual-e-declaracao-trimestr-3)
