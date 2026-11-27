# LeadPulse: parte teórica

> Aplicação Inteligente com Low-Code: gestão de leads com IA, construída em OutSystems 11 (Reactive Web) com Arquitetura Canvas.

---

## 1. Problema, público e solução proposta

### Problema
Em uma empresa de serviços B2B de pequeno/médio porte, o acompanhamento comercial acontecia em uma **planilha compartilhada**, em **conversas de WhatsApp** e na memória de cada vendedor. Na prática isso gerava:

- **Leads esquecidos**: ninguém percebia quando um contato ficava semanas sem retorno.
- **Prioridade por intuição**: cada vendedor decidia sozinho quem atender primeiro, sem critério comum.
- **Duplicidade e erros**: o mesmo lead cadastrado duas vezes, linhas sobrescritas, etapas inconsistentes ("Ganho" sem valor, "Perdido" sem motivo).
- **Gestor sem visão**: para saber a taxa de conversão ou o valor em negociação era preciso consolidar a planilha manualmente.
- **Risco de LGPD**: dados pessoais de contatos sem registro de consentimento e sem política de retenção.

### Público
| Perfil | Necessidade |
|---|---|
| **Vendedor** (role `Vendedor`) | Saber *quem atender agora* e *o que fazer a seguir*; registrar interações rapidamente. |
| **Gestor comercial** (role `Gestor`) | Visão consolidada do funil, conversão, valor em pipeline e leads parados. |

### Solução proposta
O **LeadPulse** centraliza o funil comercial em uma aplicação web reativa:

1. Cadastro único de leads com validação de duplicidade e consentimento LGPD.
2. Funil com etapas controladas por regras (Novo → Qualificação → Proposta → Negociação → Ganho/Perdido).
3. Registro de interações (ligação, e-mail, reunião, WhatsApp, nota) com histórico completo.
4. **IA de qualificação**: a cada novo lead, nova interação ou mudança de etapa, o sistema calcula um **score (0–100)**, uma **temperatura** (Quente/Morno/Frio), um **resumo** e a **próxima ação recomendada**, com a justificativa.
5. **Automação por Timers**: diariamente os leads parados há 7+ dias são "esfriados" e recebem uma ação de retomada; leads perdidos há mais de 12 meses são anonimizados.
6. **Dashboard** do gestor com indicadores e gráficos do funil.

---

## 2. Arquitetura da aplicação (Arquitetura Canvas)

A solução tem **3 módulos próprios** mais componentes do Forge, distribuídos nas três camadas do Canvas. A regra seguida foi: *as referências só descem de camada* (Interface → Core → Foundation), nunca sobem nem ficam laterais.

```
┌──────────────────────────── INTERFACE (End-User) ────────────────────────────┐
│  LeadPulse  (Reactive Web App)                                               │
│  Telas: Meus leads (inicial) · Equipe* · Dashboard* · LeadForm · LeadDetalhe │
│  (* restritas à role Gestor no nível da tela)                                │
└───────────────────────────────────┬──────────────────────────────────────────┘
                                    │ entidades read-only + server actions públicas + roles
┌───────────────────────────────────▼──────────────── CORE ────────────────────┐
│  LeadPulse_CS (Service)                                                      │
│   • Entidades e static entities (dados do domínio)                           │
│   • Regras de negócio: Lead_Criar, Lead_Atualizar, Lead_MoverEtapa,          │
│     Interacao_Registrar                                                      │
│   • Inteligência: Lead_Qualificar (IA → validação → fallback),               │
│     Lead_ScorePorRegra, Lead_AplicarQualificacao                             │
│   • Automação: Timer_LeadsParados, Timer_RetencaoLGPD · Roles Gestor/Vendedor│
└───────────────────────────────────┬──────────────────────────────────────────┘
                                    │ IA_GerarJSON
┌───────────────────────────────────▼───────────── FOUNDATION ─────────────────┐
│  OpenAI_IS (Service) - Consume REST API (Chat Completions, JSON mode), DTOs, │
│                        Site Properties (chave secreta e modelo)              │
│  Forge: OutSystems UI (padrões de tela, Card) · OutSystems Charts (gráficos) │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Decisões e porquês
| Decisão | Motivo |
|---|---|
| Entidades **somente no Core**, expostas como *read-only*. A escrita acontece só pelas Server Actions públicas (`Lead_Criar`, `Lead_MoverEtapa`…). | Nenhuma tela consegue gravar sem passar pelas regras (duplicidade, transições, consentimento). |
| A **integração com a IA é Foundation** (`OpenAI_IS`) e **não tem regra de negócio**: recebe um prompt e devolve texto, sem lançar exceção (falhas voltam em `Sucesso`/`MensagemErro`). | É técnica e reutilizável. Trocar de provedor afeta só esse módulo. |
| A **decisão sobre o resultado da IA fica no Core** (`Lead_Qualificar`): monta o contexto, valida a resposta e decide entre IA e regra. | O "o que fazer com a resposta" é regra de negócio e precisa ficar junto das demais regras do lead. |
| Timers no Core. | A automação é regra de negócio (quando um lead "esfria"), não de interface. |
| Telas de visão global (Dashboard/Equipe) com **role Gestor no nível da tela**. | A plataforma valida a role no servidor ao servir os dados da tela, então não dá para burlar pelo navegador. |
| Componentes do **Forge com suporte oficial** (OutSystems UI e OutSystems Charts). | Menos risco de manutenção. Os gráficos recebem os dados já agrupados no banco. |

---

## 3. Modelagem das entidades e relacionamentos

```
   Origem (S)      Etapa (S)      Temperatura (S)     FonteScore (S)
       │1              │1               │1                  │1
       │N              │N               │N                  │N
User ──┴───────1:N─────┴── Lead ────────┴───────────────────┘
(System)                    │1
             ┌──────────────┼──────────────────┐
             │N             │N                 │N
        Interacao     HistoricoEtapa        AnaliseIA
      (TipoInteracao)  (de → para)        (log de cada qualificação)
                (S) = Static Entity
```

| Entidade | Atributos principais | Observações |
|---|---|---|
| **Lead** | Nome, Empresa, Cargo, Email, Telefone, OrigemId, EtapaId, TemperaturaId, ResponsavelId (User), ValorEstimado, Score, FonteScoreId, ResumoIA, ProximaAcao, JustificativaScore, MotivoPerda, ConsentimentoLGPD, DataConsentimento, UltimaInteracaoEm, DataFechamento, CriadoEm, AtualizadoEm, IsAnonimizado | Índices `IX_Lead_Email` (duplicidade) e `IX_Lead_EtapaResp` (filtros do funil). Update Behavior *Optimized*. |
| **Interacao** | LeadId, TipoInteracaoId, Descricao, DataHora, UsuarioId | Delete rule *Delete*; índice (LeadId, DataHora). |
| **HistoricoEtapa** | LeadId, EtapaAnteriorId, EtapaNovaId, DataHora, UsuarioId, Observacao | Trilha de auditoria do workflow. |
| **AnaliseIA** | LeadId, DataHora, FonteScoreId, Modelo, Score, TemperaturaId, Resumo, ProximaAcao, Justificativa, Sucesso, MensagemErro, TokensUsados | Transparência: cada qualificação fica registrada e explicável, inclusive o motivo de um fallback. |
| **Etapa** (S) | Label, Ordem, IsFinal, Cor, PesoScore | Novo(1), Qualificação(2), Proposta(3), Negociação(4), Ganho(5, final), Perdido(6, final). |
| **Origem** (S) | Label, Ordem, PesoScore | Site, Indicação, LinkedIn, Evento, Anúncio, Prospecção ativa. |
| **Temperatura** (S) | Label, Ordem, Icone, Cor | Quente, Morno, Frio. |
| **TipoInteracao** (S) | Label, Ordem, Icone | Ligação, E-mail, Reunião, WhatsApp, Nota. |
| **FonteScore** (S) | Label, Ordem | IA, Regra, Manual. Mostra na tela quem definiu o score. |
| **RespostaIA** (estrutura) | score, temperatura, resumo, proxima_acao, justificativa | Contrato JSON exigido da IA. |

Etapas e origens são **Static Entities** de propósito: são listas controladas que participam de regras (ordem do funil, peso no score) e que os usuários não devem editar.

---

## 4. Regras de negócio e workflows

| # | Regra | Onde é aplicada |
|---|---|---|
| RN01 | E-mail obrigatório, com formato válido e **único** entre os leads não anonimizados. | `Lead_Criar` / `Lead_Atualizar` |
| RN02 | Só é possível salvar um lead com **consentimento LGPD**. A data do consentimento fica registrada. | `Lead_Criar` |
| RN03 | O lead avança **uma etapa por vez** e pode retroceder. **Ganho** e **Perdido** são finais e só o Gestor reabre. | `Lead_MoverEtapa` |
| RN04 | **Ganho** exige valor maior que 0. **Perdido** exige `MotivoPerda`. | `Lead_MoverEtapa` |
| RN05 | Toda mudança de etapa grava `HistoricoEtapa` e atualiza `UltimaInteracaoEm`. | `Lead_MoverEtapa` |
| RN06 | Novo lead, nova interação ou mudança para etapa aberta disparam a **requalificação** (IA, com fallback por regras). | `Lead_Criar`, `Interacao_Registrar`, `Lead_MoverEtapa` → `Lead_Qualificar` |
| RN07 | O vendedor altera apenas os próprios leads e o Gestor altera todos. "Meus leads" lista só os do usuário. Dashboard e Equipe são exclusivos do Gestor. | `CheckGestorRole` nas Server Actions + roles das telas |
| RN08 | Lead em etapa aberta sem interação há **7+ dias** → Temperatura = Frio, Score −10, próxima ação "Retomar contato". | `Timer_LeadsParados` (diário, 07:00; lotes de 200) |
| RN09 | Lead **Perdido há mais de 12 meses** tem nome, e-mail, telefone e cargo anonimizados. | `Timer_RetencaoLGPD` (diário, 03:00; lotes de 200) |

Todas as violações geram a exceção de negócio `RegraNegocio`, com uma mensagem clara que a interface mostra ao usuário (por exemplo: "É necessário o consentimento LGPD do contato para cadastrá-lo.").

### Workflow principal
```
 Cadastro ─► [Lead_Criar] ─► RN01/RN02 ─► Lead (Etapa=Novo) + HistoricoEtapa ─► [Lead_Qualificar]
                                                                                   │
              ┌──────────────────────────── IA configurada e resposta válida? ─────┤
              │sim                                                                 │não / erro / exceção
              ▼                                                                    ▼
   grava score/resumo/ação (Fonte=IA)                             Lead_ScorePorRegra (Fonte=Regra)
              └───────────────────────► AnaliseIA (log, tokens, motivo) ◄──────────┘

 Interação ─► [Interacao_Registrar] ─► atualiza última interação ─► requalifica
 Mudança de etapa ─► [Lead_MoverEtapa] ─► RN03/RN04/RN05 ─► (etapa aberta) requalifica
 Timers ─► RN08 (07:00) · RN09 (03:00)
```

---

## 5. Como a IA e a automação foram incorporadas

### 5.1 Qualificação por IA (`LeadPulse_CS.Lead_Qualificar`)
1. Monta um **contexto mínimo**: cargo, empresa, origem, etapa, valor estimado, dias sem interação e as **5 últimas interações**. Nome, e-mail e telefone **não** entram no contexto.
2. Chama `OpenAI_IS.IA_GerarJSON`, que faz `POST /v1/chat/completions` com o modelo `gpt-4o-mini`, `temperature 0.2`, `max_tokens 300` e `response_format = json_object`.
3. O prompt exige um JSON estrito, desserializado na estrutura `RespostaIA`:
   ```json
   {"score": 0-100, "temperatura": "Quente|Morno|Frio",
    "resumo": "até 2 frases", "proxima_acao": "ação concreta",
    "justificativa": "por que este score"}
   ```
4. A resposta é **validada**: o score precisa estar entre 0 e 100, a temperatura dentro da lista e a ação não pode vir vazia. O fallback por regras entra em três situações:
   - a chave não está configurada;
   - a API falha ou devolve JSON inválido;
   - acontece qualquer exceção (há um exception handler próprio).

   O fallback calcula: `score = peso da origem + peso da etapa + 5 × interações nos últimos 7 dias (máx. 20) + bônus por valor (5/10/15) − 2 × dias parado acima de 7 (máx. 30)`.
5. O resultado é gravado no Lead e em `AnaliseIA`, com fonte, modelo, justificativa, tokens e o motivo do fallback.

### 5.2 Automação (Timers)
- `Timer_LeadsParados` (07:00) aplica a RN08 e `Timer_RetencaoLGPD` (03:00) aplica a RN09, ambos em lotes de até 200 registros.
- A ação `Demo_GerarDados`, exclusiva do Gestor e disponível só com a base vazia, cria 12 leads com interações e histórico e qualifica cada um. Ela é usada na demonstração.

### 5.3 Princípios adotados
- **A IA sugere, a pessoa decide.** O card de qualificação mostra o selo "🤖 Sugerido por IA", "📏 Calculado por regras" ou "✍️ Ajuste manual", junto com a justificativa. A temperatura pode ser ajustada manualmente, e nesse caso a fonte passa a ser "Manual".
- **A IA não é ponto único de falha.** Sem a chave, ou com a API fora, o app continua funcionando, e o histórico de qualificações mostra o motivo.

---

## 6. Acessibilidade, dados e uso responsável

### Acessibilidade
- Os recursos de acessibilidade do layout do OutSystems UI estão habilitados (`EnableAccessibilityFeatures`).
- Todos os campos têm `Label` associado ao input (`TargetWidget`). Os campos obrigatórios são marcados e validados.
- A temperatura **nunca depende só de cor**: o rótulo tem ícone e texto ("🔥 Quente", "🌤️ Morno", "❄️ Frio").
- Cada gráfico tem um **resumo textual** logo abaixo, e os KPIs aparecem em texto.
- O layout é responsivo (flex-wrap). O app funciona no celular, o que importa para vendedores em campo.

### Dados e LGPD
- **Minimização**: nome, e-mail e telefone do lead não são enviados à IA. O aviso aparece no próprio formulário de cadastro.
- **Consentimento** obrigatório e datado (RN02). **Retenção** limitada com anonimização automática (RN09).
- A chave da API fica numa **Site Property secreta** (`OpenAI_ApiKey`), configurada no Service Center. Ela não fica no código nem no repositório, e o header `Authorization` é marcado como sensível nos logs.
- **Autorização** no servidor: toda escrita verifica dono ou Gestor (RN07), e as telas globais exigem a role Gestor.
- **Limitação conhecida (melhoria futura):** a leitura do detalhe de um lead por Id ainda não é filtrada por responsável. O próximo passo é servir o detalhe por uma Data Action que aplique a RN07 também na leitura.

### Desempenho e sustentabilidade digital
- A IA é chamada **apenas em eventos** (cadastro, interação, mudança de etapa), nunca ao abrir telas. O resultado fica no próprio Lead e funciona como cache.
- Modelo pequeno, `max_tokens` limitado e consumo de tokens registrado em `AnaliseIA`.
- Os aggregates agrupam no banco (o dashboard recebe só os totais), usam `Max Records` e índices nos campos de filtro. As entidades grandes usam Update Behavior *Optimized*.
- Os timers processam em lotes, fora do horário comercial.
