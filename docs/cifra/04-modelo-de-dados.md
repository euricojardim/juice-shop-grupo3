# 04 · Modelo de dados (MariaDB)

## 1. Convenções

- Motor `InnoDB`, `utf8mb4_unicode_ci`, todas as datas em UTC (`DATETIME(3)`), apresentação
  convertida para `Europe/Lisbon` na UI.
- Chaves primárias `BIGINT UNSIGNED AUTO_INCREMENT`; identificadores expostos ao exterior
  são `UUID` (`BINARY(16)`) — **nunca expor o id sequencial numa URL ou API** (enumeração).
- Toda a tabela com dados de cliente tem `tenant_id` **não nulo** e índice que o inclui como
  primeira coluna.
- Colunas de auditoria em todas as tabelas: `criado_em`, `criado_por`, `alterado_em`,
  `alterado_por`.
- Apagamento lógico (`eliminado_em`) para dados sob obrigação de conservação; apagamento
  físico apenas pelo processo de retenção.
- Nomes em português, coerentes com a linguagem do domínio e com a *stack* da equipa.

## 2. Isolamento multi-tenant

Este é o controlo de segurança mais importante do sistema. Regras:

1. **A aplicação nunca constrói SQL.** Só chama *stored procedures*.
2. **Toda a *stored procedure* que toca em dados de cliente recebe `p_tenant_id` como
   primeiro parâmetro** e filtra por ele em todas as tabelas envolvidas.
3. **O `tenant_id` vem sempre da identidade autenticada**, nunca do corpo do pedido.
4. **A conta de aplicação na base de dados só tem `EXECUTE`** nas *stored procedures* —
   sem `SELECT`, `INSERT`, `UPDATE` ou `DELETE` diretos nas tabelas. Uma injeção de SQL
   passa a não ter superfície útil.
5. **Teste automatizado obrigatório:** para cada *stored procedure* de leitura, um teste que
   cria dois *tenants*, tenta ler do tenant A com o contexto do tenant B, e exige zero
   linhas. Deve correr em CI.

## 3. Esquema nuclear

```sql
-- Cliente subscritor (tenant)
CREATE TABLE tenants (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uuid            BINARY(16) NOT NULL UNIQUE,
    nome            VARCHAR(200) NOT NULL,
    nif_cifrado     VARBINARY(256) NOT NULL,       -- cifra ao nível do campo
    nif_hash        BINARY(32) NOT NULL,           -- HMAC para pesquisa exata
    tipo            ENUM('independente','eni','sociedade') NOT NULL,
    cae_principal   CHAR(5) NULL,
    plano           ENUM('start','pro','empresa') NOT NULL,
    estado          ENUM('lead','onboarding','ativo','suspenso','cessado') NOT NULL,
    cc_responsavel_id BIGINT UNSIGNED NOT NULL,
    criado_em       DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    alterado_em     DATETIME(3) NULL,
    UNIQUE KEY uk_tenants_nif (nif_hash),
    KEY ix_tenants_cc (cc_responsavel_id, estado)
) ENGINE=InnoDB;

-- Enquadramento fiscal com vigência temporal: a resposta correta depende da data do facto
CREATE TABLE enquadramentos (
    id                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    tenant_id         BIGINT UNSIGNED NOT NULL,
    regime_iva        ENUM('isento_art53','normal_trimestral','normal_mensal','isento_art9') NOT NULL,
    regime_irs        ENUM('simplificado','contabilidade_organizada') NOT NULL,
    tem_retencoes     TINYINT(1) NOT NULL DEFAULT 0,
    tem_trabalhadores TINYINT(1) NOT NULL DEFAULT 0,
    vigente_de        DATE NOT NULL,
    vigente_ate       DATE NULL,
    criado_por        BIGINT UNSIGNED NOT NULL,
    criado_em         DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    KEY ix_enq_tenant (tenant_id, vigente_de, vigente_ate),
    CONSTRAINT fk_enq_tenant FOREIGN KEY (tenant_id) REFERENCES tenants(id)
) ENGINE=InnoDB;

-- Documento no cofre. O ficheiro vive em object storage; a BD guarda metadados e a chave.
CREATE TABLE documentos (
    id             BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uuid           BINARY(16) NOT NULL UNIQUE,
    tenant_id      BIGINT UNSIGNED NOT NULL,
    origem         ENUM('efatura','upload','email','saft','banco') NOT NULL,
    tipo           ENUM('fatura','fatura_recibo','nota_credito','recibo','extrato','outro') NOT NULL,
    storage_key    VARCHAR(500) NOT NULL,
    sha256         BINARY(32) NOT NULL,            -- integridade e deduplicação
    mime           VARCHAR(100) NOT NULL,
    bytes          INT UNSIGNED NOT NULL,
    estado         ENUM('recebido','extraido','classificado','pendente_revisao',
                        'validado','devolvido','duplicado','fechado') NOT NULL,
    data_documento DATE NULL,
    nif_emitente   VARCHAR(20) NULL,
    total_base     DECIMAL(13,2) NULL,
    total_iva      DECIMAL(13,2) NULL,
    total          DECIMAL(13,2) NULL,
    criado_em      DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    UNIQUE KEY uk_doc_tenant_hash (tenant_id, sha256),
    KEY ix_doc_fila (tenant_id, estado, data_documento),
    CONSTRAINT fk_doc_tenant FOREIGN KEY (tenant_id) REFERENCES tenants(id)
) ENGINE=InnoDB;

-- Lançamento contabilístico proposto pela IA e decidido por um humano
CREATE TABLE lancamentos (
    id                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uuid              BINARY(16) NOT NULL UNIQUE,
    tenant_id         BIGINT UNSIGNED NOT NULL,
    documento_id      BIGINT UNSIGNED NOT NULL,
    periodo           CHAR(7) NOT NULL,             -- '2027-03'
    conta             VARCHAR(20) NOT NULL,
    descricao         VARCHAR(300) NOT NULL,
    base              DECIMAL(13,2) NOT NULL,
    taxa_iva          DECIMAL(5,2) NOT NULL,
    iva               DECIMAL(13,2) NOT NULL,
    iva_dedutivel     DECIMAL(13,2) NOT NULL DEFAULT 0,
    percentagem_afetacao DECIMAL(5,2) NOT NULL DEFAULT 100.00,
    estado            ENUM('proposto','pendente_revisao','validado','corrigido','anulado') NOT NULL,
    -- proveniência da proposta
    origem_proposta   ENUM('regra','ia','manual') NOT NULL,
    ia_interacao_id   BIGINT UNSIGNED NULL,
    confianca         DECIMAL(4,3) NULL,
    -- decisão humana
    validado_por      BIGINT UNSIGNED NULL,
    validado_em       DATETIME(3) NULL,
    nota_cc           TEXT NULL,
    criado_em         DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    KEY ix_lanc_revisao (tenant_id, estado, periodo),
    KEY ix_lanc_cc (estado, criado_em),
    CONSTRAINT fk_lanc_doc FOREIGN KEY (documento_id) REFERENCES documentos(id)
) ENGINE=InnoDB;

-- Regras de obrigações: dados, nunca código
CREATE TABLE obrigacoes_regras (
    id             BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    codigo         VARCHAR(50) NOT NULL,            -- 'IVA_TRIM', 'SS_TRIM', 'IRS_M3'
    designacao     VARCHAR(200) NOT NULL,
    entidade       ENUM('at','seg_social','outra') NOT NULL,
    periodicidade  ENUM('mensal','trimestral','anual','evento') NOT NULL,
    -- condição de aplicabilidade avaliada contra o enquadramento
    condicao_json  JSON NOT NULL,
    -- cálculo do prazo, ex.: {"tipo":"dia_do_mes","dia":20,"offset_meses":1}
    prazo_json     JSON NOT NULL,
    aviso_dias     SMALLINT NOT NULL DEFAULT 10,
    vigente_de     DATE NOT NULL,
    vigente_ate    DATE NULL,
    aprovado_por   BIGINT UNSIGNED NOT NULL,        -- sempre um CC
    aprovado_em    DATETIME(3) NOT NULL,
    KEY ix_regras_vigencia (codigo, vigente_de, vigente_ate)
) ENGINE=InnoDB;

-- Instância concreta da obrigação para um cliente e período
CREATE TABLE obrigacoes (
    id           BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    tenant_id    BIGINT UNSIGNED NOT NULL,
    regra_id     BIGINT UNSIGNED NOT NULL,
    periodo      CHAR(7) NOT NULL,
    prazo        DATE NOT NULL,
    estado       ENUM('pendente','em_preparacao','pronta','entregue','dispensada','falhada') NOT NULL,
    entregue_por BIGINT UNSIGNED NULL,
    entregue_em  DATETIME(3) NULL,
    comprovativo_documento_id BIGINT UNSIGNED NULL,
    UNIQUE KEY uk_obr (tenant_id, regra_id, periodo),
    KEY ix_obr_prazo (estado, prazo)
) ENGINE=InnoDB;

-- Registo integral de cada interação com o modelo (AI Act + defesa perante reclamações)
CREATE TABLE ia_interacoes (
    id                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    tenant_id         BIGINT UNSIGNED NULL,
    utilizador_id     BIGINT UNSIGNED NULL,
    tipo              ENUM('pergunta','classificacao','extracao','triagem') NOT NULL,
    nivel_risco       ENUM('a_automatico','b_assistido','c_humano') NOT NULL,
    prompt_hash       BINARY(32) NOT NULL,
    prompt_texto      MEDIUMTEXT NULL,              -- expurgado pela política de retenção
    contexto_refs     JSON NOT NULL,                -- excertos do corpus efetivamente usados
    resposta_texto    MEDIUMTEXT NULL,
    citacoes          JSON NULL,
    modelo            VARCHAR(100) NOT NULL,
    versao_corpus     VARCHAR(50) NOT NULL,
    confianca         DECIMAL(4,3) NULL,
    escalado          TINYINT(1) NOT NULL DEFAULT 0,
    revisto_por       BIGINT UNSIGNED NULL,
    avaliacao_cc      ENUM('correta','parcial','incorreta') NULL,
    tokens_entrada    INT UNSIGNED NULL,
    tokens_saida      INT UNSIGNED NULL,
    criado_em         DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    KEY ix_ia_tenant (tenant_id, criado_em),
    KEY ix_ia_qualidade (avaliacao_cc, criado_em)
) ENGINE=InnoDB;
```

Tabelas complementares: `utilizadores`, `perfis`, `sessoes`, `kyc_verificacoes`,
`kyc_sinalizacoes`, `contratos`, `consentimentos`, `ligacoes_efatura`, `ligacoes_bancarias`,
`movimentos_bancarios`, `conciliacoes`, `conversas`, `mensagens`, `subscricoes`,
`cobrancas`, `notificacoes`, `pedidos_privacidade`, `corpus_fiscal`, `corpus_excertos`.

## 4. Corpus fiscal versionado

```sql
CREATE TABLE corpus_excertos (
    id            BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    fonte         VARCHAR(200) NOT NULL,     -- 'CIVA art. 21.º', 'Ofício Circulado 30xxx'
    referencia    VARCHAR(300) NOT NULL,     -- citação exibida ao utilizador
    url           VARCHAR(500) NULL,
    texto         MEDIUMTEXT NOT NULL,
    caes          JSON NULL,                 -- restringe a CAEs, quando aplicável
    regimes       JSON NULL,
    vigente_de    DATE NOT NULL,
    vigente_ate   DATE NULL,
    versao_corpus VARCHAR(50) NOT NULL,
    aprovado_por  BIGINT UNSIGNED NOT NULL,  -- Luís ou Carlos: nenhum excerto entra sem aprovação
    aprovado_em   DATETIME(3) NOT NULL,
    KEY ix_corpus_vig (vigente_de, vigente_ate)
) ENGINE=InnoDB;
```

Os vetores de pesquisa podem viver na coluna `VECTOR` do MariaDB 11.7+ ou num índice
externo. Recomendação: começar com **pesquisa lexical (FULLTEXT) + filtro por vigência,
CAE e regime**, e só acrescentar pesquisa vetorial se a lexical se mostrar insuficiente.
Num corpus curado de algumas centenas de excertos, a lexical costuma bastar e é
inspecionável — o que importa quando é preciso explicar uma resposta.

## 5. Cifra de campos sensíveis

NIF, IBAN, documento de identificação e morada são cifrados na aplicação
(AES-256-GCM, chave por *tenant* derivada de uma chave-mestra em KMS/HSM) antes de
chegarem à base de dados. Para permitir pesquisa exata guarda-se em paralelo um
`HMAC-SHA256` com chave dedicada. Consequências assumidas: não há pesquisa parcial
(`LIKE`) sobre estes campos, e o desenho da UI tem de o refletir.

Isto acresce — não substitui — a cifra em repouso do volume e o TLS em trânsito. Protege
contra o cenário realista de fuga de uma cópia de segurança ou de acesso de leitura à base
de dados.

## 6. Auditoria imutável

```sql
CREATE TABLE auditoria (
    id             BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    tenant_id      BIGINT UNSIGNED NULL,
    utilizador_id  BIGINT UNSIGNED NULL,
    perfil         VARCHAR(30) NULL,
    acao           VARCHAR(100) NOT NULL,      -- 'lancamento.validado', 'obrigacao.entregue'
    entidade       VARCHAR(50) NOT NULL,
    entidade_id    BIGINT UNSIGNED NULL,
    dados_antes    JSON NULL,
    dados_depois   JSON NULL,
    ip             VARBINARY(16) NULL,
    user_agent     VARCHAR(300) NULL,
    trace_id       CHAR(32) NULL,
    hash_anterior  BINARY(32) NOT NULL,
    hash_atual     BINARY(32) NOT NULL,        -- SHA256(hash_anterior || conteúdo canónico)
    criado_em      DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    KEY ix_aud_tenant (tenant_id, criado_em),
    KEY ix_aud_entidade (entidade, entidade_id)
) ENGINE=InnoDB;
```

O encadeamento de *hashes* torna qualquer adulteração detetável. Um trabalho diário verifica
a cadeia e publica o *hash* do último registo num registo separado e apenas-anexação. A conta
de aplicação tem `INSERT` mas não `UPDATE` nem `DELETE` nesta tabela.

Isto responde a três necessidades ao mesmo tempo: prova de quem validou o quê perante a AT
ou a OCC, requisito de registo do RGPD e do AI Act, e deteção de comprometimento interno.

## 7. Padrão de *stored procedure* com guarda de tenant

```sql
DELIMITER $$

CREATE PROCEDURE sp_lancamentos_fila_revisao(
    IN p_tenant_id  BIGINT UNSIGNED,
    IN p_cc_id      BIGINT UNSIGNED,
    IN p_limite     INT
)
BEGIN
    -- O CC só vê tenants que lhe estão atribuídos. Verificação na BD, não só em C#.
    IF NOT EXISTS (
        SELECT 1 FROM tenants
        WHERE id = p_tenant_id AND cc_responsavel_id = p_cc_id AND estado = 'ativo'
    ) THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Acesso não autorizado a este cliente.';
    END IF;

    SELECT l.id, l.uuid, l.periodo, l.conta, l.descricao,
           l.base, l.taxa_iva, l.iva, l.iva_dedutivel,
           l.confianca, l.origem_proposta,
           d.uuid AS documento_uuid, d.tipo, d.data_documento, d.nif_emitente
    FROM lancamentos l
    INNER JOIN documentos d ON d.id = l.documento_id AND d.tenant_id = l.tenant_id
    WHERE l.tenant_id = p_tenant_id
      AND l.estado = 'pendente_revisao'
    ORDER BY l.confianca ASC, d.data_documento ASC   -- menor confiança primeiro
    LIMIT p_limite;
END$$

CREATE PROCEDURE sp_validar_lancamento(
    IN p_tenant_id BIGINT UNSIGNED,
    IN p_cc_id     BIGINT UNSIGNED,
    IN p_lanc_id   BIGINT UNSIGNED,
    IN p_nota      TEXT
)
BEGIN
    DECLARE v_estado VARCHAR(30);
    DECLARE EXIT HANDLER FOR SQLEXCEPTION BEGIN ROLLBACK; RESIGNAL; END;

    IF NOT EXISTS (
        SELECT 1 FROM utilizadores u
        WHERE u.id = p_cc_id AND u.perfil = 'cc' AND u.ativo = 1
    ) THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Apenas um Contabilista Certificado pode validar lançamentos.';
    END IF;

    START TRANSACTION;

    SELECT estado INTO v_estado
    FROM lancamentos
    WHERE id = p_lanc_id AND tenant_id = p_tenant_id
    FOR UPDATE;

    IF v_estado IS NULL THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Lançamento inexistente.';
    END IF;
    IF v_estado <> 'pendente_revisao' THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Lançamento não está pendente de revisão.';
    END IF;

    UPDATE lancamentos
       SET estado = 'validado', validado_por = p_cc_id,
           validado_em = NOW(3), nota_cc = p_nota
     WHERE id = p_lanc_id AND tenant_id = p_tenant_id;

    UPDATE documentos d
      INNER JOIN lancamentos l ON l.documento_id = d.id
       SET d.estado = 'validado'
     WHERE l.id = p_lanc_id AND d.tenant_id = p_tenant_id;

    COMMIT;
    SELECT p_lanc_id AS id, 'validado' AS estado;
END$$

DELIMITER ;
```

O repositório Dapper correspondente segue o padrão da casa:

```csharp
public sealed class LancamentoRepository : DapperRepository, ILancamentoRepository
{
    public LancamentoRepository(IConfiguration config) : base(config) { }

    public Task<IEnumerable<LancamentoRevisaoDto>> GetFilaRevisaoAsync(
        long tenantId, long ccId, int limite = 50)
        => QueryAsync<LancamentoRevisaoDto>("sp_lancamentos_fila_revisao",
            new { p_tenant_id = tenantId, p_cc_id = ccId, p_limite = limite });

    public Task<int> ValidarAsync(long tenantId, long ccId, long lancamentoId, string? nota)
        => ExecuteAsync("sp_validar_lancamento",
            new { p_tenant_id = tenantId, p_cc_id = ccId,
                  p_lanc_id = lancamentoId, p_nota = nota });
}
```

O `tenantId` e o `ccId` são obtidos das *claims* da identidade autenticada num
`ITenantContext` com âmbito de pedido — **nunca de um parâmetro de rota ou do corpo do
pedido**. Uma análise (Roslyn analyzer ou revisão de código) deve garantir que nenhum
método de repositório aceita `tenantId` de fonte não confiável.

## 8. Desempenho

- Todas as consultas de listagem são paginadas com *keyset pagination* (`WHERE id > ?`),
  não `OFFSET` — a fila de revisão e o histórico de documentos crescem rapidamente.
- Índices desenhados a partir das consultas reais, sempre com `tenant_id` como prefixo.
- Documentos e movimentos bancários são as tabelas de crescimento rápido: prever partição
  por ano quando ultrapassarem alguns milhões de linhas.
- Réplica de leitura para relatórios e para a análise interna, para não competir com a
  operação em época de IVA.
