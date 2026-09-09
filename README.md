# Serviço 08 — Cadastro / Identidade

Projeto acadêmico desenvolvido para a disciplina de **Arquitetura de Serviços em Nuvem**, ministrada pelo professor **Andrews Fernandes Egas**. Esta API é o Serviço 08 do ecossistema distribuído **PzaaS** e é responsável pelo cadastro de clientes, consulta de perfil, validação de token e gestão de carteira/saldo.

O PzaaS simula uma pizzaria construída por serviços independentes. O Serviço 08 integra-se ao Gateway, Pagamento, Chaos Monkey e Logger.

## URLs de produção

Este serviço possui Webhooks publicados pelo n8n/PzaaS. Use as URLs abaixo exatamente como estão. Não existe uma única Base URL para todos os endpoints, pois a consulta de cliente possui um UUID próprio na rota.

| Operação | Método | URL de produção |
|---|---|---|
| Healthcheck | `GET` | `https://pzaas.online/webhook/cadastro/health` |
| Criar cliente | `POST` | `https://pzaas.online/webhook/cadastro/v1/clientes` |
| Consultar cliente | `GET` | `https://pzaas.online/webhook/7c5b0659-106a-49cd-a03f-1fcffbec089b/cadastro/v1/clientes/{clienteId}` |
| Debitar saldo | `POST` | `https://pzaas.online/webhook/cadastro/v1/carteiras/debito` |
| Validar token | `POST` | `https://pzaas.online/webhook/cadastro/v1/auth/validar` |

> O UUID presente na URL de consulta de cliente faz parte daquele Webhook. Não o remova. Em produção, o workflow deve estar **ativo**. URLs com `/webhook-test/` são apenas para testes e exigem executar o workflow antes de cada chamada.

## Padrões globais

- Protocolo: HTTP/JSON
- Serviço: `8` — Cadastro/Identidade
- ID do cliente: número inteiro sequencial (`1`, `2`, `3`...)
- Identificador de rastreamento: `x-pedido-id`, gerado pelo API Gateway e propagado pelos serviços

### Headers obrigatórios

| Header | Valor |
|---|---|
| `Content-Type` | `application/json` |
| `x-api-key` | `turma2026` |
| `x-pedido-id` | ID da requisição/pedido, por exemplo `PED-001` |

## Healthcheck

### `GET https://pzaas.online/webhook/cadastro/health`

Verifica se o Serviço de Cadastro/Identidade e o Redis estão disponíveis. É o endpoint utilizado pelo Chaos Monkey para verificar disponibilidade.

Resposta `200 OK`:

```json
{
  "status": "UP",
  "service": "cadastro-identidade",
  "redis": "UP",
  "timestamp": "2026-09-07T12:00:00.000Z"
}
```

## Criar cliente

### `POST https://pzaas.online/webhook/cadastro/v1/clientes`

Cria um novo cliente, gera o ID sequencial e um token de autenticação.

Body:

```json
{
  "nome": "Cliente Teste",
  "email": "cliente.teste@email.com",
  "senha": "senhaTeste123",
  "saldoInicial": 100
}
```

Resposta `201 Created`:

```json
{
  "id": 1,
  "nome": "Cliente Teste",
  "email": "cliente.teste@email.com",
  "saldo": 100,
  "token": "token-gerado-pelo-servico",
  "criadoEm": "2026-09-07T12:00:00.000Z"
}
```

O token é devolvido somente no cadastro. A senha e o hash da senha nunca são retornados.

## Consultar cliente

### `GET https://pzaas.online/webhook/7c5b0659-106a-49cd-a03f-1fcffbec089b/cadastro/v1/clientes/{clienteId}`

Confirma a existência de um cliente e devolve seu perfil público. Este endpoint permanece disponível para consumidores que precisarem consultar um cliente.

Exemplo:

```http
GET https://pzaas.online/webhook/7c5b0659-106a-49cd-a03f-1fcffbec089b/cadastro/v1/clientes/1
```

Resposta `200 OK`:

```json
{
  "id": 1,
  "nome": "Cliente Teste",
  "email": "cliente.teste@email.com",
  "existe": true
}
```

## Debitar saldo

### `POST https://pzaas.online/webhook/cadastro/v1/carteiras/debito`

Operação usada pelo Serviço de Pagamento. O Serviço 08 consulta o saldo, verifica se é suficiente e, se aprovado, realiza o débito.

Body:

```json
{
  "clienteId": 1,
  "valor": 30,
  "pagamentoId": "pag-001",
  "tipo": "DEBITO"
}
```

Resposta aprovada `200 OK`:

```json
{
  "aprovado": true,
  "clienteId": 1,
  "pagamentoId": "pag-001",
  "valor": 30,
  "saldoAtual": 70,
  "tipo": "DEBITO"
}
```

Resposta de saldo insuficiente `400 Bad Request`:

```json
{
  "aprovado": false,
  "error": "INSUFFICIENT_FUNDS",
  "message": "Saldo insuficiente para concluir o pagamento"
}
```

### Repetição de pagamento

O campo `tipo` é opcional e assume `DEBITO` quando não enviado. Os valores aceitos são `DEBITO` e `ESTORNO`.

O `pagamentoId` combinado com `tipo` evita duplicidade em retries. Repetir a mesma chamada com o mesmo `pagamentoId`, `tipo` e valor devolve o resultado já registrado, sem alterar o saldo novamente. O mesmo pagamento pode ser debitado e depois estornado.

Se o mesmo `pagamentoId` for enviado com valor diferente, a API responde `400 Bad Request`:

```json
{
  "aprovado": false,
  "error": "PAYMENT_ID_CONFLICT",
  "message": "pagamentoId já foi usado com dados diferentes"
}
```

## Validar token

### `POST https://pzaas.online/webhook/cadastro/v1/auth/validar`

Valida um token previamente recebido no cadastro e devolve o perfil associado.

Body:

```json
{
  "token": "token-gerado-pelo-servico"
}
```

Resposta válida `200 OK`:

```json
{
  "statusCode": 200,
  "id": 1,
  "nome": "Cliente Teste",
  "email": "cliente.teste@email.com"
}
```

Resposta de token inválido `401 Unauthorized`:

```json
{
  "statusCode": 401,
  "error": "UNAUTHORIZED",
  "message": "Token inválido"
}
```

## Erros padronizados

| HTTP | Código no body | Situação |
|---:|---|---|
| 400 | `BAD_REQUEST` | Payload inválido, saldo insuficiente ou conflito de `pagamentoId` |
| 401 | `UNAUTHORIZED` | `x-api-key` ausente ou token inválido |
| 403 | `FORBIDDEN` | `x-api-key` inválida |
| 404 | `NOT_FOUND` | Cliente não encontrado |
| 500 | `INTERNAL_ERROR` | Erro interno ao processar dados |
| 503 | `REDIS_INDISPONÍVEL` | Redis temporariamente indisponível |

## Observabilidade

O serviço envia logs estruturados ao Serviço 09 — Logger por meio de:

```http
POST https://pzaas.online/webhook/v1/log
```

Cada log usa o mesmo `x-pedido-id` da operação e contém:

```json
{
  "eventId": "PED-001",
  "service": "8",
  "action": "DEBIT_WALLET",
  "status": "SUCCESS",
  "level": "INFO",
  "message": "Saldo debitado com sucesso",
  "metadata": {
    "clienteId": 1,
    "pagamentoId": "pag-001",
    "valor": 30,
    "saldoAtual": 70
  }
}
```

As ações registradas atualmente são: `HEALTH_CHECK`, `CREATE_CLIENT`, `GET_CLIENT`, `DEBIT_WALLET`, `REFUND_WALLET` e `VALIDATE_TOKEN`.

### Métricas

Além dos logs, o serviço envia métricas para:

```http
POST https://pzaas.online/webhook/v1/metric
```

As métricas enviadas pelo Serviço 08 são:

| Métrica | Quando é enviada | Exemplo de resultado |
|---|---|---|
| `client_created_total` | Cliente criado com sucesso | `1` com unidade `count` |
| `wallet_operation_total` | Débito ou estorno aprovado/recusado | `1` com resultado em `metadata` |

Exemplo de métrica de operação de carteira:

```json
{
  "metricName": "wallet_operation_total",
  "value": 1,
  "unit": "count",
  "service": "8",
  "orderId": "PED-001",
  "metadata": {
    "tipo": "DEBITO",
    "resultado": "SUCCESS"
  }
}
```

Tanto logs quanto métricas usam `x-api-key: turma2026`. Quando a operação estiver ligada a um pedido, o mesmo `x-pedido-id` é enviado para manter o rastreamento distribuído.

## Exemplo completo de integração

### Requisição

```http
POST https://pzaas.online/webhook/cadastro/v1/carteiras/debito
Content-Type: application/json
x-api-key: turma2026
x-pedido-id: PED-001
```

```json
{
  "clienteId": 1,
  "valor": 30,
  "pagamentoId": "pag-001",
  "tipo": "DEBITO"
}
```

### Resposta

```http
200 OK
```

```json
{
  "aprovado": true,
  "clienteId": 1,
  "pagamentoId": "pag-001",
  "valor": 30,
  "saldoAtual": 70,
  "tipo": "DEBITO"
}
```
