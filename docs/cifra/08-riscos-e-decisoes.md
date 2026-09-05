# 08 · Riscos e decisões em aberto

## 1. Registo de riscos

Probabilidade (P) e impacto (I) numa escala de 1 a 5. Ordenado por exposição (P × I).

| # | Risco | P | I | Exp. | Mitigação | Dono |
|---|---|---|---|---|---|---|
| R1 | **A recolha automática do e-Fatura não é viável de forma estável e legal** | 4 | 5 | 20 | Prova de conceito na Fase 0 como porta de decisão; carregamento manual e SAF-T sempre disponíveis; monitorização por cliente com alerta operacional | Sérgio |
| R2 | **O CC torna-se o gargalo e a promessa de 24 h quebra** | 4 | 5 | 20 | Medir deflexão e tempo por lançamento desde o piloto; abertura em lotes de 25; segundo CC contratado antes dos 250 clientes; política de uso justo | Carlos |
| R3 | **O assistente dá uma resposta errada com consequência fiscal** | 3 | 5 | 15 | Citações obrigatórias; triagem em três níveis; nível C sempre humano; conjunto dourado em CI; amostragem de 5–10 %; seguro de responsabilidade profissional; aviso legal em cada resposta | Luís |
| R4 | **O calendário até ao início de 2027 não é cumprido** | 4 | 3 | 12 | Corte de âmbito já assumido (Start primeiro); contratação do 2.º developer em setembro; lançamento faseado; Pro em beta se necessário | Sérgio |
| R5 | **Indisponibilidade do único CC responsável** | 2 | 5 | 10 | CC suplente contratualizado desde o lançamento; procuração e acessos documentados; plano de sucessão | Carlos |
| R6 | **Violação de dados** | 2 | 5 | 10 | Controlos de [05](05-seguranca-e-conformidade.md); pentest pré-lançamento; cifra ao nível do campo; manual de resposta ensaiado; seguro cibernético | Sérgio |
| R7 | **Aquisição de clientes abaixo do esperado / preço demasiado baixo para o serviço** | 3 | 4 | 12 | Validar disponibilidade a pagar no piloto; testar 49 €/79 €; funil orientado para o Pro; usar a rede dos sócios antes de gastar em aquisição | Luís |
| R8 | **Consentimentos PSD2 expiram e a conciliação silenciosamente deixa de funcionar** | 4 | 2 | 8 | Ecrã de ligações com estado e expiração; aviso 15 dias antes; painel operacional interno | Sérgio |
| R9 | **Custo de inferência de IA cresce acima do previsto** | 3 | 3 | 9 | Catálogo de respostas curadas antes do modelo; *cache* de contexto; modelo menor para triagem e classificação; teto de custo por cliente com alerta | Sérgio |
| R10 | **Enquadramento NIS2 revela-se aplicável, com obrigação de registo e coimas** | 2 | 3 | 6 | Parecer jurídico na Fase 0; controlos já implementados por desenho; registo é procedimento administrativo se necessário | Jurista |
| R11 | **Dependência de um único agregador PSD2 ou fornecedor de modelo** | 3 | 3 | 9 | Interfaces (`IBankAggregator`, `IAssistantModel`) desde o primeiro dia; não usar funcionalidades proprietárias no núcleo | Sérgio |
| R12 | **Concorrência de um player nacional com o mesmo posicionamento** | 3 | 3 | 9 | Defender pela proximidade e pela especialização em CAEs do Norte, não pela tecnologia | Luís |
| R13 | **DevExpress no portal do cliente degrada desempenho móvel** | 3 | 2 | 6 | DevExpress apenas no back-office; portal com componentes próprios leves | Sérgio |
| R14 | **Incumprimento AML detetado em inspeção à atividade do CC** | 2 | 4 | 8 | KYC como funcionalidade obrigatória do *onboarding*; conservação de 7 anos; formação registada | Carlos |

## 2. Decisões de arquitetura (ADR resumidos)

| ADR | Decisão | Alternativa rejeitada | Porquê |
|---|---|---|---|
| 001 | Monólito modular numa solução .NET | Microserviços | Equipa de 2–3 pessoas; complexidade operacional não se justifica |
| 002 | Dapper + *stored procedures* | EF Core | Convenção da casa; a guarda de `tenant_id` na BD é um controlo de segurança mais forte |
| 003 | Portal em `InteractiveAuto` + PWA; back-office em `InteractiveServer` | Blazor Server para tudo | Ligação persistente é inadequada a uso móvel em rede variável |
| 004 | Cifra ao nível do campo para NIF/IBAN/documento | Só cifra em repouso do volume | Protege contra fuga de cópia de segurança e acesso de leitura à BD |
| 005 | Não construir faturação certificada | Certificar software próprio junto da AT | Projeto autónomo de vários meses; não é o núcleo da proposta de valor |
| 006 | Agregador AISP licenciado | Pedir licença própria ao Banco de Portugal | Custo e prazo incomportáveis nesta fase |
| 007 | Corpus fiscal curado e versionado por vigência | RAG sobre legislação em bruto ou web aberta | Determinismo, defensabilidade e a exigência de citar a redação em vigor à data do facto |
| 008 | Fila de trabalhos em MariaDB (*outbox*) | RabbitMQ/Kafka desde o início | Menos uma peça de infraestrutura; migrar quando o volume o exigir |
| 009 | Auditoria imutável com encadeamento de *hashes* | Registo aplicacional simples | Prova perante AT/OCC e deteção de comprometimento interno |
| 010 | Conformidade material com NIS2 sem presumir obrigação de registo | Ignorar por não ser abrangida / registar por precaução | Requisito dos fundadores + cadeia de fornecimento + risco real do domínio |

## 3. Perguntas que só os fundadores podem responder

Cada uma destas altera o plano. Estão ordenadas por urgência.

### Bloqueantes na Fase 0

1. **O Carlos já tem clientes com delegação de sub-utilizador ativa no Portal das Finanças?**
   Se sim, a prova de conceito do R1 pode começar esta semana com dados reais e resolve-se
   a maior incerteza do projeto em duas semanas em vez de seis.
2. **Existe orçamento para contratar um developer sénior a tempo inteiro a partir de
   setembro?** Se não, o lançamento no início de 2027 não é realista e o âmbito tem de
   reduzir-se ao plano Start com carregamento manual.
3. **Quantos clientes o Carlos consegue efetivamente rever por dia, hoje, sem automação?**
   É o número que calibra todo o modelo de capacidade e de preço.
4. **A Cifra é uma sociedade já constituída, e sob que forma?** Condiciona o enquadramento
   junto da OCC, o seguro de responsabilidade e os contratos com subcontratantes.

### Decisões de produto

5. **O compromisso de 24 h é em dias úteis?** E abrange que tipo de perguntas? Sem limite
   escrito, é uma responsabilidade ilimitada com preço fixo.
6. **Os 39 € são negociáveis?** Testar 49 € no piloto custa pouco e pode alterar
   materialmente a viabilidade. O Pro a 69 € parece subvalorizado face ao trabalho de IVA
   trimestral e contabilidade organizada.
7. **Quais são os 10 CAEs prioritários?** Determina o corpus fiscal v1 e concentra o esforço
   onde há clientes reais.
8. **Que percentagem dos clientes-alvo aceitará ligar a conta bancária?** Se for baixa, a
   conciliação bancária desce de prioridade e liberta semanas na Fase 2.

### Governação e continuidade

9. **Quem assina se o Carlos não puder?** Precisa de resposta contratual antes do primeiro
   cliente, não depois.
10. **Existe seguro de responsabilidade civil profissional com cobertura adequada ao risco
    de erro assistido por IA?** Confirmar com a seguradora que o uso de IA não é exclusão.
11. **Qual é a política em caso de erro que gere coima ao cliente?** Assumir o custo é
    provavelmente a decisão comercial correta, mas tem de estar orçamentada e escrita.

## 4. Próximos passos concretos

Ordenados. As primeiras quatro linhas são para as próximas duas semanas.

| # | Ação | Dono | Prazo |
|---|---|---|---|
| 1 | Prova de conceito da delegação e-Fatura com 3 clientes reais | Sérgio + Carlos | 2 semanas |
| 2 | Abrir processo de contratação do developer .NET sénior | Sérgio | Imediato |
| 3 | Medir o tempo real de revisão do Carlos em 20 lançamentos | Carlos | 1 semana |
| 4 | Responder às perguntas 1–4 da §3 | Os três sócios | 1 semana |
| 5 | Contratar apoio jurídico e iniciar a DPIA | Sérgio | 3 semanas |
| 6 | Definir os 10 CAEs prioritários e iniciar o corpus fiscal v1 | Luís | 3 semanas |
| 7 | Selecionar agregador PSD2 e parceiro de faturação | Sérgio + Luís | 4 semanas |
| 8 | Corrigir o calendário de obrigações e os textos da página de pré-lançamento | Luís | 2 semanas |
| 9 | Rever consentimento e privacidade do formulário da lista de espera | Sérgio | 1 semana |
| 10 | Esqueleto da solução, CI/CD e ambientes | Sérgio | 4 semanas |
