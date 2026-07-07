# INJECTION & FUZZING — wordlists de probes + sinais de vuln

**Nível DETECÇÃO, não exploração.** O objetivo é revelar se o input é
sanitizado, usando vetores canônicos do tipo empregado por ferramentas DAST
legítimas para avaliar a própria aplicação. Não encadeie exploração, não
exfiltre dados, não escale. Só em **alvo autorizado**; nunca produção sem
autorização explícita. Aplique rate limit.

Para cada parâmetro injetável (query, body field, path segment), envie os
probes da classe relevante e interprete a resposta pela tabela de sinais.

---

## Wordlists de probe (detecção)

### SQL injection (SQLi)
```
'
"
' OR '1'='1
' OR 1=1--
'; --
1 OR 1=1
' AND SLEEP(0)--        (baseline; compare tempo com um SLEEP maior só se autorizado)
```
Sinal: erro de SQL vazado, mudança de resultado com `OR 1=1`, diferença de
tempo em probes time-based.

### NoSQL injection (NoSQLi)
```
{"$gt": ""}
{"$ne": null}
{"$where": "true"}
[$ne]=1                 (em query string)
```
Sinal: bypass de filtro/auth, retorno de mais registros que o esperado, erro do
driver Mongo/etc.

### XSS refletido
```
<script>alert(1)</script>
"><svg/onload=alert(1)>
'"><img src=x onerror=alert(1)>
javascript:alert(1)
```
Sinal: payload **refletido sem escape** no corpo HTML/JSON da resposta
(procure a string literal de volta, com `<`/`>`/`"` intactos).

### Command injection
```
; id
| id
`id`
$(id)
&& whoami
```
Sinal: saída de comando no corpo, delay anômalo, erro de shell. **Não** rode
comandos destrutivos — use apenas `id`/`whoami` como sonda inócua.

### Path traversal
```
../../../../etc/passwd
..%2f..%2f..%2fetc%2fpasswd
....//....//etc/passwd
%2e%2e%2f
```
Sinal: conteúdo de arquivo do sistema no corpo, ou erro de path revelando
estrutura de diretórios.

### Malformado / oversized
```
- JSON quebrado: {"a":                  → espera 400
- Campo gigante: string de 1MB          → espera 413/400/422, não 500
- Tipo trocado: número onde espera obj  → espera 400/422
- Encoding: %00 (null byte), UTF-8 inválido
- Números extremos: 9e999, -1, 0, MAX_INT+1
```
Sinal de problema: `500` (não tratado) onde deveria ser `4xx`; crash;
timeout.

---

## Tabela de sinais → veredito

| Sinal na resposta | Interpretação | Severidade base |
|---|---|---|
| Erro de SQL/driver vazado (sintaxe, tabela, coluna) | input chega cru ao DB | **Alta** |
| `OR 1=1` muda o conjunto de resultados | SQLi provável | **Crítica** |
| Probe time-based altera tempo de forma consistente | SQLi blind provável | **Alta** |
| `$ne`/`$gt` retorna dados que deveriam ser filtrados | NoSQLi | **Alta** |
| Payload XSS refletido sem escape | XSS refletido | **Alta** |
| Saída de `id`/`whoami` no corpo | command injection | **Crítica** |
| Conteúdo de `/etc/passwd` ou similar | path traversal | **Crítica** |
| Stack trace / caminho absoluto / versão de framework | info disclosure | **Média** |
| `500` em input malformado | falta de validação/tratamento | **Baixa/Média** |
| Payload **escapado** ou rejeitado com `4xx` limpo | input sanitizado | — (ok) |

Severidade final ajusta pelo contexto: rota autenticada vs. pública, dado
sensível envolvido, exploitabilidade real.

---

## Regras de reporte

- Reporte: **rota + parâmetro + classe de probe + sinal observado**. Um item de
  achado por combinação vulnerável.
- Não inclua PoC armado nem instruções de exploração — só o vetor de detecção e
  a evidência do sinal.
- Se nada dispara sinal, registre "input sanitizado" como resultado positivo no
  relatório (é informação útil).
