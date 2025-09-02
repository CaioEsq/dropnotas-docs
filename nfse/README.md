
---

# Emissão VIA Token de  API Dropnotas

## Visão geral

A API de NFS-e usa **Token de API** no header `X-AUTH-TOKEN` para autenticar e autorizar chamadas às rotas `/nfse`.
O Token:

* é emitido via endpoints próprios,
* é devolvido **em texto puro** (ex.: JWT),
* é armazenado internamente em **Base64** e validado a cada requisição.

> **Importante:** envie **o token exatamente como foi recebido** no header `X-AUTH-TOKEN` (não re-encode em Base64).

---

## Endpoints de Token

| Ação             | Método e rota                         | O que faz                                                | Observações                                                                                                        |
|------------------|---------------------------------------|----------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Gerar token      | `POST /integracao/token-api`          | Cria um token “sem expiração” (controlada por revogação) | Retorna o **token cru** (ex.: JWT). Internamente, a conta guarda o token em **Base64** e o **JTI** para validação. |
| Revogar token    | `DELETE /integracao/token-api`        | Revoga o token atual da conta                            | Zera token e JTI.                                                                                                  |
| Rotacionar token | `PATCH /integracao/token-api/recriar` | Revoga o atual e emite um novo                           | Retorna novo token.                                                                                                |

### Como enviar nas chamadas à API

```http
X-AUTH-TOKEN: <SEU_TOKEN_JWT_EXATAMENTE_COMO_RECEBEU>
```

---

## Pré-requisitos para emitir NFS-e

Antes de chamar `POST /nfse`:

1. **Cadastrar a Empresa (`POST /empresa`)** com os campos:

    * **Certificado** (A1/PFX + senha);
    * **Endereço** completo;
    * **Informações fiscais**: **alíquota**, **série**, **próximo número do lote**, **próximo RPS**, **tipo de RPS**,
      **regime especial de tributação**, demais parâmetros do provedor/município.

2. **Regime Especial de Tributação** (códigos ABRASF):

    * `1` MICROEMPRESA\_MUNICIPAL
    * `2` ESTIMATIVA
    * `3` SOCIEDADE\_DE\_PROFISSIONAIS
    * `4` COOPERATIVA
    * `5` MICROEMPRESARIO\_INDIVIDUAL
    * `6` MICROEMPRESARIO\_E\_EMPRESA\_DE\_PEQUENO\_PORTE
    * `8` (uso específico do município)

> Sem esses itens, o ACBr/Prefeitura pode rejeitar a emissão.

---
Aqui vai um **Quick Start** direto-ao-ponto para sair do zero até **emitir uma NFS-e** criando **Serviço** e **Cliente**
na própria requisição.

> Use os endpoints reais do seu backend (os de **auth** abaixo são ilustrativos).
> Onde diz **PUT /empresa**, ajuste para `POST /empresa` se no seu ambiente for assim.

---

# Quick Start — NFS-e em 5 passos

## 0) Variáveis  (para utilização no Exemplo, ajuste conforme o valor correto)

```bash
export BASE_URL="https://backend.dropnotas.com"

# Conta (exemplo)
export EMAIL="fulano@exemplo.com"
export PASSWORD="sua_senha_forte"

# Empresa
export CNPJ_EMPRESA="12345678000190"

# Endereço (exemplo RJ capital)
export CODIGO_MUNICIPIO="3302502"
export UF="RJ"
```

---

## 1) Criar conta e logar (obter access\_token do Web App)

> **Observação:** Os endpoints de auth abaixo são **exemplos**. Use os da sua API de autenticação.

### Criar conta

```bash
curl -X POST "$BASE_URL/auth/signup" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "'"$EMAIL"'",
    "password": "'"$PASSWORD"'"
  }'
```

### Login → pegue o `access_token` de resposta

```bash
# Se sua API devolver JSON com { "access_token": "..." }:
ACCESS_TOKEN=$(curl -s -X POST "$BASE_URL/auth/login" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "'"$EMAIL"'",
    "password": "'"$PASSWORD"'"
  }' | jq -r '.access_token')
echo "$ACCESS_TOKEN"
```

---

## 2) Configurar a Empresa (**PUT /empresa**)

Inclua **certificado A1 (PFX em Base64 + senha)**, **endereço** e **parâmetros fiscais** (alíquota, série, próxima
numeração, etc.).

Para magé, use SERIE = "UNICA".

```bash
curl -X PUT "$BASE_URL/empresa" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -d '{
    "cpfCnpj": "'"$CNPJ_EMPRESA"'",
    "razaoSocial": "Minha Empresa de Exemplo LTDA",
    "nomeFantasia": "Minha Empresa",
    "endereco": {
      "logradouro": "Rua A",
      "numero": "100",
      "bairro": "Centro",
      "codigoMunicipio": "'"$CODIGO_MUNICIPIO"'",
      "uf": "'"$UF"'",
      "cep": "01000000"
    },
    "certificado": {
      "pfxBase64": "<SEU_PFX_EM_BASE64>",
      "senha": "<SENHA_DO_PFX>"
    },
    "fiscal": {
      "aliquota": 0.02,
      "serie": "UNICA",
      "proximoNumeroLote": 1001,
      "proximoRps": 250,
      "tipoRps": 1,
      "regimeEspecialTributacao": 8,
      "demaisParametros": {
        "provedor": "MAGE",
        "observacoes": "Qualquer parâmetro específico do município"
      }
    }
  }'
```

> **Regime Especial (ABRASF):** 1=MICROEMPRESA\_MUNICIPAL, 2=ESTIMATIVA, 3=SOCIEDADE\_DE\_PROFISSIONAIS, 4=COOPERATIVA,
> 5=MEI, 6=ME/EPP, 8=uso do município.

---

## 3) Gerar **Token de API** (para usar nas rotas `/nfse`)

> A rota de Token usa o **access\_token** do passo anterior.
> A resposta devolve o **token cru** (ex.: JWT). **Guarde exatamente como veio.**

```bash
 API_TOKEN=$(curl -s -X POST "$BASE_URL/integracao/token-api" \
   -H "Authorization: Bearer $ACCESS_TOKEN" | jq -r '.token')
  
```

> **Importante:** Envie **o token exatamente como recebido** no header `X-AUTH-TOKEN` (não re-encode em Base64).

---

## 4) Emitir NFS-e com **criação automática** de Serviço e Cliente

* Informe uma **referência UUID** para **idempotência** (repedir não duplica).
* Informe um **código de serviço novo** (`"codigo": "MENSALIDADE-1"`) para a API criar o Serviço.
* Informe um **CPF/CNPJ de tomador novo** para a API criar o Cliente (dados mínimos).

```bash
UUID_REF="$(uuidgen || echo c1a1f6a2-1623-4dcf-9d8a-31f4cde2b7e1)"

curl -X POST "$BASE_URL/nfse" \
  -H "Content-Type: application/json" \
  -H "X-AUTH-TOKEN: $API_TOKEN" \
  -d '{
    "nfse": {
      "referencia": "'"$UUID_REF"'",
      "prestadorRps": { "cpfCnpj": "'"$CNPJ_EMPRESA"'" },
      "tomadorRps": {
        "cpfCnpj": "98765432000155",
        "razaoSocial": "Cliente Exemplo LTDA",
        "endereco": {
          "logradouro": "Rua B",
          "numero": "200",
          "bairro": "Centro",
          "codigoMunicipio": "'"$CODIGO_MUNICIPIO"'",
          "uf": "'"$UF"'",
          "cep": "01001000"
        }
      },
      "servicoRps": {
        "codigo": "SERV-QUICKSTART-001",
        "descricao": "Consultoria de Implantação",
        "valorServicos": 1500.00,
        "aliquota": 0.02
      }
    }
  }'
```

**O que esperar na resposta:**

* Estado inicial pode ser `AUTORIZADA`, `REJEITADA` ou `PROCESSANDO` (dependendo do provedor/município).
* Caso `REJEITADA`, verifique a `mensagemRetorno`, corrija o que for solicitado e repita a solicitação informando a Referência da Nota.
* Caso `PROCESSANDO`, tente detalhar a nota ou baixar o XML/PDF para atualizar o estado.
* Caso `AUTORIZADA`, virão número/código da NFSe.", caso o Provedor/Prefeitura retorne de forma assíncrona,
  o número e código de verificação podem não vir na primeira resposta, sendo necessário detalhar a nota ou baixar o
  XML/PDF para atualizar o estado.
* Virão **links** para `GET /nfse/{id}`, `.../xml` e `.../pdf` quando aplicável.

---

## 5) Consultar, baixar XML/PDF (e promover resultado)

```bash
# Substitua {id} pelo id retornado na emissão
curl -X GET "$BASE_URL/nfse/{id}" -H "X-AUTH-TOKEN: $API_TOKEN"

# XML (força consulta de lote e promoção de estado)
curl -X GET "$BASE_URL/nfse/{id}/xml" -H "X-AUTH-TOKEN: $API_TOKEN"

# PDF (se autorizado)
curl -X GET "$BASE_URL/nfse/{id}/pdf" -H "X-AUTH-TOKEN: $API_TOKEN"
```

> Ao baixar `XML`/`PDF`, a API **consulta o lote** e promove para o **resultado de fato** (
> AUTORIZADA/REJEITADA/PROCESSANDO).

---

## Dicas rápidas / erros comuns

* **409 (Conflito)**: emissão duplicada nos últimos 5 min (mesmo **prestador+tomador+serviço**).
* **E10 (RPS já informado)**: a API sugere **próxima numeração** automaticamente.
* **PROCESSANDO**: tente `/xml` ou `/pdf` novamente para promover o estado.
* Em **Cancelamento**, `POST /nfse/cancelar` é **lógico interno** (sem integração com prefeitura) — em Magé não há
  cancelamento via API.
* Mais detalhes na seção abaixo.

---

Pronto!:

1. Criada a **conta** e realizado **login**;
2. **Foi configurada a Empresa** (PUT);
3. **Gerado o Token de API**;
4. **Emitida a NFS-e** criando **Serviço** e **Cliente** na mesma chamada;
5. **Consultou/baixou** XML/PDF.

## Emissão de NFS-e

### Endpoint unificado

```http
POST /nfse
Header: X-AUTH-TOKEN: <token>
Body: Dados para emissão (RPS/NFSe)
```

### Regras gerais

* O serviço **gera o RPS** (número, série, tipo) com base na configuração da empresa.
* O serviço **envia o lote** para a prefeitura e **consulta o lote** no mesmo instante.

    * Caso o envio seja **assíncrono** e ainda não houver resposta, ele fica em **PROCESSANDO** até a prefeitura
      autorizar ou rejeitar.
* O serviço **armazena o XML** retornado pela prefeitura (autorizado ou rejeitado).
* O serviço **gera o PDF** (se autorizado) com base no XML.
* O serviço valida se houve um RPS emitido com os mesmos (**prestador** + **tomador** + **serviço**) nos últimos 5
  minutos.

    * Caso detectado, retorna **409 (Conflito)**.

### Regra de **Referência**

* **Referência** é um **UUID** (obrigatório) que garante **idempotência**.

    * Repetir o POST com a mesma referência **não duplica** a nota; você recebe o estado atualizado da nota.
    * Se a nota já estiver autorizada, o resultado é o mesmo (não altera nada).
    * Se estiver rejeitada, ele atualiza a nota com o novo payload e tenta reenviar.
    * Se não informar, a API gera um UUID automaticamente (sem garantir idempotência).

> Para evitar duplicidade, sempre informe um UUID gerado pela sua aplicação.
> Exemplo de UUID v4:
> `c1a1f6a2-1623-4dcf-9d8a-31f4cde2b7e1`

### Regra de **Cliente** e **Serviço**

* **Cliente** é identificado pelo **CPF/CNPJ**.

    * Se não existir, será **criado** automaticamente (dados básicos do Tomador).

* **Serviço** é identificado por **ID** ou **Código** (**obrigatório informar ao menos um**).

    * Se você informar **código** que **não existe**, o serviço será **criado** com base no payload.

---

## Exemplo mínimo de emissão (body)

```json
{
  "nfse": {
    "referencia": "c1a1f6a2-1623-4dcf-9d8a-31f4cde2b7e1",
    "prestadorRps": {
      "cpfCnpj": "12345678000190"
    },
    "tomadorRps": {
      "cpfCnpj": "98765432000155",
      "razaoSocial": "Cliente Exemplo LTDA",
      "endereco": {
        "logradouro": "Rua A",
        "numero": "100",
        "bairro": "Centro",
        "codigoMunicipio": "3302502",
        "uf": "RJ",
        "cep": "01000000"
      }
    },
    "servicoRps": {
      "codigo": "SERV-001",
      "descricao": "Consultoria",
      "valorServicos": 1500.0,
      "aliquota": 0.02
    }
  }
}
```

---

## Resposta

O serviço retorna o estado da nota e **links** para XML e PDF:

```http
GET /nfse/{id}
GET /nfse/{id}/xml
GET /nfse/{id}/pdf
```

---

## Estados e promoção de resultado (“resultado de fato”)

Após enviar, algumas prefeituras demoram para materializar a NFSe (só devolvem **lote + protocolo**).
Sempre que você:

* detalha a nota,
* baixa o **XML** (`/nfse/{id}/xml`),
* baixa o **PDF** (`/nfse/{id}/pdf`),

o serviço automaticamente **consulta o lote** (`consultarLoteRps`) e **promove** o estado para o **resultado de fato**:

* Se houver **XML autorizado** → nota vira **AUTORIZADA**, número/código são extraídos e o **PDF** é gerado.
* Se houver **erro** → nota vira **REJEITADA** com a mensagem.
* Se ainda estiver na fila → permanece **PROCESSANDO**.

---

## Rejeição **E10** (RPS já informado)

Se a prefeitura responder **E10** (RPS duplicado), o serviço:

* consulta por **RPS** para descobrir a NFSe já emitida;
* retorna uma mensagem indicando a situação e a **próxima numeração sugerida** de RPS, alterando-a automaticamente.

Se o número sugerido também estiver em uso, o serviço continuará incrementando até encontrar um número livre.
Caso contrário, será necessário ajustar manualmente pelo endpoint de **Alteração de Empresa**.

> Para corrigir: emita novamente com a nova numeração sugerida ou ajuste o **próximo número de RPS** na configuração da
> empresa.

---

## Cancelamento

* `POST /nfse/cancelar` executa **cancelamento lógico interno** (status `CANCELADA`), **sem** integração com a
  prefeitura.
* O provedor/município (ex.: Magé) **não permite cancelamento via API** nesta integração.

---

## Erros comuns

* **PROCESSANDO**: a prefeitura ainda não disponibilizou a NFSe — tente `/xml` ou `/pdf` novamente.
* **REJEITADA**: verifique a `mensagemRetorno` (alíquota, CNAE, serviço, endereço, certificado, numerações, etc.).
* **E10**: RPS já utilizado — ajuste a numeração.

---

## Exemplos de chamadas

### Emitir

```bash
curl -X POST https://backend.dropnotas.com/nfse \
  -H "Content-Type: application/json" \
  -H "X-AUTH-TOKEN: <SEU_TOKEN>" \
  -d '{ ...payload do exemplo acima... }'
```

### Baixar XML

```bash
curl -X GET https://backend.dropnotas.com/nfse/123/xml \
  -H "X-AUTH-TOKEN: <SEU_TOKEN>"
```

### Baixar PDF

```bash
curl -X GET https://backend.dropnotas.com/nfse/123/pdf \
  -H "X-AUTH-TOKEN: <SEU_TOKEN>"
```

### Gerar Token

```bash
curl -X POST https://backend.dropnotas.com/integracao/token-api \
  -H "Authorization: Bearer <seu_access_token_web_app>"
```

> A resposta contém **o token** (ex.: JWT). Guarde e use no header `X-AUTH-TOKEN` nas rotas `/nfse`.

---
