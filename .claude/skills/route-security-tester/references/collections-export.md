# COLLECTIONS EXPORT — schemas Insomnia + Postman v2.1 + OpenAPI

**On-demand.** Só gere após confirmação explícita do usuário. Gere a partir do
mapa do DISCOVERY. Organize por recurso. Use env vars genéricas
(`{{baseUrl}}`, `{{authToken}}`, `{{apiKey}}`). **NUNCA hardcode segredos.**

Arquivos: `insomnia.json`, `postman_collection.json` (v2.1), `openapi.yaml`.

---

## Postman Collection v2.1 — `postman_collection.json`

```json
{
  "info": {
    "name": "API Routes",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "variable": [
    { "key": "baseUrl", "value": "" },
    { "key": "authToken", "value": "" },
    { "key": "apiKey", "value": "" }
  ],
  "item": [
    {
      "name": "items",                         
      "item": [
        {
          "name": "List items",
          "request": {
            "method": "GET",
            "header": [
              { "key": "Authorization", "value": "Bearer {{authToken}}" }
            ],
            "url": {
              "raw": "{{baseUrl}}/items?limit=20",
              "host": ["{{baseUrl}}"],
              "path": ["items"],
              "query": [{ "key": "limit", "value": "20" }]
            }
          }
        },
        {
          "name": "Create item",
          "request": {
            "method": "POST",
            "header": [
              { "key": "Content-Type", "value": "application/json" },
              { "key": "Authorization", "value": "Bearer {{authToken}}" }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\"name\":\"\",\"price\":0}",
              "options": { "raw": { "language": "json" } }
            },
            "url": {
              "raw": "{{baseUrl}}/items",
              "host": ["{{baseUrl}}"],
              "path": ["items"]
            }
          }
        }
      ]
    }
  ]
}
```
- Agrupe rotas do mesmo recurso num `item` "pasta" (com `item` aninhado).
- Path params viram segmento `:id` no `path` e entrada em `variable` da request.

---

## Insomnia — `insomnia.json`

Formato de export v4 (`_type: export`). Recursos ligados por `parentId`.

```json
{
  "_type": "export",
  "__export_format": 4,
  "resources": [
    {
      "_id": "wrk_root",
      "_type": "workspace",
      "name": "API Routes"
    },
    {
      "_id": "env_base",
      "_type": "environment",
      "parentId": "wrk_root",
      "name": "Base",
      "data": { "baseUrl": "", "authToken": "", "apiKey": "" }
    },
    {
      "_id": "fld_items",
      "_type": "request_group",
      "parentId": "wrk_root",
      "name": "items"
    },
    {
      "_id": "req_list_items",
      "_type": "request",
      "parentId": "fld_items",
      "name": "List items",
      "method": "GET",
      "url": "{{ _.baseUrl }}/items",
      "headers": [
        { "name": "Authorization", "value": "Bearer {{ _.authToken }}" }
      ]
    },
    {
      "_id": "req_create_item",
      "_type": "request",
      "parentId": "fld_items",
      "name": "Create item",
      "method": "POST",
      "url": "{{ _.baseUrl }}/items",
      "headers": [
        { "name": "Content-Type", "value": "application/json" },
        { "name": "Authorization", "value": "Bearer {{ _.authToken }}" }
      ],
      "body": {
        "mimeType": "application/json",
        "text": "{\"name\":\"\",\"price\":0}"
      }
    }
  ]
}
```
- Env vars referenciadas como `{{ _.baseUrl }}` no Insomnia.
- `request_group` = pasta por recurso.

---

## OpenAPI 3.x — `openapi.yaml`

```yaml
openapi: 3.0.3
info:
  title: API Routes
  version: 1.0.0
servers:
  - url: "{baseUrl}"
    variables:
      baseUrl:
        default: ""
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
    apiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key
paths:
  /items:
    get:
      summary: List items
      security: [{ bearerAuth: [] }]
      parameters:
        - name: limit
          in: query
          schema: { type: integer }
      responses:
        "200": { description: OK }
    post:
      summary: Create item
      security: [{ bearerAuth: [] }]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                name: { type: string }
                price: { type: number }
      responses:
        "201": { description: Created }
  /items/{id}:
    get:
      summary: Get item
      security: [{ bearerAuth: [] }]
      parameters:
        - name: id
          in: path
          required: true
          schema: { type: string }
      responses:
        "200": { description: OK }
        "404": { description: Not found }
```
- Declare em `securitySchemes` apenas os esquemas que o alvo realmente usa
  (mapeados no DISCOVERY). Rotas públicas ficam sem `security`.

---

## Regras invioláveis

- Segredos (token/key) **sempre** como variável vazia — nunca valor real.
- Uma pasta/tag por recurso. Nome da request = ação legível.
- Base URL sempre via variável, nunca fixa.
