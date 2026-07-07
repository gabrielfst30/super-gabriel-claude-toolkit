# REPORT — templates da tabela, notes e bloco agent-ready

Consolide os modos rodados em **2 arquivos + 1 saída opcional**. Use os
templates abaixo literalmente (colunas e ordem de seções são fixas).

**Onde salvar:** sempre dentro de uma pasta dedicada, nunca soltos na raiz —
`security-reports/<AAAA-MM-DD>-<alvo-slug>/` por default (`mkdir -p` antes de
escrever), ou o diretório passado em `--out <path>` se informado. Sugira
adicionar `security-reports/` ao `.gitignore` quando a pasta for criada pela
primeira vez.

---

## 1. `responses-table.md`

Colunas exatamente nesta ordem:

```markdown
# Tabela de Respostas

| Rota | Método | Cenário | Status | Tempo(ms) | Tamanho | Observação |
|------|--------|---------|--------|-----------|---------|------------|
| /items | GET | válido feliz | 200 | 42 | 1.2KB | corpo com campos esperados |
| /items | POST | campo faltando | 422 | 18 | 210B | validação ok |
| /items/:id | DELETE | sem credencial | 401 | 12 | 95B | protegido corretamente |
| /items/:id | GET | id inexistente | 404 | 15 | 88B | ok |
| /admin/x | GET | papel insuficiente | 200 | 40 | 900B | ⚠️ deveria ser 403 |
```

Uma linha por requisição disparada. "Cenário" = nome do caso (functional) ou da
célula da matriz (auth) ou da classe de probe (injection).

---

## 2. `notes.md` — seções na ordem fixa

```markdown
# Notas de Teste — <alvo>

## 1. Visão geral do teste
- **Alvo:** <baseUrl>
- **Autorização:** <como o alvo foi autorizado>
- **Escopo:** <rotas/recursos cobertos>
- **Modos rodados:** <discovery/functional/auth/injection/...>
- **Data:** <data>

## 2. Comportamento por rota
- `GET /items` — <resumo do comportamento observado>
- `POST /items` — <...>
- `DELETE /items/:id` — <...>

## 3. Achados de segurança
> Cada achado com severidade: baixa | média | alta | crítica.

- **[CRÍTICA]** `GET /admin/x` acessível com papel comum (retornou 200, esperado
  403). Escalonamento vertical. Rota: /admin/x | Cenário: papel insuficiente.
- **[ALTA]** `id` em `GET /items/:id` reflete erro de SQL com payload `'`.
  Possível SQLi. Evidência: mensagem "syntax error near ..." no corpo.
- **[MÉDIA]** Stack trace vazado em `500` de `POST /items` com body malformado.
- **[BAIXA]** Enumeration de usuário via mensagens de erro distintas no login.

## 4. Sugestões de performance
- <ex.: /items sem paginação retorna corpo grande; adicionar limit/offset>
- <ex.: rota X com tempo médio alto (>500ms), avaliar índice/consulta>

## 5. Sugestões de segurança
- <ex.: aplicar checagem de autorização por papel em /admin/*>
- <ex.: parametrizar queries / usar ORM; validar e escapar input>
- <ex.: ocultar stack traces em produção; retornar 4xx genérico>

## 6. Sugestões de estrutura
- <ex.: padronizar formato de erro (envelope consistente)>
- <ex.: agrupar rotas por recurso; versionar API (/v1)>
```

Severidade — critério rápido:
- **crítica**: bypass de auth, RCE/command injection, SQLi explorável, esc. vertical.
- **alta**: XSS refletido, IDOR/esc. horizontal, path traversal, SQLi provável.
- **média**: info disclosure (stack/versão), config fraca, CORS permissivo.
- **baixa**: enumeration, `500` em input malformado, headers de segurança ausentes.

---

## 3. Saída AGENT-READY (opcional)

Só gere se o usuário pedir. É um **bloco de prompt reutilizável** resumindo
achados e ações — pronto para outra skill/agente consumir. **Esta skill APENAS
produz o texto; não cria nem orquestra agentes.**

```markdown
## CONTEXTO
Foi executado um teste de rotas e segurança em <alvo> (escopo: <recursos>).
Modos rodados: <lista>.

## ACHADOS (por severidade)
- [CRÍTICA] <achado> — rota <x>, cenário <y>.
- [ALTA] <achado> — rota <x>.
- [MÉDIA] <achado>.
- [BAIXA] <achado>.

## AÇÕES SUGERIDAS (priorizadas)
1. <corrigir X — rota/arquivo provável, tipo de fix>
2. <corrigir Y>
3. <melhoria de estrutura/performance>

## RESTRIÇÕES
- Confirmar em ambiente autorizado antes de re-testar.
- Não aplicar mudanças destrutivas sem revisão.
```

O bloco deve ser autossuficiente: quem receber não teve acesso à conversa.
Inclua rota, evidência resumida e ação para cada achado relevante.
