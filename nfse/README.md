# Quick Start - NFS-e via Token API (DropNotas)

Guia rapido para emissao e consulta de NFS-e usando `X-AUTH-TOKEN` nas rotas de `nfse`.

## 1) Como funciona a autenticacao

- Para gerar/revogar/rotacionar token, use `Authorization: Bearer <access_token_web_app>`.
- Para operar `POST /nfse` e consultas de NFS-e, use `X-AUTH-TOKEN: <token_api>`.
- O token de integracao e retornado em texto puro e armazenado internamente em Base64.

```http
X-AUTH-TOKEN: <TOKEN_EXATAMENTE_COMO_RECEBIDO>
```

## 2) Endpoints de token

| Acao | Metodo e rota | Retorno |
|---|---|---|
| Gerar token | `POST /integracao/token-api` | `{ "token": "..." }` |
| Revogar token | `DELETE /integracao/token-api` | `204 No Content` |
| Rotacionar token | `PATCH /integracao/token-api/recriar` | `{ "token": "..." }` |

Exemplo em PowerShell:

```powershell
$BASE_URL = "https://backend.dropnotas.com"
$ACCESS_TOKEN = "<access_token_web_app>"

$tokenResponse = Invoke-RestMethod -Method Post -Uri "$BASE_URL/integracao/token-api" -Headers @{
  Authorization = "Bearer $ACCESS_TOKEN"
}

$API_TOKEN = $tokenResponse.token
```

## 3) Pre-requisitos para emitir

Antes do `POST /nfse`, a empresa deve estar configurada (certificado, endereco e parametros fiscais).

- Cadastro: `POST /empresa`
- Alteracao: `PUT /empresa`

Sem configuracao fiscal/certificado validos, a prefeitura pode rejeitar a emissão.

## 4) Endpoint unico de emissao

`POST /nfse` aceita **2 modos** (mutuamente exclusivos):

### Modo A - payload completo (`nfse`)

Envie o objeto `nfse` completo (`DadosEmitirRPS.NFSe`).

Regras importantes:
- Nao combinar `nfse` com `empresa`/`cliente`/`servico` no mesmo body.
- `nfse.competencia`, `nfse.prestador`, `nfse.servico` e `nfse.tomador` devem respeitar os campos obrigatorios.
- Neste modo, cliente e servico podem ser criados automaticamente se nao existirem (com base no payload).

### Modo B - payload minimo (`empresa` + `cliente` + `servico`)

Envie somente referencias de registros ja persistidos:
- `empresa`: por `id` ou `cnpj`
- `cliente`: por `id` ou `cpf_cnpj`
- `servico`: por `id` ou `codigo`

Regras importantes:
- Os 3 blocos sao obrigatórios no modo minimo.
- Se cliente/servico nao existirem, retorna erro (`EntityNotFound`).
- Campos opcionais no topo: `referencia`, `competencia`, `formas_recebimento`.

## 5) Exemplo recomendado (modo minimo)

```json
{
  "empresa": { "id": 10 },
  "cliente": { "cpf_cnpj": "98765432000155" },
  "servico": { "codigo": "SERV-001" },
  "referencia": "c1a1f6a2-1623-4dcf-9d8a-31f4cde2b7e1",
  "competencia": "2026-04-08"
}
```

Chamada:

```powershell
$payload = @'
{
  "empresa": { "id": 10 },
  "cliente": { "cpf_cnpj": "98765432000155" },
  "servico": { "codigo": "SERV-001" },
  "referencia": "c1a1f6a2-1623-4dcf-9d8a-31f4cde2b7e1",
  "competencia": "2026-04-08"
}
'@

Invoke-RestMethod -Method Post -Uri "$BASE_URL/nfse" -Headers @{
  "X-AUTH-TOKEN" = $API_TOKEN
  "Content-Type" = "application/json"
} -Body $payload
```

## 6) Exemplo de payload completo (`nfse`)

```json
{
  "nfse": {
    "referencia": "c1a1f6a2-1623-4dcf-9d8a-31f4cde2b7e1",
    "competencia": "2026-04-08",
    "prestador": {
      "cpf_cnpj": "12345678000190",
      "inscricao_municipal": "001010",
      "razao_social": "EMPRESA TESTE LTDA",
      "nome_fantasia": "EMPRESA TESTE",
      "telefone": "21999999999",
      "email": "fiscal@empresa.com",
      "regime_especial_tributacao": "NUMERO_8",
      "prestacao_sus": false,
      "optante_simples_nacional": true,
      "incentivo_fiscal": false,
      "endereco": {
        "logradouro": "Rua A",
        "numero": "100",
        "complemento": "Sala 1",
        "bairro": "Centro",
        "municipio": "Mage",
        "codigo_municipio": "3302502",
        "cep": "25900000",
        "uf": "RJ",
        "nome_pais": "BRASIL",
        "codigo_pais": "1058"
      }
    },
    "servico": {
      "id_servico": 1,
      "codigo": "SERV-001",
      "codigo_cnae": "6201500",
      "descricao": "Consultoria",
      "iss_retido": "NAO",
      "item_lista_servico": "01.07",
      "codigo_municipio": "3302502",
      "exigibilidade_iss": "EXIGIVEL",
      "responsavel_retencao": "TOMADOR",
      "municipio_incidencia": "3302502",
      "valores": {
        "base_calculo": 1500.00,
        "valor_servico": 1500.00,
        "aliquota_iss": 0.02,
        "aliquota_deducoes": 0.00,
        "aliquota_pis": 0.00,
        "aliquota_cofins": 0.00,
        "aliquota_inss": 0.00,
        "aliquota_ir": 0.00,
        "aliquota_csll": 0.00,
        "aliquota_outras_retencoes": 0.00,
        "percentual_desconto_incondicionado": 0.00,
        "percentual_desconto_condicionado": 0.00
      }
    },
    "tomador": {
      "razao_social": "CLIENTE EXEMPLO LTDA",
      "cpf_cnpj": "98765432000155",
      "contato": { "email": "financeiro@cliente.com" },
      "endereco": {
        "logradouro": "Rua B",
        "numero": "200",
        "complemento": "Loja 2",
        "bairro": "Centro",
        "municipio": "Mage",
        "codigo_municipio": "3302502",
        "cep": "25900000",
        "uf": "RJ",
        "nome_pais": "BRASIL",
        "codigo_pais": "1058"
      }
    }
  }
}
```

## 7) Idempotencia e estados

- `referencia` identifica a nota na conta e evita duplicidade de reenvio.
- Se reenviar a mesma `referencia`, a API devolve/atualiza a nota existente.
- Status possiveis: `AUTORIZADA`, `REJEITADA`, `PROCESSANDO`, `CANCELADA`.
- Existe validacao de duplicidade temporal (janela de 5 minutos para prestador+tomador+servico+valor).

## 8) Consulta e download de XML/PDF

```http
GET /nfse/{id}
GET /nfse/{id}/xml
GET /nfse/{id}/pdf
Header: X-AUTH-TOKEN: <token>
```

Ao consultar/detalhar, a API tenta atualizar o estado real da nota quando aplicavel.

## 9) Erros comuns

- `401`: token ausente/invalido/revogado ou sem escopo para `nfse`.
- `409`: possivel duplicidade de emissao na janela temporal.
- `REJEITADA`: verifique `mensagemRetorno` para ajustar dados fiscais/cadastrais.
- `PROCESSANDO`: consulte novamente `GET /nfse/{id}` ou tente `xml/pdf` depois.

## 10) Checklist final de integracao

- [ ] Empresa configurada com certificado e parametros fiscais
- [ ] Token API gerado com `POST /integracao/token-api`
- [ ] Emissao com `POST /nfse` em um unico modo (completo **ou** minimo)
- [ ] `referencia` enviada para idempotencia
- [ ] Fluxo de consulta/retentativa implementado para status `PROCESSANDO`
