# FUNCTIONAL — matriz de status e receitas de curl

Objetivo: exercitar cada rota do mapa (DISCOVERY) ao vivo, cobrindo caminhos
válidos e inválidos, e registrar comportamento observado. Sempre com rate limit;
verbos destrutivos só após confirmação explícita.

---

## Espectro de status a cobrir

| Faixa | Status | Cenário que o provoca |
|---|---|---|
| Sucesso | `200` | GET/PUT/PATCH ok com corpo |
| | `201` | POST criou recurso |
| | `204` | DELETE/ação sem corpo de retorno |
| Cliente | `400` | body malformado / JSON inválido |
| | `401` | sem credencial ou credencial inválida |
| | `403` | autenticado mas sem permissão |
| | `404` | recurso/rota inexistente |
| | `405` | método não permitido na rota |
| | `409` | conflito (duplicado, estado inconsistente) |
| | `422` | validação semântica falhou (campo fora de range) |
| Limite | `429` | rate limit disparado |
| Servidor | `500/502/503` | erro não tratado / dependência fora |

Para cada rota, tente provocar os status **plausíveis** para o seu método —
não force os impossíveis.

---

## Padrões de input por rota

Gere, por rota, ao menos:

- **Válido feliz** — todos os campos corretos → espera 2xx.
- **Campo obrigatório faltando** → espera 400/422.
- **Tipo errado** (string onde espera número, etc.) → espera 400/422.
- **Valor fora de range / limite** → espera 422.
- **Recurso inexistente** (id aleatório) → espera 404.
- **Método errado na rota** → espera 405.
- **Duplicata** quando houver unicidade → espera 409.

---

## Receita de curl com métricas

Capture status, tempo, tamanho e shape numa só chamada. Aplique rate limit
(`sleep`) entre requisições.

```bash
# Uma requisição instrumentada
curl -s -o /tmp/rst_body.json -w \
  'status=%{http_code} time_ms=%{time_total} size=%{size_download}\n' \
  -X GET "$BASE_URL/items/123" \
  -H "Authorization: Bearer $TOKEN"

# time_total vem em segundos → multiplique por 1000 para ms na tabela.
```

Formato `-w` recomendado (uma linha parseável por requisição):
```
-w 'status=%{http_code}\ttime=%{time_total}\tsize=%{size_download}\n'
```

Shape do body (só a estrutura, não o dump inteiro) — se `jq` disponível:
```bash
jq -r 'if type=="object" then keys elif type=="array" then ["array", length]
       else type end' /tmp/rst_body.json
```

POST/PUT com corpo:
```bash
curl -s -o /tmp/rst_body.json -w 'status=%{http_code}\ttime=%{time_total}\tsize=%{size_download}\n' \
  -X POST "$BASE_URL/items" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"name":"test","price":10}'
```

---

## Rate limit e concorrência

- Padrão: `sleep 0.15` (≥150ms) entre requisições; nunca dispare em paralelo
  agressivo contra o mesmo host.
- Se aparecer `429`, **aumente** o atraso e registre que o alvo tem rate limit
  ativo (isso é um dado positivo para o relatório).

---

## Verbos destrutivos

`DELETE`, e `PUT`/`PATCH` que mudem estado de forma ampla, **não disparam sem
confirmação explícita do usuário, rota a rota**. Antes de executar, liste ao
usuário exatamente o que será chamado (método + URL) e aguarde o "sim".

---

## Registro por requisição

Para cada chamada, guarde a linha que vai virar `responses-table.md`:

```
Rota | Método | Cenário | Status | Tempo(ms) | Tamanho | Observação
```

Exemplos de "Observação": "corpo retornou campos esperados",
"500 com stack trace vazado", "aceitou preço negativo (deveria 422)".
