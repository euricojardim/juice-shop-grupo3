# Cifra — Análise Profunda e Plano de Desenvolvimento

> Plataforma de contabilidade assistida por IA com validação por Contabilista Certificado.
> Norte de Portugal · lançamento previsto para o início de 2027.

Documento elaborado a partir da página de pré-lançamento e dos requisitos técnicos
definidos: **MariaDB**, **C# / Blazor**, **API segura**, front-end apelativo e *user friendly*,
conformidade com o **RGPD** e com o regime **NIS2**.

---

## Índice

| # | Documento | Conteúdo |
|---|-----------|----------|
| 01 | [Análise de produto e negócio](01-analise-produto-e-negocio.md) | Proposta de valor, segmento, capacidade do CC, unit economics, o que a página promete a mais |
| 02 | [Análise regulatória](02-analise-regulatoria.md) | OCC, AT/e-Fatura, faturação certificada, PSD2, AML, RGPD, DL 125/2025 (NIS2), AI Act |
| 03 | [Arquitetura](03-arquitetura.md) | Contexto C4, módulos, stack .NET 10 / Blazor / Dapper / MariaDB, integrações, camada de IA |
| 04 | [Modelo de dados](04-modelo-de-dados.md) | Esquema MariaDB, isolamento multi-tenant, auditoria imutável, convenções de *stored procedures* |
| 05 | [Segurança e conformidade](05-seguranca-e-conformidade.md) | Modelo de ameaças, matriz de controlos RGPD art. 32.º × NIS2 art. 21.º, SDLC, resposta a incidentes |
| 06 | [Front-end e UX](06-frontend-ux.md) | Modos de render Blazor, PWA, ecrãs-chave, transparência IA/humano, acessibilidade |
| 07 | [Roadmap, equipa e custos](07-roadmap-equipa-custos.md) | Fases Set/2026 → 2027, MVP, equipa, orçamento, infraestrutura |
| 08 | [Riscos e decisões em aberto](08-riscos-e-decisoes.md) | Registo de riscos, ADRs, perguntas para os fundadores |

---

## Sumário executivo

A Cifra é tecnicamente exequível com a *stack* pretendida, mas **o risco dominante não é
tecnológico — é regulatório, operacional e de capacidade humana**. A tecnologia (Blazor,
MariaDB, API, IA com RAG) é bem compreendida e replicável. O que determina o sucesso é:

1. conseguir aceder às faturas do cliente de forma **legal, estável e sem guardar credenciais pessoais**;
2. garantir que **um Contabilista Certificado consegue validar** o volume gerado sem se tornar o gargalo;
3. cumprir obrigações que a página de pré-lançamento ainda não reflete (**AML, DPIA, AI Act, sucessão do CC**).

### As 10 conclusões que mais condicionam o plano

| # | Conclusão | Impacto |
|---|-----------|---------|
| 1 | **Não existe API pública do e-Fatura para terceiros.** O acesso legítimo faz-se por delegação de sub-utilizador no Portal das Finanças ao NIF do CC, com descarga do ficheiro de faturas / SAF-T. | Redesenha o *onboarding*. **Nunca** guardar a senha pessoal do cliente. Ver [02](02-analise-regulatoria.md#21-autoridade-tributária-e-e-fatura). |
| 2 | **Não construir faturação certificada.** A certificação AT (DL 28/2019 + Portaria 195/2020 / ATCUD) é um projeto autónomo de vários meses com auditoria. | Fazer *white-label* / parceria com um emissor certificado. Poupa ~4–6 meses. |
| 3 | **A Cifra não pode ler contas bancárias por si.** A leitura de movimentos exige um AISP licenciado (PSD2). O consentimento caduca e obriga a renovação periódica com SCA. | Contratar agregador (Tink / GoCardless / Salt Edge). O UX tem de prever re-consentimento. |
| 4 | **O CC é entidade obrigada em matéria de branqueamento de capitais** (Lei 83/2017). | KYC, conservação de 7 anos e deveres de recusa/abstenção passam a ser **funcionalidade de produto**, não burocracia. |
| 5 | **NIS2 já está transposta**: Decreto-Lei n.º 125/2025 + Regulamento CNCS n.º 756/2026. A Cifra, como microempresa de contabilidade, muito provavelmente **não** é entidade essencial nem importante. | Conformidade **por desenho e por escolha**, não por obrigação — mas exigida por clientes B2B abrangidos e pelo crescimento. Confirmar com jurista. |
| 6 | **O AI Act art. 50.º é aplicável desde 2 de agosto de 2026.** O assistente tem de se identificar como IA na primeira interação. | A distinção visual "assistente" vs. "contabilista" deixa de ser design e passa a ser **requisito testável**. |
| 7 | **Conflito RGPD × obrigação fiscal de conservação (10 anos).** | Política de retenção por categoria de dados, com bases legais distintas e apagamento diferenciado. **DPIA obrigatória antes do lançamento.** |
| 8 | **Um único CC responsável é um ponto único de falha regulatório.** Se o Carlos ficar indisponível, ninguém assina. | Plano de sucessão contratualizado e segundo CC a partir de ~250 clientes. É requisito de continuidade de negócio, não opcional. |
| 9 | **"Resposta em < 24 h" + "preço fixo" é a maior ameaça à margem.** | Definir política de uso justo, triagem automática por risco e limites de âmbito nos planos. Ver [01](01-analise-produto-e-negocio.md#5-economia-unitária-e-capacidade). |
| 10 | **O calendário de obrigações da página está incompleto.** Faltam a declaração trimestral à Segurança Social, a DMR/Modelo 10 e a IES para ENI com contabilidade organizada. | Corrigir antes do lançamento — é a promessa central do produto ("o que a Cifra nunca deixa passar"). |

### Recomendação de âmbito para o lançamento

Entre setembro de 2026 e o lançamento no início de 2027 existem aproximadamente **cinco meses úteis**.
Não é tempo para construir tudo o que a página anuncia. O corte recomendado:

- **Entra no MVP:** *onboarding* com KYC, cofre documental, captura de fatura por telemóvel com OCR,
  classificação assistida, fila de revisão do CC, assistente fiscal com citações obrigatórias,
  calendário de obrigações e notificações, plano **Start**.
- **Entra na segunda vaga (Q1 2027):** IVA trimestral e retenções, conciliação bancária, plano **Pro**.
- **Fica para 2027 H2 ou depois:** faturação integrada (via parceiro), plano **Empresa**,
  processamento salarial, IRC/IES.

O plano detalhado está em [07 — Roadmap, equipa e custos](07-roadmap-equipa-custos.md).
