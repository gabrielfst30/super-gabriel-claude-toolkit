# DISCOVERY — extrair rotas por framework

Objetivo: ler o código-fonte em `--src` e produzir um **mapa de rotas**
estruturado. Detecte a linguagem/framework primeiro; depois aplique os padrões
de extração. Se nada casar, use o **fallback genérico**.

Para cada rota, capture os campos do mapa:

```
método | path | path params | query params | body/schema | auth (público|papel/escopo)
```

---

## Passo 1 — Detectar a stack

Âncoras rápidas (Glob/Grep) para identificar a stack:

| Sinal (arquivo/dependência) | Stack provável |
|---|---|
| `package.json` + `express` | Node/Express |
| `package.json` + `fastify` | Node/Fastify |
| `package.json` + `@nestjs/*` | Node/NestJS |
| `requirements.txt`/`pyproject.toml` + `flask` | Python/Flask |
| `+ fastapi` | Python/FastAPI |
| `manage.py` / `django` | Python/Django |
| `go.mod` | Go |
| `pom.xml`/`build.gradle` + `spring` | Java/Spring |
| `Gemfile` + `rails` | Ruby/Rails |
| `composer.json` + `laravel/framework` | PHP/Laravel |
| `*.csproj` + `Microsoft.AspNetCore` | .NET |
| `Cargo.toml` + `actix-web` | Rust/actix-web |
| `Cargo.toml` + `axum`/`rocket` | Rust (→ fallback genérico) |

---

## Passo 2 — Padrões de extração por framework

### Node / Express / Fastify
Grep por chamadas de método:
```
app.get(  app.post(  app.put(  app.patch(  app.delete(
router.get(  router.post(  ...
fastify.route({ method: ... , url: ... })
```
- Path: primeiro argumento string. Params de path: `:id`, `:slug`.
- Auth: middleware na cadeia (ex.: `authMiddleware`, `requireAuth`, `passport`,
  `verifyToken`) — anote o nome, não presuma o esquema.
- Body: procure uso de `req.body.<campo>` ou schema (Zod/Joi/JSON Schema).

### Node / NestJS
Decorators:
```
@Controller('prefixo')   @Get(':id')   @Post()   @Put()   @Patch()   @Delete()
@Query()  @Param()  @Body()
@UseGuards(...)   → indica rota protegida (anote o guard)
```
- Path efetivo = prefixo do `@Controller` + path do método.

### Python / Flask
```
@app.route('/x', methods=['GET','POST'])
@blueprint.route(...)
```
- Params: `<int:id>`, `<slug>`. Auth: decorators tipo `@login_required`,
  `@jwt_required`.

### Python / FastAPI
```
@app.get('/x')  @app.post(...)  @router.get(...)
def handler(id: int, q: str = None, body: Model = ...)
Depends(...)   → dependência de auth/permissão (anote)
```
- Query/body inferidos da assinatura e dos modelos Pydantic.

### Python / Django
- `urls.py`: `path('x/', view)`, `re_path(...)`. DRF: `@api_view([...])`,
  `ViewSet`/`Router`. Auth: `permission_classes`, `IsAuthenticated`.

### Go
- Roteadores comuns: `net/http` (`http.HandleFunc`), `gorilla/mux`
  (`r.HandleFunc(...).Methods(...)`), `chi` (`r.Get/Post/...`), `gin`
  (`r.GET/POST/...`), `echo` (`e.GET/POST/...`).
- Middleware de auth: procure `.Use(...)` e wrappers de handler.

### Java / Spring
```
@RestController  @RequestMapping('/prefixo')
@GetMapping  @PostMapping  @PutMapping  @PatchMapping  @DeleteMapping
@PathVariable  @RequestParam  @RequestBody
@PreAuthorize(...) / Spring Security → auth
```

### Ruby / Rails
- `config/routes.rb`: `resources :x`, `get 'x/:id'`. Expanda `resources` para
  index/show/create/update/destroy. Auth: `before_action :authenticate_*`.

### PHP / Laravel
- `routes/web.php` / `routes/api.php`: `Route::get('x', ...)`,
  `Route::apiResource(...)`. Auth: middleware `->middleware('auth:...')`.

### .NET
- Minimal API: `app.MapGet/MapPost/...`. Controllers: `[HttpGet]`,
  `[Route]`, `[Authorize]`. Params via binding.

### Rust / actix-web
```
#[get("/x")]  #[post("/x")]  ...
.route("/x", web::get().to(handler))
web::Path, web::Query, web::Json
```

---

## Passo 3 — Fallback genérico (framework não reconhecido)

Quando a stack não casa (inclui Rust axum/rocket e stacks exóticas), use, nesta
ordem de preferência:

1. **Spec OpenAPI existente** no repo (`openapi.yaml|json`, `swagger.*`) — é a
   fonte de verdade; parse os paths/métodos/params direto dela.
2. **Anotações/macros/decorators de rota** — grep genérico por tokens de
   roteamento (`route`, `get`, `post`, `handler`, `#[...]`, `@...Mapping`) e
   monte o mapa manualmente com o que for legível.
3. **Lista manual** — peça ao usuário a lista de rotas (método + path + auth).
   Este é o caminho garantido quando o código não é parseável com confiança.

Nunca invente rotas. Se um campo (ex.: requisito de auth) não puder ser
determinado do código, marque como `desconhecido` no mapa e siga.

---

## Saída do modo

Mapa de rotas em tabela, uma linha por rota, alimentando FUNCTIONAL, AUTH,
INJECTION e COLLECTIONS:

```
| Método | Path            | Path params | Query params | Body            | Auth               |
|--------|-----------------|-------------|--------------|-----------------|--------------------|
| GET    | /items/:id      | id          | —            | —               | Bearer (qualquer)  |
| POST   | /items          | —           | —            | {name, price}   | Bearer (admin)     |
| DELETE | /items/:id      | id          | —            | —               | Bearer (admin)     |
```
