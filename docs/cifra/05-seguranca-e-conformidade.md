# 05 · Segurança e conformidade

## 1. O que se está a proteger

A Cifra concentra, por cliente: NIF, morada, documento de identificação, faturas de compra
e venda, extratos bancários, e — o mais crítico — **acesso delegado ao Portal das Finanças**.
Um comprometimento não é um incidente de privacidade: é fraude fiscal em nome de terceiros,
com responsabilidade profissional do CC.

Perfil de atacante realista, por ordem de probabilidade:

1. **Fraude oportunista** — comprometimento de conta de cliente (credenciais reutilizadas)
   para aceder a documentos ou redirecionar reembolsos.
2. **Comprometimento do back-office do CC** — a conta com maior valor no sistema; acede a
   toda a carteira.
3. ***Ransomware* / extorsão** — o modelo dominante contra PMEs portuguesas.
4. **Insider** — colaborador de apoio com acesso excessivo.
5. **Cadeia de fornecimento** — pacote NuGet, imagem de contentor ou fornecedor SaaS.

## 2. Modelo de ameaças (STRIDE) dos fluxos críticos

| Fluxo | Ameaça | Mitigação |
|---|---|---|
| Autenticação do cliente | Credenciais reutilizadas, *credential stuffing* | Argon2id, verificação contra listas de senhas comprometidas, 2FA (TOTP/passkey), limitação de taxa por IP e por conta, bloqueio progressivo, alerta de novo dispositivo |
| Sessão do back-office | Roubo de sessão do CC | 2FA **obrigatório**, sessão curta com renovação, restrição por intervalo de IP, revogação imediata, alerta em ações sensíveis |
| Acesso a documentos | Referência direta insegura (IDOR) | Identificadores UUID, autorização por recurso, `tenant_id` validado na *stored procedure*, URLs pré-assinadas de curta duração |
| Ingestão do e-Fatura | Roubo das credenciais delegadas | Segredos em cofre (não na BD nem em variáveis de ambiente da aplicação), permissões mínimas na delegação, rotação, alerta em uso fora de janela |
| Carregamento de ficheiros | Malware, *zip bomb*, SSRF via PDF, XXE | Antivírus, limite de tamanho e de tipo real (*magic bytes*), processamento em contentor isolado sem rede, sanitização de PDF, sem *parsers* XML com entidades externas |
| Assistente de IA | Injeção de instruções via conteúdo de fatura | Conteúdo de documento tratado sempre como **dados**, nunca como instrução; sem ferramentas de escrita; validação de saída; citações obrigatórias |
| Base de dados | Injeção de SQL, exfiltração | Apenas *stored procedures* parametrizadas, conta de aplicação só com `EXECUTE`, cifra de campos sensíveis |
| Cópias de segurança | Fuga de cópia | Cifra com chave distinta da produção, acesso segregado, teste de restauro registado |
| Pagamentos | Manipulação de valores | Valores calculados no servidor, *webhooks* com assinatura verificada, idempotência |
| Auditoria | Adulteração de registos para esconder ação | Cadeia de *hashes*, sem `UPDATE`/`DELETE` para a aplicação, verificação diária |

## 3. Matriz de controlos

As dez famílias de medidas de gestão de risco previstas na NIS2 (art. 21.º), refletidas no
Decreto-Lei n.º 125/2025, cruzadas com o art. 32.º do RGPD e com a implementação concreta.

| # | Medida (NIS2 art. 21.º) | RGPD | Implementação na Cifra | Fase |
|---|---|---|---|---|
| 1 | Análise de riscos e políticas de segurança | 32.º, 24.º | Política de segurança aprovada pelos sócios; análise de risco anual; modelo de ameaças por *release* maior; registo de riscos vivo | 0 |
| 2 | Gestão de incidentes | 33.º, 34.º | Plano de resposta com papéis nomeados; classificação; **prazos: alerta 24 h / notificação 72 h / relatório final 1 mês**; canal único de reporte; exercício semestral | 1 |
| 3 | Continuidade e gestão de crises | 32.º/1/c | RPO ≤ 15 min, RTO ≤ 4 h; cópias 3-2-1; **restauro testado mensalmente com registo**; plano de crise e comunicação a clientes | 1 |
| 4 | Segurança da cadeia de fornecimento | 28.º | Inventário de fornecedores com DPA e localização de dados; avaliação antes de contratar; SBOM (CycloneDX) por *build*; verificação de dependências em CI | 0–2 |
| 5 | Segurança na aquisição, desenvolvimento e manutenção | 25.º, 32.º | SDLC seguro (§4); *code review* obrigatório; SAST/DAST/SCA em CI; gestão de vulnerabilidades com SLA | 1 |
| 6 | Avaliação da eficácia das medidas | 32.º/1/d | Métricas de segurança trimestrais; pentest externo anual e antes do lançamento; revisão de acessos trimestral | 2 |
| 7 | Higiene e formação em cibersegurança | 39.º | Formação de admissão e anual; formação AML (obrigatória para o CC); simulação de *phishing* semestral | 1 |
| 8 | Criptografia | 32.º/1/a | TLS 1.3; cifra em repouso; cifra ao nível do campo para NIF/IBAN/documento; chaves em KMS/HSM com rotação anual | 1 |
| 9 | Segurança de recursos humanos, controlo de acessos e gestão de ativos | 32.º/4 | Menor privilégio; 2FA obrigatório interno; revisão trimestral; processo de saída em 24 h; inventário de ativos | 1 |
| 10 | Autenticação multifator e comunicações seguras | 32.º | 2FA obrigatório para CC/apoio/administração e para todos os acessos administrativos; *passkeys* para clientes; acesso administrativo apenas por VPN/rede restrita | 1 |

### Controlos adicionais específicos do domínio

| Controlo | Razão |
|---|---|
| Segregação de funções: quem carrega documento não valida lançamento | Requisito profissional do CC |
| Nenhum caminho de código submete à AT sem ação humana registada | Invariante regulatório — testado em CI |
| Registo de acesso a documentos de cliente, consultável pelo próprio cliente | Transparência; deteção de abuso interno |
| Dados de produção **nunca** em ambientes de teste | Falha comum e materialmente grave |
| Ambiente de execução do OCR e do processamento de ficheiros sem acesso à rede | Contenção de conteúdo hostil |

## 4. Ciclo de desenvolvimento seguro

| Etapa | Prática | Ferramenta |
|---|---|---|
| Desenho | Modelo de ameaças por funcionalidade que toque em dados de cliente ou em dinheiro | Documento no repositório |
| Código | *Code review* obrigatório; sem segredos no repositório; `TreatWarningsAsErrors` | GitHub + `dotnet format` |
| Estático | SAST em cada PR | CodeQL (já ativo neste repositório) + Security Code Scan |
| Dependências | SCA e alerta de vulnerabilidade | Dependabot + `dotnet list package --vulnerable` em CI |
| Segredos | Deteção antes do *commit* | Gitleaks + *pre-commit hook* |
| Contentores | Análise de imagem | Trivy; imagens base *distroless* ou *chiselled* |
| Testes | Testes de isolamento multi-tenant obrigatórios; cobertura mínima na camada de domínio | xUnit + Testcontainers |
| Dinâmico | DAST em *staging* | OWASP ZAP em modo *baseline* nas noites |
| Pré-lançamento | **Pentest externo por terceiro** | Obrigatório antes do primeiro cliente pagante |
| Produção | Deteção e alerta | Serilog + OpenTelemetry → SIEM leve (Grafana/Loki com regras de alerta) |

> **Nota interna.** Este repositório é um *fork* do OWASP Juice Shop — uma aplicação
> deliberadamente vulnerável para formação em segurança. É um excelente ativo de formação
> para a equipa da Cifra: as vulnerabilidades que treina (IDOR, injeção de SQL, controlo de
> acessos partido, carregamento de ficheiros inseguro) são exatamente as que mais ameaçam
> uma plataforma com este perfil. Recomenda-se usá-lo como base da formação obrigatória
> anual prevista na medida 7.

## 5. Cabeçalhos e configuração da aplicação

```csharp
app.Use(async (ctx, next) =>
{
    var h = ctx.Response.Headers;
    h["Content-Security-Policy"] =
        "default-src 'self'; script-src 'self'; style-src 'self'; " +
        "img-src 'self' data: blob:; connect-src 'self'; frame-ancestors 'none'; " +
        "form-action 'self'; base-uri 'self'; object-src 'none'";
    h["Strict-Transport-Security"] = "max-age=63072000; includeSubDomains; preload";
    h["X-Content-Type-Options"] = "nosniff";
    h["Referrer-Policy"] = "strict-origin-when-cross-origin";
    h["Permissions-Policy"] = "camera=(self), microphone=(), geolocation=()";
    h["Cross-Origin-Opener-Policy"] = "same-origin";
    await next();
});
```

> O `script-src 'self'` sem `unsafe-inline` exige atenção ao Blazor e ao DevExpress:
> validar em *staging* e usar *nonces* onde for inevitável. Não relaxar a CSP para
> contornar um componente — substituir o componente.

Complementar com: `AddRateLimiter` (mais restritivo em autenticação, recuperação de senha e
assistente), antiforgery em todos os formulários, cookies `HttpOnly` + `Secure` +
`SameSite=Strict`, e cabeçalho `Cache-Control: no-store` em todas as respostas com dados de
cliente.

## 6. Resposta a incidentes

Papéis nomeados (numa equipa pequena, uma pessoa acumula, mas o papel tem de existir):
**coordenador de incidente** (Sérgio), **responsável de comunicação** (Luís), **responsável
de conformidade** (Carlos), **apoio jurídico externo**.

| Momento | Ação |
|---|---|
| T+0 | Deteção → abertura de incidente → classificação (P1–P4) |
| T+1 h | Contenção; isolar credenciais e sistemas afetados; preservar prova |
| T+24 h | **Alerta precoce** se houver enquadramento NIS2; avaliação preliminar de violação de dados |
| T+72 h | **Notificação à CNPD** se houver risco para direitos e liberdades; comunicação aos titulares se o risco for elevado |
| T+7 dias | Comunicação a clientes afetados; ponto de situação público se aplicável |
| T+1 mês | **Relatório final**; análise de causa raiz sem culpabilização; ações corretivas com prazo |

Cenários com manual de resposta escrito e ensaiado: (a) comprometimento de conta de CC,
(b) *ransomware* na infraestrutura, (c) fuga de documentos de clientes, (d) resposta
gravemente errada do assistente com consequência fiscal, (e) indisponibilidade prolongada
em época de IVA.

O cenário (d) merece destaque: é o mais provável de todos e não é um incidente de segurança
clássico. Precisa de processo próprio — deteção por reclamação ou amostragem, avaliação do
impacto fiscal, correção junto da AT, comunicação ao cliente, e acionamento do seguro de
responsabilidade profissional se necessário.

## 7. Requisitos verificáveis (aceitação de segurança)

Lista fechada, a validar antes do primeiro cliente pagante. Cada linha deve ter um teste
automatizado ou uma evidência documentada:

1. Um utilizador do tenant A não consegue ler nenhum recurso do tenant B, por nenhuma via.
2. Um utilizador sem perfil CC não consegue transitar um lançamento para `validado`.
3. Nenhum endpoint aceita `tenant_id` do cliente; é sempre derivado da identidade.
4. A conta de aplicação na base de dados não tem `SELECT` direto em nenhuma tabela.
5. Todos os segredos vêm de cofre; nenhum aparece no repositório nem em imagem de contentor.
6. A cadeia de auditoria verifica sem quebras nos últimos 90 dias.
7. Um restauro completo a partir de cópia foi executado e cronometrado nos últimos 30 dias.
8. O assistente não produz resposta normativa sem citação verificável do corpus.
9. Um pedido de apagamento executa apagamento parcial correto e devolve explicação.
10. O pentest externo não deixou constatações de severidade alta ou crítica por resolver.
