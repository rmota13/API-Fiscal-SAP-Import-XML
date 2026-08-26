# API Fiscal SAP - Import XML

> Automação da entrada de documentos fiscais eletrônicos (CT-e) no SAP Business One,
> da leitura do XML à criação do esboço na fila de aprovação fiscal 
> preservando integralmente o processo de conferência humana.

**Status:** ✅ Em produção · Endpoint validado

**Autor:** Rodrigo Mota de Oliveira
**Stack:** Python · FastAPI · SAP Business One Service Layer · n8n · Lovable · Microsoft Teams

> Case técnico independente. Credenciais, endpoints, hostnames, dados de fornecedores
> e parâmetros contábeis proprietários foram removidos ou substituídos por placeholders.

---

## O Problema

A entrada de CT-e (Conhecimento de Transporte Eletrônico) no ERP é um processo
recorrente, repetitivo e de alto custo operacional em qualquer empresa que compra frete:

- Cada documento exige login no cliente do ERP e preenchimento manual de ~15 campos fiscais
- Os campos são sempre os mesmos (CFOP, CST, conta contábil, utilização, série), mas errar um deles gera retrabalho contábil
- A vinculação com a NF de venda correspondente depende de conferência visual
- O volume torna o processo caro: em um cenário de 90 documentos/mês,
  são horas mensais de digitação que não agregam valor analítico

O processo não é difícil  é **repetitivo e sensível a erro**. É exatamente o tipo de
trabalho que deve ser automatizado sem alterar a governança que já funciona.

---

## A Solução

Uma API dedicada que recebe os dados já validados do CT-e, resolve a transportadora
no ERP, monta o payload fiscal com base em um gabarito auditado e cria o documento
via Service Layer  **como esboço**, não como lançamento efetivo.

```
XML do CT-e
    ↓
App de conferência (Lovable)     ← validação fiscal em 5 etapas
    ↓ webhook
n8n (orquestração + idempotência)
    ↓ POST /fiscal/lancar-cte
API Fiscal SAP  ← este projeto
    ↓ Service Layer (Drafts)
SAP Business One — fila de aprovação
    ↓ aprovação humana
Documento contábil efetivado
```

---

## A Decisão Arquitetural Central: esboço, não lançamento

Este é o ponto que define a qualidade da solução.

A primeira versão criava a nota de entrada **diretamente** (`PurchaseInvoices` → `OPCH`).
Funcionava, o documento era criado com sucesso e todos os campos fiscais corretos.
E estava **errado**.

**Por quê:** o processo manual da equipe fiscal cria um esboço que entra em uma fila
de aprovação antes de virar documento contábil. Automatizar "pulando" essa etapa
significaria que o robô teria mais autonomia fiscal do que a pessoa que ele substitui.

**A correção:** o endpoint passou a criar em `Drafts` (`ODRF`) com
`DocObjectCode = oPurchaseInvoices`, entrando na mesma fila de aprovação
do fluxo manual.

**O que isso garante:**

| Benefício | Impacto |
|---|---|
| Aprovador mantém controle total | Nada entra na contabilidade sem revisão humana |
| Fiscal pode editar antes de aprovar | Data, CFOP, utilização e vencimento continuam ajustáveis |
| Erro do robô é reversível | Cancela-se o esboço; nenhum lançamento contábil foi feito |
| Zero mudança de processo | Quem aprova não percebe diferença entre manual e automático |

Automação madura não remove o controle humano **remove a digitação**.

---

## Endpoints

### `POST /fiscal/lancar-cte`

Cria o esboço da nota de entrada de frete no ERP.

**Headers**
```
Content-Type: application/json
x-api-key: {API_KEY}
```

**Request**
```json
{
  "chave_cte": "00000000000000000000000000000000000000000000",
  "numero_cte": 1,
  "transportadora_cnpj": "00000000000000",
  "transportadora_nome": "TRANSPORTADORA EXEMPLO LTDA",
  "valor_frete": 279.02,
  "data_emissao": "2026-08-18",
  "numero_nf_venda": 53392,
  "numero_pedido": 35660,
  "card_code": "F0000"
}
```

> `card_code` é opcional. Quando omitido, a API resolve o fornecedor
> pelo CNPJ no cadastro do ERP. Fornecedor não cadastrado retorna `422`
> com mensagem acionável não falha silenciosamente.

**Response `200`**
```json
{
  "sucesso": true,
  "doc_entry": 0000,
  "doc_num": 0000,
  "chave_cte": "00000000000000000000000000000000000000000000",
  "numero_cte": 1,
  "card_code": "F0000",
  "valor_frete": 279.02,
  "mensagem": "Esboço criado com sucesso — aguardando aprovação fiscal"
}
```

### `GET /fiscal/saude-sap`
Verifica conectividade e autenticação com o Service Layer do ERP.

### `GET /health`
Healthcheck do serviço (usado por monitoramento e deploy).

---

## Gabarito Fiscal

Os campos fiscais são **fixos e auditados** extraídos de um documento real
lançado manualmente pela equipe fiscal e validado contabilmente antes da implementação.

```
DocType               dDocument_Items
ItemCode              FRETE
AccountCode           {conta contábil de frete}
CFOPCode              {CFOP de entrada de serviço de transporte}
CSTCode               {CST aplicável}
TaxCode               {código de imposto}
Series                {série de numeração}
BPL_IDAssignedToInvoice  {filial}
PaymentGroupCode      {condição de pagamento}
ShippingMethod        {método de envio}
CSTforIPI / PIS / COFINS  {CSTs de tributos federais}
```

**Decisão:** gabarito fixo, não parametrização dinâmica. Variações fiscais
(operações interestaduais, fornecedores do Simples Nacional) serão tratadas
como extensões explícitas e testadas não como configuração livre.
Flexibilidade prematura em regra fiscal é passivo, não feature.

---

## Arquitetura Técnica

**Isolamento de domínio.** Esta API é deliberadamente separada da plataforma de
integração de e-commerce mantida pelo mesmo autor
([SAP Commerce Integration Platform](https://github.com/rmota13/sap-business-one-integration-platform)).

Os dois sistemas conversam com o mesmo ERP, mas têm:

| Dimensão | Commerce | Fiscal |
|---|---|---|
| Origem do dado | Webhooks de marketplace | XML de documento fiscal |
| Operador | Equipe comercial | Equipe fiscal/contábil |
| Ciclo de release | Contínuo | Conservador (impacto contábil) |
| Consequência de erro | Pedido não criado | Lançamento contábil indevido |

Compartilhar deploy entre eles significaria que uma correção de marketplace
poderia derrubar o processamento fiscal. O custo de manter dois serviços
é menor que o custo desse acoplamento.

**Session management explícito.** O Service Layer do SAP B1 usa sessão com cookie
(`B1SESSION` + `ROUTEID` para afinidade de nó em cluster). A API abre e encerra
a sessão por requisição, com `logout` garantido em bloco `finally` 
sessões órfãs consomem licença de acesso indireto, que é um recurso escasso.

**Autenticação por API key em header**, com validação centralizada via
dependência do FastAPI aplicada no router inteiro, não endpoint a endpoint.

**Erros traduzidos.** Falhas do ERP são capturadas, extraídas do envelope OData
e devolvidas em linguagem acionável (`422` para regra de negócio,
`502` para indisponibilidade). O operador nunca vê um stack trace.

---

## Ecossistema

| Camada | Ferramenta | Papel |
|---|---|---|
| Captura e conferência | Lovable | App web que lê o XML e valida o documento em etapas antes de liberar o lançamento |
| Orquestração | n8n (self-hosted) | Webhook, idempotência por chave do documento, resolução de fornecedor, roteamento de exceções |
| Lançamento | FastAPI (este projeto) | Montagem do payload fiscal e criação do esboço via Service Layer |
| ERP | SAP Business One | Fila de aprovação e efetivação contábil |
| Observabilidade | Microsoft Teams | Alertas via Adaptive Cards apenas em exceção o caminho feliz é silencioso |

**Idempotência** por chave do documento fiscal: antes de qualquer lançamento,
verifica-se se aquela chave já gerou documento no ERP. Reenvio de webhook,
retry de rede ou reprocessamento manual não duplicam lançamento contábil.

---

## Stack

```
Python 3.x · FastAPI · Uvicorn · Pydantic v2
SAP Business One Service Layer (OData v4)
n8n · SQL Server · Windows Service (NSSM)
```

---

## Estrutura

```
app/
├── main.py          # App, autenticação por API key, registro de routers
├── config.py        # Settings tipadas via .env (pydantic-settings)
├── sap_session.py   # Session manager do Service Layer (login/request/logout)
└── routers/
    └── fiscal.py    # Endpoints, resolução de fornecedor, montagem do payload
```

---

## Execução local

```bash
# 1. Configure o .env a partir do exemplo
cp .env.example .env

# 2. Instale as dependências
pip install -r requirements.txt

# 3. Suba o servidor
uvicorn app.main:app --host 0.0.0.0 --port 8003
```

**Variáveis de ambiente**
```env
SAP_BASE_URL=https://{host}:{porta}/b1s/v1
SAP_HOST_HEADER={hostname}
SAP_USER={usuario}
SAP_PASSWORD={senha}
SAP_COMPANY_DB={banco}
API_KEY={chave}
PORT=8003
```

---

## Roadmap

- [x] Estrutura da API e session management do Service Layer
- [x] Endpoint de lançamento com gabarito fiscal auditado
- [x] Resolução de fornecedor por CNPJ com fallback explícito
- [x] Correção arquitetural: esboço em vez de lançamento direto
- [x] Deploy como serviço de sistema com início automático
- [x] Validação ponta a ponta em ambiente de homologação
- [x] Idempotência por chave do documento na camada de API
- [x] Integração com o app de conferência de XML
- [x] Cobertura de cenários fiscais adicionais (interestadual, Simples Nacional)
- [x] Aprovação → produção
- [ ] Extensão para outros documentos de entrada (NF-e de serviço)

---

## O que este projeto demonstra

- Integração com ERP corporativo via API oficial, respeitando limitações reais
  de licenciamento e sessão
- Tradução de regra fiscal brasileira para payload de sistema, com gabarito auditado
- Decisão arquitetural orientada a governança: automatizar o trabalho,
  não a responsabilidade
- Isolamento de domínio entre sistemas que compartilham o mesmo backend
- Design de erro voltado ao operador, não ao desenvolvedor
- Deploy em ambiente Windows Server corporativo, com serviço gerenciado
 
---

## Autor

**Rodrigo Mota de Oliveira**
Dados, Business Intelligence, Integrações e Automação Corporativa

- GitHub: [@rmota13](https://github.com/rmota13)

---

## Aviso

Case técnico independente. Não representa documentação oficial da SAP
nem material de qualquer organização. Valores fiscais, identificadores,
endpoints e credenciais foram substituídos por placeholders.
