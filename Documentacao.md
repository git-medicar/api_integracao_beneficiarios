# API de Integração de Beneficiários — Medicar

**Versão do documento:** 2.0
**Sistema:** ERP TOTVS Protheus — Módulo PLS (Planos de Saúde)
**Última revisão:** 2026

---

## ⚠️ Aviso Importante

Todos os exemplos contidos nesta documentação são **ilustrativos**. Dados como CPF, matrícula, contratos e nomes são fictícios e não retornarão resultado real no ambiente de produção.

---

## 1. Visão Geral

Esta integração permite que sistemas externos realizem, de forma programática, o **cadastro e gerenciamento de beneficiários** no módulo PLS do ERP TOTVS Protheus operado pela Medicar Soluções.

### Operações disponíveis:
- **Inclusão** de novos beneficiários (titulares e dependentes)
- **Edição** de protocolo de inclusão pendente
- **Alteração** de dados cadastrais de beneficiários ativos
- **Bloqueio/Cancelamento** de beneficiários

---

## 3. Ambiente e Pré-requisitos

### URLs

| Ambiente | URL | Porta |
|---|---|---|
| **Produção** | `https://medicar146707.protheus.cloudtotvs.com.br` | 1307 |
| **Homologação** | `https://medicar146708.protheus.cloudtotvs.com.br` | 1356 |

### Credenciais

- `username` - Usuário do ERP Medicar
- `password` - Senha do usuário
- `cnpjmedicar` - CNPJ sem pontuação
- `grupoempresa` - Código do grupo
- `contrato` - Número do contrato (= `BBA_SUBCON`)

---

## 5. Endpoints

### 5.1 Autenticação — Token

```
POST https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/api/oauth2/v1/token
  ?grant_type=password
  &username=SEU_LOGIN
  &password=SUA_SENHA
```

**Response:** HTTP 201
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

---

### 5.2 Consulta de Dados do Contrato

```
GET https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/client/v1/contract
  ?cnpjmedicar=XXXXX&grupoempresa=XXXX&contrato=XXXXXXX
Authorization: Bearer {{access_token}}
```

---

### 5.3 Inclusão de Beneficiários — PLIncBenModel POST

```
POST https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/fwmodel/PLIncBenModel/
Authorization: Bearer {{access_token}}
tenantid: 01,006001
Content-Type: application/json
```

---

### 5.6 Bloqueio de Beneficiário

```
POST https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/totvsHealthPlans/familyContract/v1/beneficiaries/blockProtocol
Authorization: Bearer {{access_token}}
tenantid: 01,006001
Content-Type: application/json
```

```json
{
  "subscriberId": "10010006000024002",
  "reason": "000001",
  "blockDate": "2025-10-09",
  "loginUser": "OPERADOR"
}
```

---

## 8. Validações e Formatos

### Formatos de data

| Campo | Formato | Exemplo |
|---|---|---|
| `B2N_DATNAS` | YYYYMMDD | "19850304" |
| `blockDate` | YYYY-MM-DD | "2025-10-09" |

### Formato de CPF
- Apenas dígitos, sem pontuação
- 11 caracteres: "63090663074"

### Todos os valores são strings
```json
{ "B2N_SEXO": "1" }  // ✅ Correto
{ "B2N_SEXO": 1 }    // ❌ Errado
```

---

## 9. Tratamento de Erros

| Código | Significado | Ação |
|---|---|---|
| 200 | OK | Sucesso |
| 201 | Created | Token criado |
| 400 | Bad Request | Verificar payload |
| 401 | Unauthorized | Renovar token |
| 404 | Not Found | Verificar parâmetros |
| 500 | Server Error | Contactar suporte |

---

**Para mais detalhes, acesse o repositório:** [git-medicar/api_integracao_beneficiarios](https://github.com/git-medicar/api_integracao_beneficiarios)
