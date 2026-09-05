# 06 · Front-end e experiência de utilização

## 1. Duas aplicações, dois problemas diferentes

| | Portal do Cliente | Back-office do CC |
|---|---|---|
| Utilizador | Independente, pouca literacia fiscal, com pressa | Profissional, alto volume, teclado |
| Contexto | Telemóvel, rede variável, sessões curtas | Escritório, ecrã grande, sessões longas |
| Tarefa dominante | Fotografar fatura, perguntar, ver se está tudo em dia | Rever, corrigir, aprovar em lote, responder |
| Métrica de sucesso | Tempo até "está tratado" | Lançamentos revistos por hora |
| Tecnologia | **Blazor Web App `InteractiveAuto` + PWA** | **Blazor Server + DevExpress Blazor** |

Tratar isto como uma aplicação só é o erro mais provável do projeto. As restrições são
opostas: o cliente precisa de tolerância a rede fraca, o CC precisa de densidade de
informação.

## 2. Portal do Cliente

### Princípios

1. **A ação principal é sempre uma.** No ecrã inicial, um botão grande: fotografar fatura.
   Tudo o resto é secundário.
2. **Linguagem simples, tom da marca.** "Contas claras" é a promessa; a interface tem de a
   cumprir. Nunca "dedutibilidade do IVA suportado" — sempre "podes recuperar 54,05 € de IVA".
3. **Estado sempre visível.** O cliente deve saber, sem perguntar: o que está tratado, o que
   falta, e o que está à espera dele.
4. **Distinguir sempre quem fala.** Assistente e contabilista têm identidade visual
   inconfundível. É requisito legal (AI Act art. 50.º) e é o núcleo da confiança.
5. **Tolerar rede fraca.** Fotografar → guardar localmente → enviar quando houver rede,
   com confirmação. Falhar um envio no meio da rua não pode perder a fatura.

### Ecrãs

| Ecrã | Conteúdo | Notas |
|---|---|---|
| **Início** | Botão de captura; "o que falta de ti"; próxima obrigação com contagem decrescente; estado do mês | Um cartão de estado, não um *dashboard* |
| **Capturar** | Câmara com deteção de limites, correção de perspetiva, multi-página; confirmação com valores extraídos editáveis | O passo mais usado. Merece o maior investimento de UX |
| **Documentos** | Lista com estado (`recebido`, `em revisão`, `validado`, `precisa de ti`), filtro por mês, pesquisa | *Keyset pagination*; rolagem infinita |
| **Perguntar** | Conversa; resposta com citação e aviso; escalada visível para o CC | Ver §3 |
| **Obrigações** | Linha temporal dos prazos aplicáveis ao enquadramento do cliente; estado e comprovativo | Materializa "a Cifra nunca deixa passar" |
| **Ligações** | Estado do e-Fatura e do banco, com renovação de consentimento | Ver §4 |
| **A minha conta** | Plano, faturas da Cifra, dados pessoais, exportar tudo, apagar dados | RGPD acessível, não escondido |

### Acessibilidade e desempenho

- **WCAG 2.2 nível AA** como requisito, não aspiração: contraste ≥ 4.5:1, alvos de toque
  ≥ 44 px, navegação por teclado completa, `aria-live` para estados assíncronos, respeito por
  `prefers-reduced-motion`.
- Testar com tamanho de letra do sistema a 200 % — parte do público-alvo tem 50+ anos.
- Orçamento de desempenho: primeiro conteúdo < 1,5 s em 4G; *bundle* WASM inicial < 2 MB
  comprimido (usar *lazy loading* por rota e *trimming*).
- **pt-PT rigoroso**, com tratamento por tu, coerente com a página de pré-lançamento.

## 3. O padrão de conversa — o elemento distintivo

O exemplo da página já define o padrão certo. Formalizá-lo como componente:

```
┌──────────────────────────────────────────────┐
│ ● assistente                          14:28  │   ← rótulo permanente, cor e ícone próprios
│ Sim. Equipamento informático não está entre  │
│ as despesas excluídas de dedução.            │
│                                              │
│ ⌗ CIVA, art. 21.º · redação em vigor em 2027 │   ← citação obrigatória e clicável
│                                              │
│ ┌────────────────────────────────────────┐   │
│ │ Base            235,00 €               │   │   ← lançamento proposto, estruturado
│ │ IVA 23 % (dedutível)  54,05 €          │   │
│ │ Total           289,05 €               │   │
│ └────────────────────────────────────────┘   │
│ ⏳ Pendente de validação pelo contabilista    │   ← estado real, não decorativo
│ ⓘ Orientação preliminar sujeita a validação  │   ← aviso legal em cada resposta normativa
└──────────────────────────────────────────────┘
┌──────────────────────────────────────────────┐
│ 👤 Carlos · Contabilista Certificado   14:32 │   ← visual claramente distinto
│ Confirmado e lançado. Uma nota: ...          │
│ ✓ Validado                                   │
└──────────────────────────────────────────────┘
```

Requisitos testáveis que decorrem daqui:

1. Ao abrir a conversa pela primeira vez, o utilizador é informado de que fala com um
   sistema de IA (AI Act art. 50.º) — teste de UI automatizado.
2. Nenhuma mensagem do assistente pode ser confundida com uma mensagem do CC: rótulo,
   cor e ícone distintos, em ambos os temas e em modo de alto contraste.
3. Toda a resposta normativa exibe pelo menos uma citação clicável e o aviso de orientação
   preliminar.
4. O estado de validação de um lançamento apresentado na conversa reflete o estado real na
   base de dados, em tempo real.

## 4. O ecrã que ninguém desenha e que causa metade do suporte

**Ligações.** As duas integrações críticas (e-Fatura e banco) vão falhar periodicamente —
delegação revogada, consentimento PSD2 expirado, alteração no portal da AT. Se isso for
silencioso, o cliente descobre quando falhar um prazo, e a promessa central da Cifra quebra.

O ecrã deve mostrar, para cada ligação: estado atual, data da última sincronização com
sucesso, **data de expiração do consentimento**, e um botão de reparação com instruções
passo a passo. O sistema avisa proativamente (*push* + email) 15 dias antes da expiração e
imediatamente quando uma sincronização falha duas vezes seguidas.

Do lado interno, o mesmo estado alimenta um painel operacional: nenhum cliente deve ficar
mais de X dias sem sincronização bem-sucedida sem que alguém saiba.

## 5. Back-office do CC

O objetivo é maximizar **lançamentos revistos por hora**, com qualidade.

- **Fila única**, ordenada por risco (confiança do modelo ascendente) e por prazo, não por
  cliente. O CC trabalha a fila, não navega entre clientes.
- **Aprovação em lote** para lançamentos semelhantes e de alta confiança: selecionar todos
  os que correspondem a um padrão já validado (ex.: combustível do mesmo fornecedor, mesma
  classificação) e aprovar num gesto. É aqui que se ganha a margem.
- **Documento e proposta lado a lado**, com atalhos de teclado (`A` aprovar, `C` corrigir,
  `D` devolver, `→` seguinte). Sem rato.
- **Correções alimentam as regras.** Quando o CC corrige a mesma classificação três vezes
  para o mesmo padrão, o sistema propõe criar uma regra permanente. Explicitamente sugerido,
  nunca aplicado automaticamente.
- **Painel de carteira**: clientes em risco de falhar prazo, ligações partidas, perguntas
  por responder com tempo decorrido face ao compromisso de 24 h.
- **Medição de tempo por lançamento e por resposta**, sem vigilância intrusiva — é métrica
  do negócio (ver [01 §5](01-analise-produto-e-negocio.md#5-economia-unitária-e-capacidade)),
  agregada e transparente para o próprio CC.

DevExpress Blazor entrega isto bem: `DxGrid` com filtro, seleção múltipla e templates de
célula; `DxPopup` para o detalhe; e relatórios para os mapas periódicos.

## 6. Sistema de design

- **Fundação de *tokens*** (cor, tipografia, espaçamento, raio, sombra) partilhada entre
  portal e back-office, definida em CSS custom properties. Suporte a tema claro e escuro.
- **Paleta** que sustente a promessa: sóbria e legível, com uma cor de acento para ações e
  cores de estado inequívocas (pendente / validado / precisa de ti). Estado **nunca**
  comunicado só por cor — sempre cor + ícone + texto.
- **Componentes partilhados** numa biblioteca de classes Razor (`Cifra.Ui`): cartão de
  documento, cartão de lançamento, bolha de conversa (com variante assistente/humano),
  cartão de obrigação, estado de ligação, banner de aviso legal.
- **Storybook equivalente**: uma página `/dev/componentes` no ambiente de desenvolvimento com
  todos os componentes em todos os estados, incluindo estados de erro e de carregamento —
  é o que evita divergência entre as duas aplicações.
