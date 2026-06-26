# API de Integração de Beneficiários — Medicar

}} **Versão do documento:** 2.0 
}} **Sistema:** ERP TOTVS Protheus — Módulo PLS (Planos de Saúde)  
}} **Última revisão:** 2025

---

## ⚠️ Aviso Importante

Todos os exemplos contidos nesta documentação são **ilustrativos**. Dados como CPF, matrícula, contratos e nomes são fictícios e não retornarão resultado real no ambiente de produção.

---

## Sumário

1. [Visão Geral](#1-visão-geral)
2. [Arquitetura da Integração](#2-arquitetura-da-integração)
3. [Ambiente e Pré-requisitos](#3-ambiente-e-pré-requisitos)
4. [Fluxos de Negócio](#4-fluxos-de-negócio)
5. [Endpoints](#5-endpoints)
   - 5.1 [Autenticação — Token](#51-autenticação--token)
   - 5.2 [Consulta de Dados do Contrato](#52-consulta-de-dados-do-contrato)
   - 5.3 [Inclusão de Beneficiários — PLIncBenModel POST](#53-inclusão-de-beneficiários--plincbenmodel-post)
   - 5.4 [Edição de Protocolo de Inclusão — PLIncBenModel PUT](#54-edição-de-protocolo-de-inclusão--plincbenmodel-put)
   - 5.5 [Alteração de Dados Cadastrais — PLAltBenModel](#55-alteração-de-dados-cadastrais--plaltbenmodel)
   - 5.6 [Bloqueio/Cancelamento de Beneficiários](#56-bloqueiocancelamento-de-beneficiários)
6. [Referência de Campos](#6-referência-de-campos)
7. [Regras de Negócio](#7-regras-de-negócio)
8. [Validações e Formatos](#8-validações-e-formatos)
9. [Tratamento de Erros](#9-tratamento-de-erros)
10. [Exemplos Práticos Completos](#10-exemplos-práticos-completos)
11. [Tabelas de Domínio](#11-tabelas-de-domínio)
12. [Tabelas TOTVS Envolvidas](#12-tabelas-totvs-envolvidas)
13. [Parâmetros de Sistema](#13-parâmetros-de-sistema)
14. [Perguntas Frequentes (FAQ)](#14-perguntas-frequentes-faq)
15. [Guia Rápido de Implementação](#15-guia-rápido-de-implementação)

---

## 1. Visão Geral

### O que é esta integração?

Esta integração permite que sistemas externos realizem, de forma programática, o **cadastro e gerenciamento de beneficiários** no módulo PLS do ERP TOTVS Protheus operado pela Medicar Soluções em Saúde.

As operações disponíveis são:

- **Inclusão** de novos beneficiários (titulares e dependentes)
- **Edição** de um protocolo de inclusão ainda pendente de análise
- **Alteração** de dados cadastrais de beneficiários já ativos
- **Bloqueio/Cancelamento** de beneficiários

### Como funciona o processo?

As operações de inclusão e alteração de dados seguem um **fluxo de protocolização**: ao invés de alterar o cadastro diretamente, a API gera um **protocolo de solicitação** que fica pendente de análise pela operadora na rotina **PLSA977AB — Análise de Beneficiários**. O cadastro só é efetivado após aprovação do protocolo.

O bloqueio age diretamente, sem necessidade de aprovação manual.

### Casos de uso típicos

- Sistemas de RH enviando novos funcionários e dependentes para o plano de saúde
- Portais de autoatendimento de operadoras
- Migração de dados de sistemas legados
- Integrações com sistemas de gestão de convênios empresariais

---

## 2. Arquitetura da Integração

### Visão geral dos componentes

```
┌─────────────────────┐         ┌────────────────────────────────────────┐
│   Sistema Cliente   │         │       ERP TOTVS Protheus (Medicar)     │
│  (seu sistema)      │         │                                        │
│                     │  HTTPS  │  ┌──────────────┐  ┌────────────────┐  │
│  ┌───────────────┐  │◄───────►│  │  REST API    │  │  Módulo PLS    │  │
│  │  HTTP Client  │  │  JSON   │  │  (AppServer) │  │  PLSA977AB     │  │
│  └───────────────┘  │         │  └──────────────┘  └────────────────┘  │
└─────────────────────┘         └────────────────────────────────────────┘
```

### Fluxo de comunicação

```
┌─────────┐    ┌─────────┐    ┌──────────────────┐    ┌──────────────┐
│  Início │───►│  Token  │───►│ Dados do Contrato│───►│  Operação    │
└─────────┘    └─────────┘    └──────────────────┘    │  (Inclusão,  │
                    │                   │             │  Alteração   │
               access_token        tenantid +         │  ou Bloqueio)│
                                   BBA_MATRIC         └──────┬───────┘
                                                             │
                                                  ┌──────────▼────────┐
                                                  │  Protocolo gerado │
                                                  │  (aguarda análise │
                                                  │  na PLSA977AB)    │
                                                  └───────────────────┘
```

### Responsabilidades de cada sistema

| Responsabilidade | Sistema Cliente | Medicar / TOTVS |
|---|---|---|
| Autenticar e gerenciar token | ✅ | — |
| Consultar dados do contrato | ✅ | — |
| Enviar payload bem formado | ✅ | — |
| Validar regras de negócio | — | ✅ |
| Gerar número de protocolo | — | ✅ (automático) |
| Analisar e aprovar protocolos | — | ✅ |
| Efetivar cadastro no BA1 | — | ✅ |

---

## 3. Ambiente e Pré-requisitos

### URLs de produção

| Item | Valor |
|---|---|
| **URL base (produção)** | `https://medicar146707.protheus.cloudtotvs.com.br:1307/rest` |
| **Protocolo** | HTTPS obrigatório |
| **Porta** | `1307` (produção) / `1356` (homologação) |
| **Formato de dados** | JSON (`Content-Type: application/json`) |

}} **URL de homologação:** `https://medicar146708.protheus.cloudtotvs.com.br:1356/rest`

}} ⚠️ **Fluxo de onboarding:** Credenciais são fornecidas **primeiro para homologação**. Somente após validação bem-sucedida dos testes de inclusão e bloqueio, as credenciais de produção são criadas e disponibilizadas.

### Credenciais de acesso

| Item | Descrição | Como obter |
|---|---|---|
| `username` | Usuário de acesso ao ERP Medicar | Fornecido pela equipe Medicar |
| `password` | Senha do usuário no ERP Medicar | Fornecido pela equipe Medicar |
| `cnpjmedicar` | CNPJ da filial Medicar **sem pontuação** | Fornecido pela equipe Medicar |
| `grupoempresa` | Código do grupo de empresa | Fornecido pela equipe Medicar |
| `contrato` | Número do contrato — **equivale ao `BBA_SUBCON`** | Fornecido pela equipe Medicar |

### Ambientes disponíveis

| Ambiente | URL base | `tenantid` |
|---|---|---|
| **Homologação** | `https://medicar146708.protheus.cloudtotvs.com.br:1356/rest` | `01,001001` |
| **Produção** | `https://medicar146707.protheus.cloudtotvs.com.br:1307/rest` | `01,006001` |

}} O valor correto do `tenantid` para o seu ambiente é retornado automaticamente pelo endpoint de **Consulta de Dados do Contrato** (campo `tenantid` na resposta).

---

## 4. Fluxos de Negócio

### ⚠️ Dois tipos de "Alteração" — entenda a diferença

Esta integração possui **dois endpoints distintos** para operações que podem ser chamadas de "alteração":

| Operação | Endpoint | Quando usar |
|---|---|---|
| **Editar protocolo de inclusão pendente** | `PUT PLIncBenModel` | Corrigir dados de um protocolo de inclusão que ainda **não foi aprovado** |
| **Alterar cadastro de beneficiário ativo** | `POST PLAltBenModel` | Atualizar dados de um beneficiário **já cadastrado e ativo** no sistema |

Não confundir as duas operações. O `PUT` no `PLIncBenModel` apenas edita o protocolo antes da aprovação. O `POST` no `PLAltBenModel` cria um novo protocolo de solicitação de alteração cadastral.

---

### 4.1 Inclusão de Beneficiário (PLIncBenModel — POST)

**Quando usar:** Ao adicionar um novo titular ou dependente a um plano de saúde.

**Cenário A — Nova família (novo titular):**
- Não informar `BBA_MATRIC` no bloco `MASTERBBA`
- O sistema criará uma nova família
- Os dados do titular em `DETAILB2N` devem ser idênticos aos de `MASTERBBA`:
  - `B2N_CPFUSR` = `BBA_CPFTIT`
  - `B2N_NOMUSR` = `BBA_EMPBEN`
  - `B2N_CODPRO` = `BBA_CODPRO` (quando informado)

**Cenário B — Dependente em família existente:**
- Informar `BBA_MATRIC` com a matrícula do titular (obtida via `GET /contract?`)
- Neste caso, os demais campos do `MASTERBBA` (`BBA_CODINT`, `BBA_CODEMP`, etc.) **não precisam ser enviados**
- Enviar os dados do dependente em `DETAILB2N`

**Resultado:** Protocolo de inclusão gerado na rotina **PLSA977AB** para análise da operadora.

---

### 4.2 Edição de Protocolo de Inclusão (PLIncBenModel — PUT)

**Quando usar:** Para corrigir dados de um protocolo de inclusão que já foi enviado mas **ainda não foi analisado ou finalizado**.

**Como funciona:**
- Mesmo payload do POST, com `"operation": 4`
- Para titular: enviar todos os campos do `MASTERBBA` + dados em `DETAILB2N`
- Para dependente: enviar apenas `BBA_MATRIC` no `MASTERBBA` + dados em `DETAILB2N`
- A URL não inclui PK — a identificação do protocolo é feita pelos dados do beneficiário

}} **Atenção:** Protocolos **já analisados ou finalizados** não podem ser editados.

---

### 4.3 Alteração de Dados Cadastrais (PLAltBenModel — POST)

**Quando usar:** Para atualizar dados de um beneficiário **já ativo** no sistema (cadastro efetivado na tabela BA1).

**Como funciona:**
- Cada campo a ser alterado é enviado como um item no array `DETAILB7L`
- `B7L_CAMPO`: nome exato do campo na tabela **BA1**
- `B7L_VLPOS`: novo valor para o campo
- O comportamento depende do Layout Genérico configurado em `MV_PLLAYAL`:
  - Campo sem necessidade de análise → protocolo com status `7` (Aprovado Automaticamente) → BA1 atualizada imediatamente
  - Campo que exige análise → protocolo com status `2` (Em Análise) → aguarda aprovação manual

**Resultado:** Protocolo de alteração gerado na rotina **PLSA977AB**.

---

### 4.4 Bloqueio/Cancelamento de Beneficiário

**Quando usar:** Para cancelar ou bloquear o acesso de um beneficiário ao plano. A operação é definitiva e não permite reativação. Caso o beneficiário precise voltar a ficar ativo, deverá ser enviado como novo beneficiário.

**Como funciona:**
- Requer a matrícula do beneficiário (`subscriberId` = `BBA_MATRIC`)
- Requer um código de motivo (`reason`) da tabela B9G
- A data de vigência do bloqueio é definida pelo sistema cliente (`blockDate`)

---

## 5. Endpoints

}} **Nota sobre as URLs nos exemplos:** Todos os exemplos de request nesta seção utilizam a URL do ambiente de **homologação** (`medicar146708...1356`). Para produção, substitua pela URL `https://medicar146707.protheus.cloudtotvs.com.br:1307/rest`.

### 5.1 Autenticação — Token

Gera o `access_token` necessário para autenticar todas as demais chamadas.

| Atributo | Valor |
|---|---|
| **Método** | `POST` |
| **URL** | `https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/api/oauth2/v1/token` |
| **Content-Type** | `application/json` |
| **Autenticação** | Nenhuma (esta é a chamada de autenticação) |

**Query Parameters:**

| Parâmetro | Obrigatório | Valor fixo | Descrição |
|---|---|---|---|
| `grant_type` | ✅ Sim | `password` | Tipo de concessão OAuth2 |
| `username` | ✅ Sim | — | Usuário do ERP Medicar |
| `password` | ✅ Sim | — | Senha do usuário no ERP Medicar |

**Exemplo de request:**

```
POST https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/api/oauth2/v1/token
  ?grant_type=password
  &username=SEU_LOGIN
  &password=SUA_SENHA
```

**Response de sucesso (HTTP 201):**

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

| Campo | Descrição |
|---|---|
| `access_token` | Token a ser usado no header `Authorization: Bearer {{token}}` de todas as chamadas subsequentes |
| `expires_in` | Tempo de validade em segundos (3600 = 1 hora) |
| `token_type` | Sempre `Bearer` |

}} ⚠️ **O token expira em 3600 segundos (1 hora).** Implemente renovação automática na sua integração.

---

### 5.2 Consulta de Dados do Contrato

Retorna os metadados do contrato necessários para as demais operações. **Deve ser chamado antes de qualquer operação de beneficiário.**

| Atributo | Valor |
|---|---|
| **Método** | `GET` |
| **URL** | `https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/client/v1/contract` |
| **Content-Type** | `application/json` |
| **Autenticação** | `Authorization: Bearer {{access_token}}` |

**Query Parameters:**

| Parâmetro | Obrigatório | Descrição |
|---|---|---|
| `cnpjmedicar` | ✅ Sim | CNPJ da filial Medicar **sem pontuação** |
| `grupoempresa` | ✅ Sim | Código do grupo de empresa |
| `contrato` | ✅ Sim | Número do contrato — **é o mesmo valor que `BBA_SUBCON`** |
| `cgcbeneficiario` | ❌ Não | CPF do beneficiário (sem pontuação). Ao informar, retorna a `BBA_MATRIC` desse beneficiário específico. Use este parâmetro para obter o `subscriberId` (matrícula) antes de um bloqueio. |

**Response de sucesso:**

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

**Mapeamento dos campos de resposta para uso nos endpoints seguintes:**

| Campo da resposta | Onde usar |
|---|---|
| `tenantid` | Header obrigatório em todas as chamadas de manutenção |
| `BBA_MATRIC` | `BBA_MATRIC` no MASTERBBA (inclusão de dependente) e `subscriberId` no bloqueio |
| `BBA_CODINT` | `BBA_CODINT` no MASTERBBA da inclusão de titular |
| `BBA_CODEMP` | `BBA_CODEMP` no MASTERBBA da inclusão de titular |
| `BBA_CONEMP` | `BBA_CONEMP` no MASTERBBA |
| `BBA_VERCON` | `BBA_VERCON` no MASTERBBA |
| `BBA_SUBCON` | `BBA_SUBCON` no MASTERBBA |
| `BBA_VERSUB` | `BBA_VERSUB` no MASTERBBA |
| `TITULAR` | Indica se o CPF consultado é titular (`true`) ou dependente (`false`) |
| `FOUND` | Indica se o beneficiário foi encontrado |

**Response de erro:**

```json
{
  "msgerror": "Informe os parametros grupoempresa e contrato."
}
```

---

### 5.3 Inclusão de Beneficiários — PLIncBenModel POST

Cria um protocolo de solicitação de inclusão de novo beneficiário.

| Atributo | Valor |
|---|---|
| **Método** | `POST` |
| **URL** | `https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/fwmodel/PLIncBenModel/` |
| **Content-Type** | `application/json` |
| **Autenticação** | `Authorization: Bearer {{access_token}}` |

**Headers obrigatórios:**

| Header | Exemplo | Descrição |
|---|---|---|
| `Authorization` | `Bearer eyJhbGci...` | Access token da autenticação |
| `tenantid` | `01,006001` | Obtido na resposta do endpoint de Dados do Contrato |

**Headers opcionais de consulta (para GET):**

| Header | Padrão | Descrição |
|---|---|---|
| `COUNT` | `10` | Quantidade de registros retornados |
| `STARTINDEX` | `1` | Índice inicial da paginação |
| `FILTER` | — | Filtro no formato `CAMPO=VALOR` (ex: `BBA_CODSEQ=000770`) |
| `FIELDDETAIL` | `10` | Nível de detalhe dos campos |
| `FIELDVIRTUAL` | `false` | Retorna campos virtuais |
| `FIELDEMPTY` | `false` | Retorna campos sem valor |
| `FIRSTLEVEL` | `true` | Retorna sub-modelos |
| `DEBUG` | `false` | Habilita modo debug |
| `CACHE` | `true` | Cache do total de registros |
| `INTERNALID` | `false` | Retorna o ID (Recno) nas linhas do GRID |

**Outros métodos disponíveis neste endpoint:**

| Método | URL | `operation` | Descrição |
|---|---|---|---|
| `GET` | `.../PLIncBenModel/` | — | Lista todos os protocolos |
| `GET` | `.../PLIncBenModel/{{pk}}` | — | Consulta protocolo específico |
| `PUT` | `.../PLIncBenModel/` | `4` | Edita protocolo pendente (ver seção 5.4) |
| `DELETE` | `.../PLIncBenModel/{{pk}}` | `5` | Exclui protocolo pendente |

**Estrutura do Body (inclusão de titular — nova família):**

```json
{
  "id": "PLIncBenModel",
  "operation": 3,
  "models": [
    {
      "id": "MASTERBBA",
      "modeltype": "FIELDS",
      "fields": [
        { "id": "BBA_CODINT", "order": 1, "value": "{{BBA_CODINT do /contract}}" },
        { "id": "BBA_CODEMP", "order": 2, "value": "{{BBA_CODEMP do /contract" },
        { "id": "BBA_CONEMP", "order": 3, "value": "{{BBA_CONEMP do /contract}}" },
        { "id": "BBA_VERCON", "order": 4, "value": "{{BBA_VERCON do /contract}}" },
        { "id": "BBA_SUBCON", "order": 5, "value": "{{BBA_SUBCON do /contract}}" },
        { "id": "BBA_VERSUB", "order": 6, "value": "{{BBA_VERSUB do /contract}}" },
        { "id": "BBA_EMPBEN", "order": 7, "value": "{{Nome completo do titular}}" },
        { "id": "BBA_CODPRO", "order": 8, "value": "{{Código do plano}}" },
        { "id": "BBA_VERSAO", "order": 9, "value": "{{Versão do plano}}" },
        { "id": "BBA_CPFTIT", "order": 10,"value": "{{CPF do titular sem pontuação}}" }
      ],
      "models": [
        {
          "id": "DETAILB2N",
          "modeltype": "GRID",
          "items": [
            {
              "id": 1,
              "deleted": 0,
              "fields": [
                { "id": "B2N_NOMUSR", "value": "{{Nome completo — igual a BBA_EMPBEN}}" },
                { "id": "B2N_DATNAS", "value": "{{YYYYMMDD}}" },
                { "id": "B2N_GRAUPA", "value": "00" },
                { "id": "B2N_ESTCIV", "value": "{{código estado civil}}" },
                { "id": "B2N_SEXO",   "value": "{{1 ou 2}}" },
                { "id": "B2N_CPFUSR", "value": "{{CPF — igual a BBA_CPFTIT}}" },
                { "id": "B2N_MAE",    "value": "{{Nome da mãe}}" },
                { "id": "B2N_CODPRO", "value": "{{Código do plano — igual a BBA_CODPRO}}" }
              ]
            }
          ]
        },
        {
          "id": "DETAILANEXO",
          "modeltype": "GRID",
          "items": [
            { "id": 1, "deleted": 0, "fields": [] }
          ]
        }
      ]
    }
  ]
}
```

**Estrutura do Body (inclusão de dependente em família existente):**

```json
{
  "id": "PLIncBenModel",
  "operation": 3,
  "models": [
    {
      "id": "MASTERBBA",
      "modeltype": "FIELDS",
      "fields": [
        { "id": "BBA_MATRIC", "order": 1, "value": "{{matrícula do titular — do /contract}}" }
      ],
      "models": [
        {
          "id": "DETAILB2N",
          "modeltype": "GRID",
          "items": [
            {
              "id": 1,
              "deleted": 0,
              "fields": [
                { "id": "B2N_NOMUSR", "value": "{{Nome do dependente}}" },
                { "id": "B2N_DATNAS", "value": "{{YYYYMMDD}}" },
                { "id": "B2N_GRAUPA", "value": "{{código grau parentesco}}" },
                { "id": "B2N_ESTCIV", "value": "{{código estado civil}}" },
                { "id": "B2N_SEXO",   "value": "{{1 ou 2}}" },
                { "id": "B2N_CPFUSR", "value": "{{CPF se }}= 18 anos}}" },
                { "id": "B2N_MAE",    "value": "{{Nome da mãe}}" },
                { "id": "B2N_CODPRO", "value": "{{Código do plano}}" }
              ]
            }
          ]
        },
        {
          "id": "DETAILANEXO",
          "modeltype": "GRID",
          "items": [
            { "id": 1, "deleted": 0, "fields": [] }
          ]
        }
      ]
    }
  ]
}
```

}} **Nota sobre o modelo DETAILANEXO:** Este modelo deve sempre ser enviado no payload, mesmo sem campos preenchidos. Ele é utilizado para anexar arquivos ao protocolo via `DIRECTORY` e `FILENAME`.

---

### 5.4 Edição de Protocolo de Inclusão — PLIncBenModel PUT

Edita um protocolo de inclusão ainda **não analisado ou finalizado**.

| Atributo | Valor |
|---|---|
| **Método** | `PUT` |
| **URL** | `https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/fwmodel/PLIncBenModel/` |
| **Content-Type** | `application/json` |
| **Autenticação** | `Authorization: Bearer {{access_token}}` |

**Headers obrigatórios:** Mesmos do POST (Authorization + tenantid).

**Diferença em relação ao POST:** O campo `"operation"` deve ser `4` (em vez de `3`). O restante do payload é idêntico ao POST.

**Body para edição de protocolo de titular:**

```json
{
  "id": "PLIncBenModel",
  "operation": 4,
  "models": [
    {
      "id": "MASTERBBA",
      "modeltype": "FIELDS",
      "fields": [
        { "id": "BBA_CODINT", "order": 1, "value": "..." },
        { "id": "BBA_CODEMP", "order": 2, "value": "..." },
        { "id": "BBA_CONEMP", "order": 3, "value": "..." },
        { "id": "BBA_VERCON", "order": 4, "value": "..." },
        { "id": "BBA_SUBCON", "order": 5, "value": "..." },
        { "id": "BBA_VERSUB", "order": 6, "value": "..." },
        { "id": "BBA_EMPBEN", "order": 7, "value": "..." },
        { "id": "BBA_CODPRO", "order": 8, "value": "..." },
        { "id": "BBA_VERSAO", "order": 9, "value": "..." },
        { "id": "BBA_CPFTIT", "order": 10,"value": "..." }
      ],
      "models": [ "...mesmo DETAILB2N e DETAILANEXO do POST..." ]
    }
  ]
}
```

**Body para edição de protocolo de dependente:**

```json
{
  "id": "PLIncBenModel",
  "operation": 4,
  "models": [
    {
      "id": "MASTERBBA",
      "modeltype": "FIELDS",
      "fields": [
        { "id": "BBA_MATRIC", "order": 1, "value": "{{matrícula do titular}}" }
      ],
      "models": [ "...mesmo DETAILB2N e DETAILANEXO do POST..." ]
    }
  ]
}
```

---

### 5.5 Alteração de Dados Cadastrais — PLAltBenModel

Cria um protocolo de solicitação de alteração de dados de um beneficiário **já ativo**.

| Atributo | Valor |
|---|---|
| **Método** | `POST` |
| **URL** | `https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/fwmodel/PLAltBenModel/` |
| **Content-Type** | `application/json` |
| **Autenticação** | `Authorization: Bearer {{access_token}}` |

**Headers obrigatórios:** Mesmos do PLIncBenModel (Authorization + tenantid).

**Outros métodos disponíveis:**

| Método | Descrição |
|---|---|
| `GET .../PLAltBenModel/` | Lista protocolos de alteração |
| `GET .../PLAltBenModel/{{pk}}` | Consulta protocolo específico |
| `PUT .../PLAltBenModel/{{pk}}` | Edita protocolo pendente (`operation: 4`) |
| `DELETE .../PLAltBenModel/{{pk}}` | Exclui protocolo pendente (`operation: 5`) |

**Estrutura do Body:**

```json
{
  "id": "PLAltBenModel",
  "operation": 3,
  "models": [
    {
      "id": "MASTERBBA",
      "modeltype": "FIELDS",
      "fields": [
        { "id": "BBA_MATRIC", "order": 1, "value": "{{matrícula do beneficiário}}" }
      ],
      "models": [
        {
          "id": "DETAILB7L",
          "modeltype": "GRID",
          "items": [
            {
              "id": 1,
              "deleted": 0,
              "fields": [
                { "id": "B7L_CAMPO", "value": "{{campo da tabela BA1}}" },
                { "id": "B7L_VLPOS", "value": "{{novo valor}}" },
                { "id": "B7L_USR",   "value": "{{usuário solicitante}}" }
              ]
            }
          ]
        },
        {
          "id": "DETAILANEXO",
          "modeltype": "GRID",
          "items": [
            { "id": 1, "deleted": 0, "fields": [] }
          ]
        }
      ]
    }
  ]
}
```

}} Para alterar múltiplos campos, adicione múltiplos objetos no array `items` com `"id": 1`, `"id": 2`, etc.

---

### 5.6 Bloqueio/Cancelamento de Beneficiários

Registra o bloqueio ou cancelamento de um beneficiário.

| Atributo | Valor |
|---|---|
| **Método** | `POST` |
| **URL** | `https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/totvsHealthPlans/familyContract/v1/beneficiaries/blockProtocol` |
| **Content-Type** | `application/json` |
| **Autenticação** | `Authorization: Bearer {{access_token}}` |

**Headers obrigatórios:**

| Header | Produção | Teste |
|---|---|---|
| `Authorization` | `Bearer {{token}}` | `Bearer {{token}}` |
| `tenantid` | `01,006001` | `01,001001` |

**Request Body:**

```json
{
  "subscriberId": "{{BBA_MATRIC do beneficiário}}",
  "reason": "{{código do motivo}}",
  "blockDate": "{{YYYY-MM-DD}}",
  "loginUser": "{{nome do operador}}"
}
```

**Endpoint auxiliar — Consulta de motivos de bloqueio:**

```
GET https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/totvsHealthPlans/familyContract/v1/reasons
Authorization: Bearer {{access_token}}
```

}} **Informação não encontrada na documentação fornecida:** A estrutura de request/response deste endpoint auxiliar não foi documentada. Consultar a equipe Medicar.

---

## 6. Referência de Campos

### 6.1 MASTERBBA — Cabeçalho da Solicitação (Inclusão de Titular)

| Campo | Obrigatório | Tipo | Descrição | Fonte |
|---|---|---|---|---|
| `BBA_CODINT` | ✅ Sim | string | Código da Operadora (tabela BA0) | `GET /contract?` → `BBA_CODINT` |
| `BBA_CODEMP` | ✅ Sim | string | Código da Empresa (tabela BG9) | `GET /contract?` → `BBA_CODEMP` |
| `BBA_CONEMP` | ⚠️ Obrig. p/ PJ | string | Código do Contrato (tabela BT5) | `GET /contract?` → `BBA_CONEMP` |
| `BBA_VERCON` | ⚠️ Obrig. p/ PJ | string | Versão do Contrato | `GET /contract?` → `BBA_VERCON` |
| `BBA_SUBCON` | ⚠️ Obrig. p/ PJ | string | Código do SubContrato (tabela BQC) | `GET /contract?` → `BBA_SUBCON` |
| `BBA_VERSUB` | ⚠️ Obrig. p/ PJ | string | Versão do SubContrato | `GET /contract?` → `BBA_VERSUB` |
| `BBA_EMPBEN` | ✅ Sim | string | Nome completo do beneficiário titular | Dado do cliente |
| `BBA_CODPRO` | ✅ Sim | string | Código do plano do titular | Fornecido pela Medicar |
| `BBA_VERSAO` | ✅ Sim | string | Versão do plano do titular | Fornecido pela Medicar |
| `BBA_CPFTIT` | ✅ Sim | string | CPF do titular (11 dígitos, sem pontuação) | Dado do cliente |
| `BBA_NROPRO` | ❌ Não | string | Número do protocolo. Se omitido, gerado automaticamente | — |
| `BBA_MATRIC` | ❌ Nova família / ✅ Dependente | string | Matrícula do titular. **Omitir para nova família. Obrigatório para incluir dependente.** | `GET /contract?` → `BBA_MATRIC` |
| `BBA_CODCLI` | ❌ Não | string | Código do cliente na tabela SA1. Se omitido: PF cria novo cliente; PJ segue regra do `MV_PLSFMCL` | — |
| `BBA_LOJA` | ❌ Não | string | Código da loja do cliente na SA1 | — |

### 6.2 DETAILB2N — Dados do Beneficiário

| Campo | Obrigatório | Tipo | Descrição | Valores/Formato |
|---|---|---|---|---|
| `B2N_NOMUSR` | ✅ Sim | string | Nome completo do beneficiário | Texto livre |
| `B2N_DATNAS` | ✅ Sim | string | Data de nascimento | `YYYYMMDD` ex: `"19850304"` |
| `B2N_GRAUPA` | ✅ Sim | string | Código do grau de parentesco (tabela BRP) | Ver [seção 11.1](#111-grau-de-parentesco-b2n_graupa) |
| `B2N_ESTCIV` | ✅ Sim | string | Código do estado civil (SX5 tabela 33) | Ver [seção 11.2](#112-estado-civil-b2n_estciv) |
| `B2N_SEXO` | ✅ Sim | string | Sexo: `"1"` = Masculino, `"2"` = Feminino | `"1"` ou `"2"` |
| `B2N_MAE` | ✅ Sim | string | Nome da mãe do beneficiário. | Texto livre |
| `B2N_CPFUSR` | ✅ Sim | string | CPF do beneficiário (11 dígitos, sem pontuação). | `"63090663074"` |
| `B2N_CODPRO` | ❌ Não | string | Código do plano do beneficiário. Se omitido, usa o plano do titular (`BBA_CODPRO`) | — |
| `B2N_DRGUSR` | ❌ Não | string | Número do RG | — |
| `B2N_ORGEM` | ❌ Não | string | Órgão emissor do RG | `"SSP"` |
| `B2N_RGEST` | ❌ Não | string | Estado emissor do RG (sigla UF) | `"SP"` |
| `B2N_NRCRNA` | ❌ Não | string | Número da Carteira Nacional de Saúde (CNS) | — |
| `B2N_CEPUSR` | ❌ Não | string | CEP (tabela BC9) | `"01310100"` |
| `B2N_ENDERE` | ❌ Não | string | Logradouro | — |
| `B2N_NR_END` | ❌ Não | string | Número do endereço | — |
| `B2N_COMEND` | ❌ Não | string | Complemento | — |
| `B2N_BAIRRO` | ❌ Não | string | Bairro | — |
| `B2N_CODMUN` | ❌ Não | string | Código do município (tabela BID) | — |
| `B2N_MUNICI` | ❌ Não | string | Descrição do município | — |
| `B2N_ESTADO` | ❌ Não | string | UF do estado | `"SP"` |
| `B2N_DDD` | ❌ Não | string | DDD do telefone | `"11"` |
| `B2N_TELEFO` | ❌ Não | string | Número do telefone (sem DDD) | `"999998888"` |
| `B2N_EMAIL` | ❌ Não | string | E-mail | — |
| `B2N_UNIVER` | ❌ Não | string | Universitário: `"1"` = Sim, `"0"` = Não | — |
| `B2N_INVALI` | ❌ Não | string | Invalidez: `"1"` = Sim, `"0"` = Não | — |
| `B2N_COMUNI` | ❌ Não | string | Preferência de comunicação: `"0"` E-mail, `"1"` SMS, `"2"` Ambos | — |
| `B2N_PAI` | ❌ Não | string | Nome do pai | — |
| `B2N_BANCO` | ❌ Não | string | Código do banco (tabela SA6) | — |
| `B2N_AGENC` | ❌ Não | string | Agência bancária | — |
| `B2N_CONTA` | ❌ Não | string | Conta bancária | — |
| `B2N_DATADT` | ❌ Não | string | Data de adoção (formato YYYYMMDD) | — |
| `B2N_DATINC` | ❌ Não | string | Data de inclusão (formato YYYYMMDD) | — |
| `DIRECTORY` | ❌ Não | string | URL HTTP do arquivo a anexar ao protocolo | — |
| `FILENAME` | ❌ Não | string | Nome do arquivo a anexar | — |

### 6.3 MASTERBBA — Cabeçalho (Alteração de Dados — PLAltBenModel)

| Campo | Obrigatório | Tipo | Descrição |
|---|---|---|---|
| `BBA_MATRIC` | ✅ Sim | string | Matrícula do beneficiário cujos dados serão alterados |

### 6.4 DETAILB7L — Campos a Alterar (PLAltBenModel)

| Campo | Obrigatório | Tipo | Descrição |
|---|---|---|---|
| `B7L_CAMPO` | ✅ Sim | string | Nome exato do campo na tabela **BA1** a ser alterado |
| `B7L_VLPOS` | ✅ Sim | string | Novo valor do campo |
| `B7L_USR` | ✅ Sim | string | Identificação do usuário/sistema que solicitou a alteração |
| `DIRECTORY` | ❌ Não | string | URL do arquivo a anexar - Não utilizar|
| `FILENAME` | ❌ Não | string | Nome do arquivo a anexar - Não utilizar|

### 6.5 Campos do Bloqueio

| Campo JSON | Parâmetro TOTVS | Obrigatório | Descrição | Como obter |
|---|---|---|---|---|
| `subscriberId` | `BBA_MATRIC` | ✅ Sim | Matrícula do beneficiário | `GET /contract?cgcbeneficiario={{CPF}}` → campo `BBA_MATRIC` |
| `reason` | `B9G_COD` | ✅ Sim | Código do motivo do bloqueio | `GET .../v1/reasons` |
| `blockDate` | — | ✅ Sim | Data de vigência do bloqueio — formato `YYYY-MM-DD` | Definida pelo sistema cliente |
| `loginUser` | — | ✅ Sim | Nome do operador que solicitou o bloqueio | Definida pelo sistema cliente |

---

## 7. Regras de Negócio

### 7.1 Autenticação e sessão

- O `access_token` expira em **3.600 segundos (1 hora)**
- Todas as chamadas, exceto a de autenticação, exigem `Authorization: Bearer {{token}}`
- O `tenantid` deve ser passado no header de todas as chamadas de manutenção de beneficiários
- O valor do `tenantid` é retornado pelo endpoint de Dados do Contrato

### 7.2 Inclusão — nova família vs dependente

| Cenário | `BBA_MATRIC` no MASTERBBA | Outros campos do MASTERBBA |
|---|---|---|
| Novo titular (nova família) | ❌ Não enviar | Todos obrigatórios |
| Dependente em família existente | ✅ Matrícula do titular | Não precisam ser enviados |

### 7.3 Consistência dos dados do titular

Quando se inclui um titular, os dados na seção `DETAILB2N` **devem ser idênticos** aos da seção `MASTERBBA`:

| Campo B2N | Deve ser igual a |
|---|---|
| `B2N_CPFUSR` | `BBA_CPFTIT` |
| `B2N_NOMUSR` | `BBA_EMPBEN` |
| `B2N_CODPRO` | `BBA_CODPRO` (quando informado) |

### 7.4 Obrigatoriedade de CPF

| Situação | CPF (`B2N_CPFUSR`) |
|---|---|
| Titular (qualquer idade) | ✅ Obrigatório |
| Dependente com 18 anos ou mais | ✅ Obrigatório |
| Dependente menor de 18 anos | ✅ Obrigatório  |

### 7.5 Nome da mãe (`B2N_MAE`)

- A documentação principal da Medicar marca este campo como **obrigatório**
- A documentação técnica TOTVS indica que é condicional ao parâmetro `BQC_INFANS = "1 - Sim"` do subcontrato
- **Recomendação prática:** Sempre enviar o nome da mãe para evitar rejeição

### 7.6 Plano do dependente (`B2N_CODPRO`)

- Se não informado, o sistema utiliza automaticamente o plano do titular (`BBA_CODPRO`)

### 7.7 Cadastro de cliente (`BBA_CODCLI`)

- **Não informado + pessoa física:** cria novo cliente (SA1) automaticamente
- **Não informado + pessoa jurídica:** cria cliente somente se não houver nível de cobrança configurado e `MV_PLSFMCL` estiver ativo

### 7.8 Alteração de dados (`PLAltBenModel`)

- Cada campo a ser alterado é um item separado no array `items` do `DETAILB7L`
- O comportamento (imediato ou com análise) depende do Layout Genérico (`MV_PLLAYAL`):
  - Sem necessidade de análise → `BBA_STATUS = 7` (Aprovado Automaticamente) → BA1 atualizada imediatamente
  - Com necessidade de análise → `BBA_STATUS = 2` (Em Análise) → aprovação manual necessária
- O Layout de alteração usa **exclusivamente a tabela BA1**

### 7.9 Edição e exclusão de protocolos

- Apenas protocolos com status **não analisado e não finalizado** podem ser editados (PUT) ou excluídos (DELETE)
- Para PLIncBenModel, o PUT usa `operation: 4`; para DELETE, usa `operation: 5`


---

## 8. Validações e Formatos

### 8.1 Formatos de data

| Campo | Formato | Separador | Exemplo |
|---|---|---|---|
| `B2N_DATNAS` | `YYYYMMDD` | Nenhum | `"19850304"` |
| `B2N_DATINC` | `YYYYMMDD` | Nenhum | `"20250101"` |
| `B2N_DATADT` | `YYYYMMDD` | Nenhum | `"20250101"` |
| `blockDate` | `YYYY-MM-DD` | Hifens | `"2025-10-09"` |

}} ⚠️ Os formatos de data são **diferentes** entre os endpoints de inclusão/edição e o de bloqueio.

### 8.2 Formato de CPF

- Apenas dígitos, sem pontos ou traços
- 11 caracteres: `"63090663074"`

### 8.3 Todos os valores são strings

Mesmo valores numéricos devem ser enviados como strings. Exemplo: `"B2N_SEXO": "1"`, não `"B2N_SEXO": 1`.

### 8.4 Validações obrigatórias

| Regra | Detalhe |
|---|---|
| CPF do titular | Obrigatório; 11 dígitos numéricos sem pontuação |
| CPF do dependente | Obrigatório; 11 dígitos numéricos sem pontuação  |
| Grau de parentesco | Código válido da tabela BRP (ver seção 11.1) |
| Estado civil | Código válido da SX5 tabela 33 (ver seção 11.2) |
| Sexo | Apenas `"1"` ou `"2"` |
| Data de nascimento | Formato `YYYYMMDD`, data válida |
| Código do motivo de bloqueio | Deve existir na tabela B9G do sistema |

---

## 9. Tratamento de Erros

### 9.1 Códigos HTTP esperados

| Código | Situação | Ação |
|---|---|---|
| `200 OK` | Sucesso nas chamadas de consulta e manutenção | Processar resposta |
| `201 Created` | Sucesso na criação do token | Usar o `access_token` |
| `400 Bad Request` | Payload inválido ou parâmetros ausentes | Verificar campos obrigatórios e formatos |
| `401 Unauthorized` | Token inválido ou expirado | Renovar token via endpoint de autenticação |
| `404 Not Found` | Recurso não encontrado | Verificar parâmetros enviados |
| `500 Internal Server Error` | Erro interno no Protheus | Verificar payload; acionar suporte Medicar |

}} Os testes do Postman esperam `200 ou 201` para o token; `200` para consultas; `200, 201 ou 204` para PUT; e `200, 202 ou 204` para o cancelamento.

### 9.2 Erros de negócio conhecidos

| Mensagem | Causa | Solução |
|---|---|---|
| `"Informe os parametros grupoempresa e contrato."` | Parâmetros ausentes na consulta de contrato | Incluir `grupoempresa` e `contrato` na query string |
| Protocolo já analisado / finalizado | Tentativa de PUT ou DELETE em protocolo fechado | Não é possível editar — criar novo protocolo se necessário |

### 9.3 Formato geral de erro

```json
{
  "msgerror": "Descrição do erro aqui"
}
```

}} **Informação não encontrada na documentação fornecida:** O formato exato de resposta de erro para PLIncBenModel, PLAltBenModel e blockProtocol não foi exemplificado. Tratar qualquer resposta com campo `msgerror` ou HTTP ≠ 2xx como erro.

---

## 10. Exemplos Práticos Completos

### 10.1 Inclusão de Novo Titular (Nova Família)

**Step 1 — Token:**

```
POST https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/api/oauth2/v1/token
  ?grant_type=password&username=SEU_LOGIN&password=SUA_SENHA
```

**Step 2 — Consultar contrato:**

```
GET https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/client/v1/contract
  ?cnpjmedicar=12345678000100&grupoempresa=0004&contrato=001022033
Authorization: Bearer eyJhbGci...
```

Response:
```json
{
  "BBA_CODINT": "1001",
  "BBA_CODEMP": "0004",
  "BBA_CONEMP": "000000000002",
  "BBA_VERCON": "001",
  "BBA_SUBCON": "001022033",
  "BBA_VERSUB": "001",
  "BBA_MATRIC": "",
  "TITULAR": "",
  "FOUND": true,
  "tenantid": "01,006001"
}
```

**Step 3 — Incluir titular + dependente:**

```
POST https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/fwmodel/PLIncBenModel/
Authorization: Bearer eyJhbGci...
Content-Type: application/json
tenantid: 01,006001
```

```json
{
  "id": "PLIncBenModel",
  "operation": 3,
  "models": [
    {
      "id": "MASTERBBA",
      "modeltype": "FIELDS",
      "fields": [
        { "id": "BBA_CODINT", "order": 1,  "value": "1001" },
        { "id": "BBA_CODEMP", "order": 2,  "value": "0004" },
        { "id": "BBA_CONEMP", "order": 3,  "value": "000000000002" },
        { "id": "BBA_VERCON", "order": 4,  "value": "001" },
        { "id": "BBA_SUBCON", "order": 5,  "value": "001022033" },
        { "id": "BBA_VERSUB", "order": 6,  "value": "001" },
        { "id": "BBA_EMPBEN", "order": 7,  "value": "NOME COMPLETO DO TITULAR" },
        { "id": "BBA_CODPRO", "order": 8,  "value": "0066" },
        { "id": "BBA_VERSAO", "order": 9,  "value": "001" },
        { "id": "BBA_CPFTIT", "order": 10, "value": "00000000000" }
      ],
      "models": [
        {
          "id": "DETAILB2N",
          "modeltype": "GRID",
          "items": [
            {
              "id": 1,
              "deleted": 0,
              "fields": [
                { "id": "B2N_NOMUSR", "value": "NOME COMPLETO DO TITULAR" },
                { "id": "B2N_DATNAS", "value": "19850304" },
                { "id": "B2N_GRAUPA", "value": "00" },
                { "id": "B2N_ESTCIV", "value": "C" },
                { "id": "B2N_SEXO",   "value": "2" },
                { "id": "B2N_CPFUSR", "value": "00000000000" },
                { "id": "B2N_MAE",    "value": "NOME DA MAE DO TITULAR" },
                { "id": "B2N_CODPRO", "value": "0066" }
              ]
            },
            {
              "id": 2,
              "deleted": 0,
              "fields": [
                {"id":"B2N_NOMUSR","value": "NOME DO DEPENDENTE"},
                {"id":"B2N_DATNAS","value": "19910304"},
                {"id":"B2N_GRAUPA","value": "11"},
                {"id":"B2N_ESTCIV","value": "C"},
                {"id":"B2N_SEXO"  ,"value": "1"},
                {"id":"B2N_CPFUSR","value": "20777841053"},
                {"id":"B2N_MAE"   ,"value": "MAE DO DEPENDENTE"},
                {"id":"B2N_CODPRO" ,"value":"0066"}
              ]
            }
          ]
        },
        {
          "id": "DETAILANEXO",
          "modeltype": "GRID",
          "items": [
            { "id": 1, "deleted": 0, "fields": [] }
          ]
        }
      ]
    }
  ]
}
```

---

### 10.2 Inclusão de Dependente em Família Existente

Para incluir um dependente, **consulte o contrato com o CPF do titular** para obter a `BBA_MATRIC`, depois envie apenas essa matrícula no MASTERBBA:

```
GET .../contract?cnpjmedicar=...&grupoempresa=...&contrato=...&cgcbeneficiario={{CPF_TITULAR}}
```

```json
{
  "id": "PLIncBenModel",
  "operation": 3,
  "models": [
    {
      "id": "MASTERBBA",
      "modeltype": "FIELDS",
      "fields": [
        { "id": "BBA_MATRIC", "order": 1, "value": "00011008000019017" }
      ],
      "models": [
        {
          "id": "DETAILB2N",
          "modeltype": "GRID",
          "items": [
            {
              "id": 1,
              "deleted": 0,
              "fields": [
                { "id": "B2N_NOMUSR", "value": "NOME DO DEPENDENTE" },
                { "id": "B2N_DATNAS", "value": "20100522" },
                { "id": "B2N_GRAUPA", "value": "06" },
                { "id": "B2N_ESTCIV", "value": "S" },
                { "id": "B2N_SEXO",   "value": "2" },
                {"id":"B2N_CPFUSR","value": "20777841053"},
                { "id": "B2N_MAE",    "value": "NOME DA MAE DO DEPENDENTE" },
                { "id": "B2N_CODPRO", "value": "0066" }
              ]
            }
          ]
        },
        {
          "id": "DETAILANEXO",
          "modeltype": "GRID",
          "items": [
            { "id": 1, "deleted": 0, "fields": [] }
          ]
        }
      ]
    }
  ]
}
```

---

### 10.3 Edição de Protocolo de Inclusão Pendente (PUT)

Para editar um protocolo ainda não aprovado, envie o mesmo payload com `"operation": 4`:

```
PUT https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/fwmodel/PLIncBenModel/
Authorization: Bearer ...
tenantid: 01,006001
```

```json
{
  "id": "PLIncBenModel",
  "operation": 4,
  "models": [
    {
      "id": "MASTERBBA",
      "modeltype": "FIELDS",
      "fields": [
        { "id": "BBA_CODINT", "order": 1,  "value": "1001" },
        { "id": "BBA_CODEMP", "order": 2,  "value": "0004" },
        { "id": "BBA_CONEMP", "order": 3,  "value": "000000000002" },
        { "id": "BBA_VERCON", "order": 4,  "value": "001" },
        { "id": "BBA_SUBCON", "order": 5,  "value": "001022033" },
        { "id": "BBA_VERSUB", "order": 6,  "value": "001" },
        { "id": "BBA_EMPBEN", "order": 7,  "value": "NOME COMPLETO CORRIGIDO" },
        { "id": "BBA_CODPRO", "order": 8,  "value": "0066" },
        { "id": "BBA_VERSAO", "order": 9,  "value": "001" },
        { "id": "BBA_CPFTIT", "order": 10, "value": "00000000000" }
      ],
      "models": [
        {
          "id": "DETAILB2N",
          "modeltype": "GRID",
          "items": [
            {
              "id": 1,
              "deleted": 0,
              "fields": [
                { "id": "B2N_NOMUSR", "value": "NOME COMPLETO CORRIGIDO" },
                { "id": "B2N_DATNAS", "value": "19850304" },
                { "id": "B2N_GRAUPA", "value": "00" },
                { "id": "B2N_ESTCIV", "value": "C" },
                { "id": "B2N_SEXO",   "value": "2" },
                { "id": "B2N_CPFUSR", "value": "00000000000" },
                { "id": "B2N_MAE",    "value": "NOME DA MAE" },
                { "id": "B2N_CODPRO", "value": "0066" }
              ]
            }
          ]
        },
        {
          "id": "DETAILANEXO",
          "modeltype": "GRID",
          "items": [
            { "id": 1, "deleted": 0, "fields": [] }
          ]
        }
      ]
    }
  ]
}
```

---

### 10.4 Alteração de Dados Cadastrais de Beneficiário Ativo (PLAltBenModel)

```
POST https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/fwmodel/PLAltBenModel/
Authorization: Bearer ...
tenantid: 01,006001
```

```json
{
  "id": "PLAltBenModel",
  "operation": 3,
  "models": [
    {
      "id": "MASTERBBA",
      "modeltype": "FIELDS",
      "fields": [
        { "id": "BBA_MATRIC", "order": 1, "value": "10010006000024002" }
      ],
      "models": [
        {
          "id": "DETAILB7L",
          "modeltype": "GRID",
          "items": [
            {
              "id": 1,
              "deleted": 0,
              "fields": [
                { "id": "B7L_CAMPO", "value": "BA1_EMAIL" },
                { "id": "B7L_VLPOS", "value": "novo.email@empresa.com" },
                { "id": "B7L_USR",   "value": "SISTEMA.INTEGRACAO" }
              ]
            },
            {
              "id": 2,
              "deleted": 0,
              "fields": [
                { "id": "B7L_CAMPO", "value": "BA1_TELEFO" },
                { "id": "B7L_VLPOS", "value": "988887777" },
                { "id": "B7L_USR",   "value": "SISTEMA.INTEGRACAO" }
              ]
            }
          ]
        },
        {
          "id": "DETAILANEXO",
          "modeltype": "GRID",
          "items": [
            { "id": 1, "deleted": 0, "fields": [] }
          ]
        }
      ]
    }
  ]
}
```

---

### 10.5 Bloqueio/Cancelamento de Beneficiário

**Step 1 — Consultar o contrato com CPF do beneficiário para obter a matrícula:**

```
GET .../contract?cnpjmedicar=...&grupoempresa=...&contrato=...&cgcbeneficiario={{CPF}}
```

Guardar o valor de `BBA_MATRIC` da resposta.

**Step 2 — Bloquear:**

```
POST https://medicar146708.protheus.cloudtotvs.com.br:1356/rest/totvsHealthPlans/familyContract/v1/beneficiaries/blockProtocol
Authorization: Bearer ...
Content-Type: application/json
tenantid: 01,006001
```

```json
{
  "subscriberId": "10010004152488005",
  "reason": "000001",
  "blockDate": "2025-10-09",
  "loginUser": "NOME DO BENEFICIARIO"
}
```

---

## 11. Tabelas de Domínio

### 11.1 Grau de Parentesco (B2N_GRAUPA)

Referência: tabela **BRP** do TOTVS.

| Código | Tipo | Sexo esperado |
|---|---|---|
| `00` | TITULAR | A (qualquer) |
| `02` | ESPOSA | F |
| `03` | COMPANHEIRO(A) | A |
| `04` | ESPOSO | M |
| `05` | FILHO | M |
| `06` | FILHA | F |
| `07` | PAI | M |
| `08` | MÃE | F |
| `09` | SOGRO | M |
| `10` | SOGRA | F |
| `11` | OUTRO DEPENDENTE | A |
| `12` | FILHO ADOTIVO | M |
| `13` | FILHA ADOTIVA | F |
| `14` | IRMÃO | M |
| `15` | IRMÃ | F |
| `16` | AGREGADO | A |

### 11.2 Estado Civil (B2N_ESTCIV)

Referência: **SX5 tabela 33** do TOTVS.

| Código | Status |
|---|---|
| `C` | Casado(a) |
| `D` | Divorciado(a) |
| `M` | União Estável |
| `Q` | Desquitado(a) |
| `S` | Solteiro(a) |
| `V` | Viúvo(a) |
| `O` | Outros |

### 11.3 Motivos de Bloqueio (B9G_COD)

| Código | Descrição |
|---|---|
| `000001` | DESLIGAMENTO DA EMPRESA |
| `000002` | FINANCEIRO |
| `000003` | CADASTRO INDEVIDO |

}} Esta tabela pode conter outros motivos configurados no ambiente. Consultar `GET .../v1/reasons` para lista completa e atualizada.

### 11.4 Sexo

| Código | Descrição |
|---|---|
| `1` | Masculino |
| `2` | Feminino |

### 11.5 Preferência de Comunicação (B2N_COMUNI)

| Código | Descrição |
|---|---|
| `0` | E-mail |
| `1` | SMS |
| `2` | Ambos |

### 11.6 Valores booleanos em campos string

| Campo | Verdadeiro | Falso |
|---|---|---|
| `B2N_UNIVER` (universitário) | `"1"` | `"0"` |
| `B2N_INVALI` (invalidez) | `"1"` | `"0"` |

---

## 12. Tabelas TOTVS Envolvidas

| Tabela | Nome | Relevância |
|---|---|---|
| `BBA` | Cabeçalho Solicitação de Beneficiários | Protocolo criado pela API |
| `B2N` | Inclusão de Beneficiários | Dados do beneficiário na inclusão |
| `B7L` | Campos para Alteração | Campos e valores na alteração cadastral |
| `BA1` | Cadastro de Beneficiários | Cadastro efetivo após aprovação |
| `BA3` | Família | Agrupamento familiar |
| `BRP` | Graus de Parentesco | Domínio de `B2N_GRAUPA` |
| `SA6` | Bancos | Domínio de dados bancários |
| `BC9` | CEPs | Domínio de CEP |
| `BID` | Municípios | Domínio de município |
| `B90` | Layout Pág. Web | Configuração do Layout Genérico |
| `B91` | Campos Layout | Regras por campo do Layout Genérico |
| `BG9` | Grupos Empresas | Código do grupo de empresa |
| `BQC` | Subcontrato | Dados do subcontrato |
| `BT6` | Empresa Contrato Produto | Vínculo empresa-contrato-produto |
| `BI3` | Produtos de Saúde | Planos disponíveis |
| `SA1` | Cadastro de Clientes | Clientes vinculados aos beneficiários |
| `ACB` | Bancos de Conhecimentos | Configurações internas |
| `AC9` | Relação de Objetos x Entidades | Configurações internas |

---

## 13. Parâmetros de Sistema

| Parâmetro | Descrição | Afeta |
|---|---|---|
| `MV_PLLAYIN` | Layout Genérico Web para **inclusão** de beneficiários. Ex: `PPLINCBEN` | PLIncBenModel |
| `MV_PLLAYAL` | Layout Genérico Web para **alteração** de beneficiários. Ex: `PPLALTBEN` | PLAltBenModel |
| `MV_PLURDOW` | Diretório web para salvar arquivos recebidos via API (`DIRECTORY`/`FILENAME`) | Ambos |
| `MV_PLSFMCL` | Controla criação automática de cliente (SA1) para pessoa jurídica | PLIncBenModel |
| `BQC_INFANS` | Quando `"1 - Sim"`, torna `B2N_MAE` obrigatório na inclusão | PLIncBenModel |

---

## 14. Perguntas Frequentes (FAQ)

**P: Qual a URL de produção do ambiente Medicar?**  
R: **Homologação:** `https://medicar146708.protheus.cloudtotvs.com.br:1356/rest`  
R: **Produção:** `https://medicar146707.protheus.cloudtotvs.com.br:1307/rest`

---

**P: O campo `contrato` no endpoint de consulta é o mesmo que `BBA_SUBCON`?**  
R: Sim. A collection Postman confirma: `contrato = NUMERO CONTRATO (MESMO NUMERO DO BBA_SUBCON)`.

---

**P: Como obtenho a `BBA_MATRIC` de um beneficiário específico para fazer um bloqueio?**  
R: Chame o endpoint de Consulta de Dados do Contrato com o CPF do beneficiário no parâmetro `cgcbeneficiario`. O campo `BBA_MATRIC` na resposta é o `subscriberId` a ser usado no bloqueio.

---

**P: Qual a diferença entre "Alteração" no Postman e a API PLAltBenModel?**  
R: São coisas distintas. O **PUT no PLIncBenModel** (com `operation: 4`) edita um protocolo de *inclusão ainda pendente* — útil para corrigir dados antes da aprovação. O **POST no PLAltBenModel** cria um protocolo de *alteração cadastral* de um beneficiário já ativo no sistema.

---

**P: Preciso chamar o endpoint de Dados do Contrato toda vez?**  
R: Não necessariamente. Se você já armazenou o `tenantid`, `BBA_MATRIC` e demais campos, pode reutilizá-los. 

---

**P: Posso incluir titular e dependentes em um único request?**  
R: Sim, pode-se adicionar novo titular junto com os dependentes numa única request.

---

**P: Posso cancelar um protocolo após enviá-lo?**  
R: Sim, com `DELETE` na URL `.../PLIncBenModel/{{pk}}` (ou `PUT` com `operation: 5` no body). Apenas protocolos **não analisados ou finalizados** podem ser excluídos.

---

**P: O que é o `DETAILANEXO` e preciso enviá-lo?**  
R: É o modelo para anexar arquivos ao protocolo via `DIRECTORY` e `FILENAME`. Deve ser sempre enviado no payload, mesmo sem campos preenchidos (`"fields": []`), para manter a estrutura esperada pela API.

---

**P: O campo `deleted: 0` nos itens do GRID tem alguma função?**  
R: Indica que o item não está marcado para exclusão. Em operações de PUT onde se deseja remover um item do GRID, defina `"deleted": 1` naquele item.

---

**P: Como sei se o protocolo foi aprovado?**  
R: Consulte via `GET .../PLIncBenModel/{{pk}}` ou `GET .../PLAltBenModel/{{pk}}`. Status conhecidos: `2` = Em Análise, `7` = Aprovado Automaticamente. A lista completa de status não foi documentada — consultar a equipe Medicar.

---

## 15. Guia Rápido de Implementação

**Passo 1 — Coletar informações**

Solicitar à equipe Medicar:
- `username` e `password`
- `cnpjmedicar` (sem pontuação)
- `grupoempresa`
- `contrato` (= `BBA_SUBCON`)
- Código do plano (`BBA_CODPRO`) e versão (`BBA_VERSAO`)

**Passo 2 — Autenticar**

```
POST .../rest/api/oauth2/v1/token?grant_type=password&username=X&password=Y
→ Salvar access_token
→ Renovar antes de 3600s
```

**Passo 3 — Consultar dados do contrato**

```
GET .../rest/client/v1/contract?cnpjmedicar=X&grupoempresa=Y&contrato=Z
→ Salvar: tenantid, BBA_CODINT, BBA_CODEMP, BBA_CONEMP,
          BBA_VERCON, BBA_SUBCON, BBA_VERSUB
```

**Passo 4 — Incluir novo titular**

```
POST .../rest/fwmodel/PLIncBenModel/
Headers: Authorization + tenantid
Body: operation:3 + MASTERBBA completo + DETAILB2N + DETAILANEXO
→ Verificar protocolo na rotina PLSA977AB
```

**Passo 5 — Incluir dependente**

```
GET .../contract?...&cgcbeneficiario={{CPF_TITULAR}}  → obter BBA_MATRIC
POST .../rest/fwmodel/PLIncBenModel/
Body: operation:3 + MASTERBBA só com BBA_MATRIC + DETAILB2N + DETAILANEXO
```

**Passo 6 — Alterar cadastro de beneficiário ativo**

```
GET .../contract?...&cgcbeneficiario={{CPF}}  → obter BBA_MATRIC
POST .../rest/fwmodel/PLAltBenModel/
Body: operation:3 + BBA_MATRIC + DETAILB7L (campos a alterar) + DETAILANEXO
```

**Passo 7 — Bloquear beneficiário**

```
GET .../contract?...&cgcbeneficiario={{CPF}}  → obter BBA_MATRIC (= subscriberId)
GET .../v1/reasons  → obter código do motivo
POST .../v1/beneficiaries/blockProtocol
Body: { subscriberId, reason, blockDate, loginUser }
```

**Passo 8 — Implementar tratamento de erros**

- Capturar HTTP ≠ 2xx
- Verificar campo `msgerror` nas respostas
- Implementar logging completo de request e response
- Implementar retry com backoff exponencial para erros 5xx
- Renovar token automaticamente ao receber 401

**Passo 9 — Validar em homologação antes da produção**

- URL: `https://medicar146708.protheus.cloudtotvs.com.br:1356/rest`
- `tenantid`: `01,001001`
- Executar todos os fluxos: inclusão, edição, alteração, bloqueio
- Verificar os protocolos gerados na rotina PLSA977AB

---

## Referências

- Documentação TOTVS — API PLIncBenModel: https://tdn.totvs.com/pages/releaseview.action?pageId=691456568
- Documentação TOTVS — FWRestModel: Portal TDN TOTVS
- Documentação TOTVS — Layout Genérico Web (PLSCADLAY): Portal TDN TOTVS

---

*Documento consolidado a partir de: Documentação_API_de_Beneficiarios.pdf, API_PLIncBenModel.pdf, API_PLAltBenModel.pdf e collection Postman JSON_Api_integração_Beneficiarios.*
