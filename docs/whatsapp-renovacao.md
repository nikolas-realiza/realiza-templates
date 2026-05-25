# Cadência de Renovação no WhatsApp (Z-API + n8n + Notion)

Implementa três mensagens automáticas (D+0, D+3, D+7) para acelerar o
fechamento da renovação contratual. As respostas do cliente são classificadas
por LLM e refletidas no Notion (cancelando a cadência quando há aceite).

## 1. Arquitetura

```
Notion (status = "Em renovação")
        │  automation → webhook
        ▼
WF1: cadencia-renovacao-init       envia M1, agenda Próximo Disparo=+3d
                                   (executa uma vez por cliente)

Schedule diário 09:00 BRT
        ▼
WF2: cadencia-renovacao-daily      varre Notion, dispara M2/M3 conforme etapa

Z-API "Ao receber mensagem"
        │  webhook
        ▼
WF3: cadencia-renovacao-reply      LLM classifica resposta e atualiza Notion
```

Os três workflows não se chamam diretamente — coordenam pelo Notion.

## 2. Pré-requisitos

### 2.1 Conta Z-API

1. Criar conta em https://z-api.io, criar uma **instância** e conectar o
   número do WhatsApp via QR Code no painel.
2. Anotar `INSTANCE_ID` e `TOKEN` (ambos visíveis na URL de chamada da
   instância).
3. Em **Segurança da Conta**, ativar o **Account Security Token** e copiar o
   valor — ele será enviado no header `Client-Token` em toda chamada (boa
   prática; sem ele a API aceita qualquer requisição com o token da
   instância).
4. Em **Webhooks → Ao receber mensagem**, colar a URL pública do webhook do
   WF3 (ex.: `https://n8n.realiza.com/webhook/whatsapp-reply`) e marcar
   "Notificar apenas mensagens recebidas".

### 2.2 Variáveis de ambiente no n8n

Adicionar em **Settings → Environment Variables** (ou no `.env` do
self-hosted):

| Variável            | Conteúdo                                          |
|---------------------|---------------------------------------------------|
| `ZAPI_INSTANCE`     | ID da instância Z-API.                            |
| `ZAPI_TOKEN`        | Token da instância (parte da URL).                |
| `ZAPI_CLIENT_TOKEN` | Account Security Token (header `Client-Token`).   |

Reaproveitar credenciais existentes:

- **Notion** — a mesma usada pelo CRM2.0.
- **OpenAI** (ou Anthropic) — para o classificador do WF3.

Após importar os JSONs no n8n, substituir os placeholders:

- `REPLACE_NOTION_CRED_ID` → ID da credencial Notion (clicar no node Notion
  e selecionar a credencial existente).
- `REPLACE_NOTION_DATABASE_ID` → ID do database de Clientes/Renovações.
- `REPLACE_OPENAI_CRED_ID` → ID da credencial OpenAI.

## 3. Propriedades novas no Notion

Adicionar no database de Clientes/Renovações:

| Propriedade           | Tipo        | Notas                                                       |
|-----------------------|-------------|-------------------------------------------------------------|
| `Telefone WhatsApp`   | Phone       | E.164 sem `+` — ex.: `5511999990000`.                       |
| `Status Renovação`    | Select      | Opções: `Em renovação`, `Aceito`, `Recusou`, `Sem resposta`.|
| `Etapa Cadência WPP`  | Number      | `0`=não iniciado, `1`=M1, `2`=M2, `3`=M3.                   |
| `Próximo Disparo WPP` | Date        | Data da próxima mensagem (D+3 ou D+7).                      |
| `Aceite WPP`          | Checkbox    | Marcado pelo WF3 quando LLM classifica como `aceite`.       |
| `Última Resposta WPP` | Rich text   | Snapshot da última resposta recebida (auditoria).           |
| `ID Renovação`        | Formula/Text| Identificador único (opcional, usado como `clientMessageId`).|

Configurar a **automação do Notion**: quando `Status Renovação` mudar para
`Em renovação`, chamar `POST https://n8n.realiza.com/webhook/cadencia-renovacao`
com body `{ "pageId": "<id da página>" }`.

## 4. Mensagens (templates)

Os textos finais ficam no nó **Code** ("Monta M1" no WF1, "Planeja envios"
no WF2). Variáveis lidas do Notion: `nome`, `plano`, `valor`, `vencimento`,
`linkProposta`.

### M1 (D+0) — Convite

```
Olá {{nome}}, aqui é da Realiza Consultoria.

Sua renovação do plano {{plano}} está pronta no valor de {{valor}}, com
vencimento em {{vencimento}}.

Proposta: {{linkProposta}}

Posso seguir com a renovação? Responda *sim* para confirmar ou me conte se
prefere conversar antes.
```

### M2 (D+3) — Lembrete

```
Oi {{nome}}, passando para reforçar a renovação do {{plano}} ({{valor}}),
com vencimento em {{vencimento}}.

Proposta: {{linkProposta}}

Consegue me dar um *sim* ou *não* para destravarmos por aqui?
```

### M3 (D+7) — Última chamada

```
{{nome}}, esta é nossa última tentativa por aqui antes de pausar a renovação
do {{plano}}.

Se quiser seguir, basta responder *sim* — caso contrário, posso te chamar
por outro canal?
```

## 5. Payload de referência

### 5.1 Envio (n8n → Z-API)

`POST https://api.z-api.io/instances/{INSTANCE}/token/{TOKEN}/send-text`

Headers:
```
Content-Type: application/json
Client-Token: <ZAPI_CLIENT_TOKEN>
```

Body:
```json
{
  "phone": "5511999990000",
  "message": "Olá Nikolas, aqui é da Realiza Consultoria..."
}
```

### 5.2 Webhook de recebimento (Z-API → n8n WF3)

A Z-API envia algo como:

```json
{
  "instanceId": "3D...",
  "messageId": "ABCD",
  "phone": "5511999990000",
  "fromMe": false,
  "momment": 1716636000000,
  "senderName": "Cliente",
  "text": { "message": "sim, pode renovar" },
  "type": "ReceivedCallback"
}
```

O WF3 lê `phone`, `fromMe` e `text.message`. Outros tipos (`image`, `audio`,
`document`) são ignorados na v1 — o nó `IF processar?` exige texto não vazio.

## 6. Verificação ponta-a-ponta

1. Criar página de teste no Notion com seu próprio número em
   `Telefone WhatsApp`, mudar `Status Renovação` para `Em renovação`.
2. Confirmar no WhatsApp o recebimento da M1; no Notion, `Etapa Cadência WPP = 1`
   e `Próximo Disparo WPP = hoje + 3d`.
3. Editar manualmente `Próximo Disparo WPP` para hoje e executar o WF2 via
   "Execute Workflow" no n8n → M2 deve chegar, etapa avança para `2` e
   próximo disparo vira `hoje + 4d`.
4. Repetir o passo 3 → M3 deve chegar, etapa vira `3`, status muda para
   `Sem resposta`.
5. Responder no WhatsApp com cada uma das classes para validar o WF3:
   - `"sim, pode renovar"` → `Aceite WPP = true`, status `Aceito`.
   - `"não quero renovar"` → status `Recusou`.
   - `"qual o valor exato?"` → `Última Resposta WPP` recebe prefixo `[DÚVIDA]`.
   - `"obrigado"` → `Última Resposta WPP` recebe prefixo `[OUTRO]`.
6. Após marcar `Aceite WPP = true`, rodar o WF2 e confirmar que a página não
   aparece mais no filtro (cadência cancelada).

## 7. Operação

- **Pausa global**: desative o workflow `cadencia-renovacao-daily` no n8n.
- **Pausar um cliente**: limpe `Próximo Disparo WPP` ou mude `Status Renovação`
  para qualquer valor diferente de `Em renovação`.
- **Reenviar M1**: zere `Etapa Cadência WPP` e dispare manualmente o webhook
  do WF1 com `{ "pageId": "<id>" }`.
- **Auditoria**: `Última Resposta WPP` guarda a última mensagem recebida com
  prefixo da classe (quando aplicável).
