# AUTH & ACCESS CONTROL — matriz configurável

Objetivo: sobre o esquema de auth **declarado pelo usuário**, rodar uma matriz
de cenários e comparar status esperado vs. observado. Divergência = achado.
Nunca assuma um modelo específico; a matriz se adapta ao esquema.

---

## Esquemas suportados e como manipular a credencial

| Esquema | Onde vai a credencial | Como enviar (curl) |
|---|---|---|
| Bearer token | header `Authorization` | `-H "Authorization: Bearer $TOKEN"` |
| Cookie de sessão | header `Cookie` | `-H "Cookie: session=$SID"` |
| API key (header) | header custom | `-H "X-API-Key: $KEY"` |
| API key (query) | query string | `?api_key=$KEY` |
| Basic | header `Authorization` | `-u "$USER:$PASS"` |
| OAuth | header `Authorization` (Bearer) | `-H "Authorization: Bearer $ACCESS"` |
| Custom (ex.: wallet/assinatura) | conforme declarado | header/param que o usuário indicar |

> Auth por wallet ou qualquer modelo incomum é tratado **apenas** como "custom":
> peça ao usuário como a credencial é montada e enviada. Não presuma nada.

---

## Matriz de cenários

Rode cada cenário contra rotas protegidas representativas (uma pública de
controle, uma protegida comum, uma privilegiada/admin).

| # | Cenário | Como forjar | Status esperado |
|---|---|---|---|
| 1 | Sem credencial | omitir header/cookie/key | `401` |
| 2 | Credencial inválida | token/keys aleatórios/malformados | `401` |
| 3 | Credencial expirada | token com `exp` no passado (se disponível) | `401` |
| 4 | Credencial adulterada | alterar 1 claim/assinatura do token | `401` |
| 5 | Papel/escopo insuficiente | credencial válida de papel baixo em rota privilegiada | `403` |
| 6 | Escalonamento horizontal | usuário A acessando recurso de usuário B (mesmo nível) | `403`/`404` |
| 7 | Escalonamento vertical | usuário comum tentando ação de admin | `403` |

Para tokens JWT (Bearer/OAuth), cenários 3 e 4 são possíveis se você tiver como
gerar/editar o token de teste. Se não, marque a célula como `n/a` e registre a
limitação no relatório.

---

## Interpretação — o que é achado

| Observado | Esperado | Veredito |
|---|---|---|
| `200`/`201` | `401` | **Crítico** — rota protegida acessível sem auth |
| `200` | `403` | **Alto** — bypass de autorização / IDOR |
| Recurso de B retornado ao A | `403`/`404` | **Alto** — escalonamento horizontal (IDOR) |
| Ação admin executada por comum | `403` | **Crítico** — escalonamento vertical |
| `403` correto | `403` | ok — controle funcionando |
| Token adulterado aceito | `401` | **Crítico** — verificação de assinatura falha |
| Msg de erro vaza detalhe (usuário existe/não) | — | **Baixo/Médio** — user enumeration |

---

## Cuidados

- Cenários 6 e 7 precisam de **duas identidades de teste** (dois usuários /
  dois papéis). Peça-as ao usuário se não existirem; não crie contas no alvo
  sem autorização.
- Aplique rate limit também aqui.
- Nunca use credenciais reais de produção como "credencial válida" sem
  autorização explícita.

---

## Registro

Cada célula da matriz vira uma linha em `responses-table.md` (Cenário = nome do
cenário da matriz) e, quando houver divergência, um item na seção "Achados de
segurança" de `notes.md` com severidade.
