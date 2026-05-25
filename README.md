# realiza-templates

Templates de automações da Realiza Consultoria (n8n + Notion + integrações).

## Templates disponíveis

### Cadência de Renovação no WhatsApp (Z-API)

Três workflows do n8n que dirigem uma cadência de mensagens (D+0, D+3, D+7)
para clientes em processo de renovação contratual, com leitura de respostas
classificada por LLM.

- [`n8n/cadencia-renovacao-init.json`](n8n/cadencia-renovacao-init.json) —
  webhook do Notion → envia M1, agenda a próxima.
- [`n8n/cadencia-renovacao-daily.json`](n8n/cadencia-renovacao-daily.json) —
  schedule diário → dispara M2 e M3 conforme a etapa.
- [`n8n/cadencia-renovacao-reply.json`](n8n/cadencia-renovacao-reply.json) —
  webhook Z-API → LLM classifica resposta e atualiza o Notion.
- [`docs/whatsapp-renovacao.md`](docs/whatsapp-renovacao.md) — variáveis de
  ambiente, propriedades do Notion, exemplos de payload e checklist de
  verificação.
