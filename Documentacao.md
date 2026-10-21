# API de Integração de Beneficiários — Medicar

**Versão do documento:** 2.3  
**Sistema:** ERP TOTVS Protheus — Módulo PLS (Planos de Saúde)  
**Última revisão:** 2026

---

## ⚠️ Aviso Importante

Essa é uma versão adaptada da documentação para a Medicar. Para a versão completa e oficial, [visite o site da TOTVS](https://tdn.totvs.com/pages/releaseview.action?pageId=794393323).

Todos os exemplos contidos nesta documentação são ilustrativos. Dados como CPF, matrícula, contratos e nomes são fictícios e não retornarão resultado real no ambiente de produção.

---

## 1. Visão Geral

Esta integração permite que sistemas externos realizem, de forma programática, o cadastro e gerenciamento de beneficiários no módulo PLS do ERP TOTVS Protheus operado pela Medicar.

As operações disponíveis são:

- Inclusão de novos beneficiários (titulares e dependentes)
- Edição de um protocolo de inclusão ainda pendente de análise
- Alteração de dados cadastrais de beneficiários já ativos
- Bloqueio e desbloqueio de beneficiários
- Consulta da base completa de beneficiários por subcontrato

---

## 2. Ambiente e Pré-requisitos

### URLs de produção e homologação

| Ambiente | URL base |
|---|---|
| Homologação | `https://medicar146708.protheus.cloudtotvs.com.br:1356/rest` |
| Produção | `https://medicar146707.protheus.cloudtotvs.com.br:1307/rest` |

### Credenciais de acesso

| Item | Descrição |
|---|---|
| `username` | Usuário de acesso ao ERP Medicar |
| `password` | Senha do usuário no ERP Medicar |
| `cnpjmedicar` | CNPJ da filial Medicar sem pontuação |
| `grupoempresa` | Código do grupo de empresa |
| `contrato` | Número do contrato (equivale ao `BBA_SUBCON`) |

---

## 3. Autenticação

### 3.1 Obter token

Método: `POST`

URL:

`https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/api/oauth2/v1/token`

Query params:

| Parâmetro | Obrigatório | Valor |
|---|---|---|
| `grant_type` | Sim | `password` |
| `username` | Sim | Usuário do ERP Medicar |
| `password` | Sim | Senha do ERP Medicar |

Exemplo:

```http
POST https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/api/oauth2/v1/token?grant_type=password&username=SEU_LOGIN&password=SUA_SENHA
```

Resposta esperada:

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI...",
  "refresh_token": "Td6bvFRy73CBYCo4EzJXj06A...",
  "scope": "default",
  "token_type": "Bearer",
  "expires_in": 3600,
  "hasMFA": false
}
```

> O token expira em 3600 segundos (1 hora). Implemente renovação automática na sua integração.

---

## 4. Consulta de Dados do Contrato

Retorna os metadados do contrato necessários para as demais operações. Deve ser chamado antes de qualquer operação de beneficiário.

Método: `GET`  
URL: `https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/client/v1/contract`

Headers:

```http
Authorization: Bearer {{access_token}}
```

Query params:

| Parâmetro | Obrigatório | Descrição |
|---|---|---|
| `cnpjmedicar` | Sim | CNPJ da filial Medicar sem pontuação |
| `grupoempresa` | Sim | Código do grupo de empresa |
| `contrato` | Sim | Número do contrato / subcontrato |
| `cgcbeneficiario` | Não | CPF do beneficiário para retorno específico |

Exemplo:

```http
GET https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/client/v1/contract?cnpjmedicar=12345678000100&grupoempresa=0004&contrato=001022033
Authorization: Bearer {{access_token}}
```

Resposta:

```json
{
  "BBA_CODINT": "1001",
  "BBA_CODEMP": "0006",
  "BBA_CONEMP": "000000000005",
  "BBA_VERCON": "001",
  "BBA_SUBCON": "000000001",
  "BBA_VERSUB": "001",
  "BBA_MATRIC": "10010006000024002",
  "TITULAR": true,
  "FOUND": true,
  "tenantid": "01,006001"
}
```

Importante:

- `tenantid` deve ser enviado em todas as chamadas de manutenção de beneficiários.
- `BBA_MATRIC` é usado para inclusão de dependente e para o bloqueio/desbloqueio.

---

## 5. Consulta de todos os beneficiários do subcontrato

A API conta com consulta da base de todos os beneficiários ativos de um subcontrato. Ao enviar o código do subcontrato, a API retorna os dados correspondentes.

### 5.1 Endpoint

Método: `POST`

Autenticação: `Authorization: Bearer {{access_token}}`

Homologação:

`https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/medicarpls/contratos/beneficiario`

Produção:

`https://medicar146707.protheus.cloudtotvs.com.br:1307/rest/medicarpls/contratos/beneficiario`

### 5.2 Body

```json
{
  "subcontrato": "002110579"
}
```

### 5.3 Exemplo de retorno

```json
{
  "beneficiarios": [
    {
      "matricula": "10010004999O7N009",
      "cpf": "05099999902",
      "beneficiario": "Roberto Carlos Cardoso Junior",
      "tipo": "T",
      "nascimento": "13/06/1976",
      "endereco": "QD 4E CNJ. C CASA 13L",
      "municipio": "BRASSILIA",
      "ddd": "61",
      "telefone": "9999992",
      "inclusao": "12/07/2026"
    },
    {
      "matricula": "1001000499CS3O007",
      "cpf": "08999999957",
      "beneficiario": "Raissa pereira",
      "tipo": "T",
      "nascimento": "28/06/2015",
      "endereco": "",
      "municipio": "",
      "ddd": "",
      "telefone": "61982000096",
      "inclusao": "21/01/2026"
    }
  ]
}
```

Observação:

- A API retorna apenas os dados que foram fornecidos pelo cliente no momento da inclusão.
- Campos vazios ou não informados não são retornados nessa consulta.

---

## 6. Inclusão de Beneficiário

### 6.1 Status da solicitação

Quando há uma requisição de inclusão de beneficiário, o usuário pode consultar o status da solicitação. Os valores possíveis são:

| Status | Descrição |
|---|---|
| `1` | Pendente de documentação |
| `2` | Em análise |
| `3` | Processado |
| `4` | Aprovado |
| `5` | Rejeitado |
| `6` | Aprovado parcialmente |
| `7` | Aprovado automaticamente |

> Em solicitações de inclusão, o status `5` normalmente indica rejeição por duplicidade, como quando o CPF informado já está ativo no contrato.

---

## 7. Bloqueio e Desbloqueio de Beneficiário

No payload de bloqueio/desbloqueio, o campo `loginUser` deve ser informado com o nome do beneficiário.

Exemplo de bloqueio:

```json
{
  "subscriberId": "10014146985124BD45",
  "reason": "000001",
  "blockDate": "2026-05-15",
  "loginUser": "NOME DO BENEFICIARIO"
}
```

Exemplo de desbloqueio:

```json
{
  "subscriberId": "10014146985124BD45",
  "reason": "000004",
  "blockDate": "2026-05-14",
  "loginUser": "NOME DO BENEFICIARIO"
}
```

### Regras

- Bloqueio e desbloqueio utilizam o mesmo endpoint.
- A diferenciação é feita por `reason` e `blockDate`.
- Para desbloqueio, o `reason` deve ser `"000004"`.
- Para desbloqueio, a data deve ser um dia anterior à data do bloqueio.

---

## 8. Endpoint de bloqueio/desbloqueio

Método: `POST`

URL:

`https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/totvsHealthPlans/familyContract/v1/beneficiaries/blockProtocol`

Headers:

```http
Authorization: Bearer {{access_token}}
tenantid: {{tenantid}}
```

Body:

```json
{
  "subscriberId": "{{BBA_MATRIC do beneficiário}}",
  "reason": "{{código do motivo}}",
  "blockDate": "{{YYYY-MM-DD}}",
  "loginUser": "{{nome do beneficiário}}"
}
```

---

## 9. Regras de Negócio

### 9.1 CPF

- O CPF deve ser enviado sem máscara, apenas com dígitos.
- Exemplo: `63090663074`

### 9.2 Datas

- Datas de inclusão/alteração no payload do PLS no formato `YYYYMMDD`.
- Datas de bloqueio/desbloqueio no formato `YYYY-MM-DD`.

### 9.3 Autenticação

- O `access_token` expira em 3600 segundos (1 hora).
- Todas as chamadas, exceto a de autenticação, exigem `Authorization: Bearer {{token}}`.
- O `tenantid` deve ser enviado no header de todas as chamadas de manutenção.

---

## 10. Tratamento de Erros

| Código | Situação |
|---|---|
| `200 OK` | Sucesso em consultas e atualizações |
| `201 Created` | Token gerado com sucesso |
| `400 Bad Request` | Payload inválido ou parâmetros ausentes |
| `401 Unauthorized` | Token inválido ou expirado |
| `404 Not Found` | Recurso não encontrado |
| `500 Internal Server Error` | Erro interno do Protheus |

Exemplo de erro:

```json
{
  "msgerror": "Informe os parametros grupoempresa e contrato."
}
```

---

## 11. Referência rápida

| Item | Valor |
|---|---|
| Versão da documentação | `2.3` |
| Ambiente homologação | `https://medicar146708.protheus.cloudtotvs.com.br:1356/rest` |
| Ambiente produção | `https://medicar146707.protheus.cloudtotvs.com.br:1307/rest` |
| Endpoint autenticação | `/api/oauth2/v1/token` |
| Endpoint contrato | `/client/v1/contract` |
| Endpoint consulta beneficiários | `/medicarpls/contratos/beneficiario` |
| Endpoint bloqueio/desbloqueio | `/totvsHealthPlans/familyContract/v1/beneficiaries/blockProtocol` |
