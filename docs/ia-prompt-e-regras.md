# Especificação da IA e do fallback por regras

## Requisição (OpenAI_IS → `POST https://api.openai.com/v1/chat/completions`)

Cabeçalho: `Authorization: Bearer {Site.OpenAI_ApiKey}` (Site Property, nunca versionada).

```json
{
  "model": "gpt-4o-mini",
  "temperature": 0.2,
  "max_tokens": 300,
  "response_format": { "type": "json_object" },
  "messages": [
    { "role": "system", "content": "<PROMPT_SISTEMA>" },
    { "role": "user",   "content": "<CONTEXTO_DO_LEAD>" }
  ]
}
```

### PROMPT_SISTEMA
```
Você é um analista comercial B2B. Avalie a chance de um lead fechar negócio.
Responda SOMENTE um JSON com as chaves:
score (inteiro 0-100), temperatura ("Quente","Morno" ou "Frio"),
resumo (no máximo 2 frases), proxima_acao (uma ação concreta e curta),
justificativa (1 frase explicando o score).
Considere: cargo com poder de decisão, origem, etapa do funil, valor estimado,
tempo sem contato e o conteúdo das interações. Não invente dados.
```

### CONTEXTO_DO_LEAD (montado em `LeadPulse_AI_BL`; sem nome, e-mail ou telefone)
```
Cargo: {Lead.Cargo}
Empresa: {Lead.Empresa}
Origem: {Origem.Label}
Etapa: {Etapa.Label}
Valor estimado: R$ {Lead.ValorEstimado}
Dias sem interação: {DiffDays(Lead.UltimaInteracaoEm, CurrDate())}
Últimas interações:
- {DataHora} [{TipoInteracao.Label}] {Descricao}
...
```

## Codificação (UTF-8)
A API responde em UTF-8, mas o Content-Type não declara o charset, e a decodificação padrão quebrava os acentos ("estÃ¡"). A solução tem duas partes:
- `ChatCompletions` recebe a resposta como **Binary**. `IA_GerarJSON` decodifica explicitamente em UTF-8 (`BinaryDataToText`) e só então desserializa o JSON.
- O Core aplica um saneamento nos campos textuais (`Replace` das sequências típicas de UTF-8 lidas como Latin-1). Ele é idempotente: num texto que já está correto, não altera nada.

> Lição de OutSystems 11: Server Actions de outro módulo são compiladas **dentro** do consumidor. Depois de alterar `OpenAI_IS` ou `LeadPulse_CS`, é preciso atualizar as referências e republicar os consumidores, até a Interface.

## Validação da resposta
- `score` inteiro entre 0 e 100.
- `temperatura` precisa estar em {Quente, Morno, Frio}.
- `proxima_acao` com texto não vazio.

Se qualquer item falhar, se der erro HTTP ou se a Site Property estiver vazia, usa o fallback.

## Fallback por regras (`Lead_ScorePorRegra`)

| Fator | Pontos |
|---|---|
| Origem: Indicação 30 · Evento 25 · Site 20 · LinkedIn 15 · Prospecção ativa 10 · Anúncio 10 | 10–30 |
| Etapa: Novo 5 · Qualificação 15 · Proposta 25 · Negociação 35 | 5–35 |
| Interações nos últimos 7 dias: 5 por interação (máx. 20) | 0–20 |
| Valor estimado ≥ R$ 50.000 → 15 · ≥ R$ 10.000 → 10 · > 0 → 5 | 0–15 |
| Penalidade: −2 por dia sem interação acima de 7 (máx. −30) | −30–0 |

`score = clamp(soma, 0, 100)` → Temperatura: ≥ 70 Quente · 40–69 Morno · < 40 Frio.

Próxima ação por regra:

| Situação | Ação |
|---|---|
| Etapa Novo | "Fazer primeiro contato e qualificar necessidade" |
| Etapa Qualificação | "Agendar reunião de diagnóstico" |
| Etapa Proposta | "Enviar/ajustar proposta e confirmar decisor" |
| Etapa Negociação | "Negociar condições e definir data de fechamento" |
| Mais de 7 dias sem interação | "Retomar contato — lead esfriando" (sobrepõe as anteriores) |
