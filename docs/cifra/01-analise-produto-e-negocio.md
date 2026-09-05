# 01 · Análise de produto e negócio

## 1. O que a Cifra está realmente a vender

A página comunica três coisas ao mesmo tempo, e é importante separá-las porque têm
custos marginais muito diferentes:

| Camada | Promessa | Custo marginal por cliente | Escala |
|---|---|---|---|
| **Software** | Recolha de faturas, classificação, alertas, portal | Quase nulo | Ilimitada |
| **Assistente** | Resposta a dúvidas fiscais 24/7 | Baixo mas real (inferência + curadoria) | Alta |
| **Serviço profissional** | Um CC valida, assina e responde em < 24 h | **Alto e humano** | Limitada pelo número de CCs |

O erro clássico neste segmento é fixar o preço como se fosse SaaS (39 €) e entregar
como se fosse consultoria (acesso ilimitado a um CC). A viabilidade depende de o assistente
**deflectir** a maior parte das interações — não de as encaminhar todas com uma camada
bonita por cima.

**Métrica-mestra do negócio:** *taxa de deflexão* — percentagem de interações resolvidas
sem tempo do CC. Deve ser instrumentada desde o primeiro dia do piloto e revista semanalmente.
Abaixo de ~70 % o modelo de preço não fecha.

## 2. Diferenciação face à concorrência

O mercado português já tem players relevantes: contabilistas tradicionais de proximidade,
plataformas de contabilidade online com CC incluído, e software de faturação com módulo
contabilístico. A vantagem defensável da Cifra **não é a IA** (será *commodity* em 18 meses),
é a combinação de:

1. **Proximidade geográfica declarada** (Viana, Braga, Barcelos, Porto) com reunião presencial
   incluída — algo que nenhuma plataforma nacional generalista oferece;
2. **Responsabilidade profissional nominal e visível** — a página mostra o rosto do CC que assina.
   Isto tem valor real num setor onde a confiança é o produto;
3. **Especialização no independente do Norte** — CAEs recorrentes (consultoria imobiliária,
   saúde, formação, design, construção civil em nome individual) permitem uma base de
   conhecimento fiscal **estreita e profunda** em vez de larga e superficial.

**Recomendação:** transformar (3) em ativo técnico. Um catálogo curado de 150–300 respostas
para os 20 CAEs mais frequentes, escritas e assinadas pelo Luís, é mais valioso e mais
seguro do que um assistente genérico sobre todo o Código do IVA.

## 3. Segmento e dimensionamento

O segmento declarado — trabalhador independente / ENI no Norte — é grande e mal servido,
mas é também o de **menor disponibilidade a pagar**. Notas de calibração:

- O limite de isenção do art. 53.º do CIVA é de **15 000 €** de volume de negócios anual
  (saída imediata do regime acima de 18 750 €). Um cliente **Start** típico fatura abaixo
  disso — 39 €/mês representa uma fração relevante do rendimento dele. A sensibilidade ao
  preço será alta e o *churn* por cessação de atividade também.
- O cliente **Pro** (69 €) tem IVA trimestral, mais faturas e mais valor a proteger: é onde
  está a margem e onde o produto é mais defensável. **O funil deve ser desenhado para o Pro**,
  com o Start a funcionar como porta de entrada.
- Micro-empresas (Empresa, 129 €) são um negócio diferente — processamento salarial e IRC
  implicam outro perfil de risco e outro produto. Bem colocado em 2027; não antecipar.

## 4. Revisão crítica da página de pré-lançamento

O que está bem feito e deve ser preservado no produto:

- A conversa de exemplo mostra **a IA a propor e o humano a confirmar e a corrigir**. É
  honesto, é o modelo operacional correto, e satisfaz por acaso a transparência exigida
  pelo AI Act. Manter como padrão de interface.
- O aviso "Pendente de validação pelo contabilista" é o elemento de confiança mais forte
  da página. Deve existir como **estado de máquina real** no sistema, não como decoração.
- A FAQ "E se o assistente se enganar?" antecipa a objeção principal. Manter, e ligar a
  uma política pública de responsabilidade.

Correções necessárias antes do lançamento:

| Problema | Porquê importa | Correção |
|---|---|---|
| O calendário omite a **declaração trimestral à Segurança Social** (jan/abr/jul/out) | É a obrigação que o independente mais falha; a promessa é precisamente "nunca deixar passar" | Acrescentar, e implementar no motor de obrigações |
| Omite **DMR/Modelo 10** e **IES** para ENI com contabilidade organizada | O plano Pro anuncia contabilidade organizada | Acrescentar ao calendário do Pro |
| "Resposta do contabilista < 24 h" sem qualificação | Compromisso ilimitado com custo humano | Qualificar: dias úteis, âmbito, política de uso justo |
| "Lê os movimentos da tua conta bancária" | Só é possível através de um AISP licenciado, com consentimento renovável | Reformular e explicar a renovação do consentimento |
| "Oferecemos faturação certificada integrada" | Certificação AT é um projeto próprio | Anunciar como parceria, ou remover até existir |
| Formulário de lista de espera | Recolhe nome, email, situação fiscal e zona — a situação fiscal aproxima-se de dado sensível de perfil | Consentimento explícito, política de privacidade ligada, minimização (a zona pode ser opcional) |
| "90 anos de experiência" | Soma de percursos; lê-se como exagero | Reformular para "três décadas cada" |

## 5. Economia unitária e capacidade

O modelo abaixo é uma **estimativa de trabalho** para orientar decisões, não uma projeção
financeira. Os pressupostos de tempo do CC devem ser substituídos por medições reais
durante o piloto.

### Pressupostos

| Variável | Start | Pro |
|---|---|---|
| Preço/mês | 39 € | 69 € |
| Receita anual | 468 € | 828 € |
| Documentos/ano | ~60 | ~400 |
| Tempo do CC/ano **sem** automação | ~6 h | ~18 h |
| Tempo do CC/ano **com** automação (alvo) | ~1,5 h | ~5 h |
| Custo carregado do CC | ~35 €/h | ~35 €/h |

### Margem de contribuição estimada (alvo, por cliente/ano)

| Rubrica | Start | Pro |
|---|---|---|
| Receita | 468 € | 828 € |
| Tempo do CC | −53 € | −175 € |
| Inferência de IA e OCR | −12 € | −30 € |
| Agregador bancário (PSD2) | — | −18 € |
| Infraestrutura e observabilidade | −9 € | −14 € |
| Comissões de pagamento (~1,5 %) | −7 € | −12 € |
| Suporte de 1.ª linha | −25 € | −40 € |
| **Margem de contribuição** | **~362 € (77 %)** | **~539 € (65 %)** |

A margem é saudável **desde que os alvos de automação sejam atingidos**. Sem automação,
o Start passa a ~200 €/ano de margem e o Pro fica perto do ponto crítico.

### Capacidade e ponto de equilíbrio

Um CC com carteira de regime simplificado, apoiado por revisão assistida, deve conseguir
**250–400 clientes Start** ou **80–120 clientes Pro** — a validar no piloto. Numa mistura
de 60 % Start / 40 % Pro:

- **500 clientes** ⇒ MRR ≈ 300 × 39 € + 200 × 69 € = **25 500 €/mês**
- Necessitam de aproximadamente **1,5 a 2 CC a tempo inteiro**
- Ponto de equilíbrio estimado (com 2 developers, 1 CC, 1 apoio e infraestrutura):
  **~250–320 clientes**, atingível em 12–18 meses a partir do lançamento se a lista de
  espera converter razoavelmente.

### Consequências de desenho que decorrem daqui

1. **Instrumentar o tempo do CC por lançamento e por resposta.** Sem esta métrica não é
   possível gerir o negócio. É requisito funcional da fila de revisão, não *analytics*.
2. **A fila de revisão é o produto interno mais importante.** Deve ordenar por risco fiscal
   e prazo, agrupar lançamentos semelhantes para aprovação em lote e aprender com as
   correções do CC.
3. **Limitar o âmbito das perguntas por plano.** O Start dá orientação; planeamento fiscal,
   inspeções, litígio e operações imobiliárias são serviço avulso faturado à parte.
4. **Reduzir o custo de aquisição por via da rede dos sócios.** Trinta anos de carteira no
   Norte valem mais do que qualquer campanha paga nesta fase. Os primeiros 50 aderentes
   devem sair daí, e servem simultaneamente de piloto.
