# PHP MVC (no-framework)

A small object-oriented MVC framework in plain PHP, built from scratch as a learning project. No Composer packages: routing, dependency injection, database access, views, sessions and CSRF protection are all written in this repository.

This README documents what exists **today**, why it was designed that way, and how to run and extend it.

---

## Requirements

- PHP 8.2+ (uses constructor promotion, `match`, `str_contains`, named arguments)
- MySQL or MariaDB with the **InnoDB** engine (transactions do not work on MyISAM)
- PDO MySQL extension
- A web server whose document root is `public/` (Laravel Herd, Apache, Nginx, or `php -S`)

---

## Project structure

```
.
├── .env                      # Local config and secrets (NOT committed)
├── .env.example              # Documents the required keys (committed)
├── public/
│   ├── index.php             # Front controller: requires src/bootstrap.php
│   └── .htaccess             # Sends non-file requests to index.php (Apache)
└── src/
    ├── bootstrap.php         # Autoloader, config, container, routes, dispatch
    ├── App/                  # Application code (yours)
    │   ├── Controllers/
    │   │   ├── HomeController.php
    │   │   └── ArticleController.php
    │   ├── Models/
    │   │   └── HomeModel.php          # Table: article
    │   └── Views/
    │       ├── layouts/app.php
    │       ├── home/index.php
    │       ├── articles/              # show, create, edit, _form
    │       └── errors/                # 403, 404, 405, 500
    └── Framework/            # Reusable framework code
        ├── Router.php
        ├── Dispatcher.php
        ├── Container.php
        ├── Request.php
        ├── Response.php
        ├── View.php
        ├── Session.php
        ├── Csrf.php
        ├── Validator.php
        ├── Database.php
        ├── Model.php
        ├── Dotenv.php
        ├── ErrorHandler.php
        └── Exceptions/
            ├── HttpException.php
            ├── NotFoundException.php
            ├── ForbiddenException.php
            ├── MethodNotAllowedException.php
            └── ContainerException.php
```

Every class lives in a namespace matching its folder (`Framework\Router` is `src/Framework/Router.php`). The autoloader in `bootstrap.php` maps namespace to path, so no Composer is needed.

**Rule of thumb:** `Framework/` knows nothing about your application. `App/` uses `Framework/`, never the other way around.

---

## Setup

1. Copy `.env.example` to `.env` and fill in your values.
2. Create the database and tables (see [Database schema](#database-schema)).
3. Point your web server's document root at `public/`.
4. Open the site in the browser.

Without a configured server: `php -S localhost:8000 -t public` from the project root.

### Environment variables

```
APP_DEBUG=true

DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=
DB_USERNAME=
DB_PASSWORD=
DB_CHARSET=utf8mb4
```

- `APP_DEBUG=true` shows full error details and stack traces. **Set it to `false` in production.**
- Use `127.0.0.1` rather than `localhost` for `DB_HOST` on Windows. `localhost` can make PDO try a named pipe instead of TCP.
- `.env` must be listed in `.gitignore`. Never commit real credentials.

### Local environment notes (Windows)

- Laravel Herd (free) serves PHP but has no bundled MySQL. XAMPP's MySQL can run on its own on port 3306 while Herd serves the site.
- Do not run XAMPP's Apache on port 80 at the same time as Herd.
- Do not put a root-level `.htaccess` that rewrites into `public/` when the vhost already points at `public/`. The two together cause a rewrite loop and a `400 Bad Request`.

---

## Request lifecycle

```
public/index.php
  └─ src/bootstrap.php
       1. Register autoloader
       2. Load .env (Dotenv)
       3. Create View, register ErrorHandler
       4. Build Container (bind Database, View, Session)
       5. Register routes on the Router
       6. Build Request from PHP globals
       7. Dispatcher::dispatch($request)
            ├─ match URI + method against routes
            ├─ resolve controller through the Container (constructor DI)
            ├─ resolve action arguments (Request + route params)
            ├─ call the action
            └─ send the returned Response
```

The `ErrorHandler` is registered before anything else can fail, so errors during setup are also handled.

---

## Components

### Router

Stores routes only. It does not match anything.

```php
$router->get("/articles/{id:int}", [ArticleController::class, "show"]);
$router->post("/articles", [ArticleController::class, "store"]);
```

### Dispatcher

Matches the request against the registered routes and calls the controller.

- **Typed route parameters** are converted to anchored regexes (`^...$`) with the `u` flag for UTF-8.

  | Syntax | Pattern | Passed to controller as |
  |---|---|---|
  | `{name}` | any characters except `/` | `string` |
  | `{id:int}` | digits only | `int` |
  | `{slug:slug}` | `[a-z0-9-]+` | `string` |
  | `{token:uuid}` | UUID format | `string` |
  | `{word:alpha}` | letters only | `string` |

- An **unknown type** (for example `{id:itn}`) throws an `InvalidArgumentException`. This fails loudly because it is a developer mistake, not user input.
- **Trailing slashes** are removed before matching, so `/users/` and `/users` are the same route. Matching is **case-sensitive**.
- **404 vs 405:** if the URI matches a route but not for this HTTP method, a `MethodNotAllowedException` is thrown (405 with an `Allow` header). If nothing matches, a `NotFoundException` is thrown (404).
- **`HEAD`** requests are matched against `GET` routes and sent without a body.
- **Action arguments** are resolved with reflection: a parameter type-hinted `Request` receives the current request, and the remaining parameters are filled from route parameters in order.

```php
public function update(Request $request, int $id): Response
```

- An action may return a `Response`, a plain `string` (sent as HTML), or nothing (for an older action that echoes directly).

### Container

Dependency injection with **autowiring** through reflection.

- Resolves constructor dependencies recursively by type hint.
- Scalar parameters can't be autowired. They use their default value, or the container throws a `ContainerException`.
- Detects circular dependencies and reports the chain.
- `bind($abstract, $factory)` creates a new instance each time. `singleton($abstract, $factory)` creates it once.
- Instances are built only when first requested, so a request that never touches the database never connects to it.

### Request

An immutable wrapper around `$_GET`, `$_POST`, `$_SERVER`, cookies, files and headers, created once with `Request::fromGlobals()`.

`method()`, `uri()` (path only, decoded with `rawurldecode`), `query()`, `post()`, `input()`, `string()`, `header()`, `cookie()`, `file()`, `server()`, `isJson()`.

`string($key)` returns a trimmed string, or `""` if the value is missing or not a string. This avoids warnings when someone sends `title[]=x`.

Controllers should read input through `Request`, not through the superglobals.

### Response

Holds a body, status code and headers. `send()` is the only place that calls `http_response_code()`, `header()` and `echo`.

```php
Response::html($body, 200);
Response::json(["status" => "ok"]);
Response::redirect("/articles/5", 303);   // use 303 after a POST
```

`withHeader()`, `withHeaders()` and `withStatus()` return modified copies (immutable).

> Never pass user input directly to `Response::redirect()`. That is an open-redirect vulnerability. Only redirect to paths you write, or validate the value first.

### View

Plain PHP templates under `src/App/Views`. There is no template language.

```php
$this->view->render("articles/show", ["article" => $article], "layouts/app");
```

- `render()` **returns a string** (it does not echo). This keeps headers and status codes controllable, and lets the layout and error pages wrap the inner content.
- Variables in the data array become local variables in the template.
- Dots or slashes work in names: `"home.index"` and `"home/index"` are equivalent.
- The optional third argument is a layout. The rendered view is passed to the layout as `$content`, together with the same data.
- Inside templates, `$this` is the `View`, so `$this->e($value)` escapes output and `$this->render(...)` includes a partial.
- **Escape at output.** Use `$this->e()` for every value that came from the database or the user. `$content` and rendered partials are already-built HTML, so they are echoed without escaping.
- Output buffering is cleaned up if a template throws.

### Database and Model

`Database` creates a PDO connection with exceptions enabled, associative fetch mode, and **real prepared statements** (`ATTR_EMULATE_PREPARES => false`). Connection errors are wrapped so credentials never reach an error page.

`Model` is an abstract base class. A child declares its table:

```php
class HomeModel extends Model
{
    protected string $table = "article";
}
```

Inherited methods:

| Method | Purpose |
|---|---|
| `find($id)` | One row, or `false` |
| `all()` | All rows |
| `create(array $data)` | Insert one row, returns the new id |
| `createMany(array $rows)` | Insert many rows in one statement |
| `update($id, array $data)` | Update one row |
| `updateMany(array $updates)` | Update several rows (`id => data`) inside a transaction |
| `delete($id)` | Delete one row |
| `deleteMany(array $ids)` | Delete several rows in one statement |
| `transaction(callable $fn)` | Run a callback in a transaction, rolling back on any `Throwable` |

**Important security note:** table and column names cannot be bound as parameters, so `create()` and `update()` place the **array keys directly into the SQL**. Values are always bound, but keys are not. Therefore callers must whitelist the fields themselves and never pass raw request data. `ArticleController::input()` does this.

Put table-specific queries (`findByEmail()`, `published()`) in the child Model, using `$this->connection`.

### ErrorHandler

Registers handlers for PHP errors (converted to `ErrorException`), uncaught exceptions, and fatal errors at shutdown.

- `HttpException` subclasses set the response status (and headers, such as `Allow`). Anything else is a 500.
- Every exception is logged with `error_log()`.
- **Debug mode** (`APP_DEBUG=true`): shows class, message, file, line and trace, HTML-escaped.
- **Production mode:** renders `errors/<status>.php` inside the layout. If that rendering fails, it falls back to plain HTML so an error page can never produce a blank page.
- Statuses with a template: 403, 404, 405, 500. Any other status uses the 500 page.

Exceptions: `NotFoundException` (404), `ForbiddenException` (403), `MethodNotAllowedException` (405), all extending `HttpException`.

### Session

A wrapper around `$_SESSION`, started lazily on first use, so pages that never touch it send no cookie.

- Cookie flags: `HttpOnly`, `SameSite=Lax`, and `Secure` when the request is HTTPS. Cookie-only IDs with strict mode.
- `get()`, `set()`, `has()`, `remove()`, `pull()`, `regenerateId()`, `destroy()`.
- **Flash data** lasts exactly one following request: `flash($key, $value)` then `getFlash($key)`, `hasFlash($key)`.
- Call `regenerateId()` whenever privilege changes (for example at login).

### Csrf

`token()` creates one random token per session. `isValid($submitted)` compares in constant time with `hash_equals()`. Every state-changing form includes it:

```html
<input type="hidden" name="_token" value="<?= $this->e($csrf_token) ?>">
```

### Validator

```php
$errors = Validator::validate($data, [
    "title" => ["required", "max:128"],
    "published_at" => ["datetime"],
]);
```

Rules: `required`, `min:n`, `max:n`, `datetime` (the `datetime-local` format, rejecting impossible dates). Returns `field => first error message`. An unknown rule throws. Optional fields are skipped when empty.

---

## Routes

| Method | URI | Action | Purpose |
|---|---|---|---|
| GET | `/` | `HomeController::index` | List articles |
| GET | `/articles/create` | `ArticleController::create` | New article form |
| POST | `/articles` | `ArticleController::store` | Save new article |
| GET | `/articles/{id:int}` | `ArticleController::show` | Show article |
| GET | `/articles/{id:int}/edit` | `ArticleController::edit` | Edit form |
| POST | `/articles/{id:int}/update` | `ArticleController::update` | Save changes |
| POST | `/articles/{id:int}/delete` | `ArticleController::destroy` | Delete article |

HTML forms only send GET and POST, so update and delete are separate POST routes instead of using method override. `/articles/create` does not conflict with `/articles/{id:int}` because `create` is not numeric.

---

## Form flow (create / edit / delete)

1. **Show form:** the controller gives the view a CSRF token, plus any flashed `errors` and `old` input.
2. **Submit (POST):** verify the CSRF token (403 on failure), whitelist the input fields, validate.
3. **Validation fails:** flash `errors` and `old`, then **redirect (303)** back to the form.
4. **Validation passes:** save through the Model, flash a success message, then redirect (303) to the article.

Redirecting after every POST means refreshing the page can't resubmit the form. Deleting is always a POST with a CSRF token, never a link, because GET requests must not change data.

---

## Database schema

Tables used so far. The `article` columns come from the live schema. All tables must use **InnoDB**.

```sql
CREATE TABLE article (
    id           INT(11)      NOT NULL AUTO_INCREMENT PRIMARY KEY,
    title        VARCHAR(128) NOT NULL,
    content      TEXT         NOT NULL,
    published_at DATETIME     NULL,
    image_file   VARCHAR(200) NULL,
    INDEX (title)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

Also present (many-to-many between articles and categories, not yet used by the code):

- `category` (`id`, `name`)
- `article_category` (`article_id`, `category_id`), both foreign keys with `ON DELETE CASCADE` and `ON UPDATE CASCADE`

Check the engine of a table with `SHOW TABLE STATUS WHERE Name = 'article';` and convert it with `ALTER TABLE article ENGINE = InnoDB;`.

---

## Conventions

- `declare(strict_types=1);` in every file.
- Class names are `PascalCase`, methods `camelCase`, properties and variables `snake_case`.
- Use `private` for internals of a class that is not meant to be extended. Use `protected` where subclasses need access (for example in `Model`).
- Inside a namespace, global classes need a leading backslash or a `use` import (`\InvalidArgumentException`, `use Throwable;`). A missing import in `catch (Throwable $e)` silently stops the catch from working. This caused a real bug where transaction rollback did not run.
- Controllers never touch `$_GET`, `$_POST` or `$_SESSION` directly.
- Escape on output, in views. Do not sanitize on input.
- Failures caused by developer mistakes should throw loudly. Failures caused by visitors (bad URLs, bad input) become 4xx responses or validation errors.

---

## Testing

There is no automated test suite yet. Manual checks that were used:

```bash
curl -i http://php-mvc.test/                           # 200
curl -I http://php-mvc.test/                           # 200, headers only (HEAD)
curl -i http://php-mvc.test/articles/9999              # 404 (article missing)
curl -i http://php-mvc.test/articles/abc               # 404 (fails :int)
curl -i http://php-mvc.test/articles                   # 405, Allow: POST
curl -i -X POST http://php-mvc.test/articles -d "title=x&content=y"   # 403 (no CSRF token)
```

On Windows PowerShell use `curl.exe`, because `curl` is an alias for `Invoke-WebRequest`.

For a form POST with curl, load the form first with a cookie jar (`-c jar.txt`), copy the `_token` value, then submit with `-b jar.txt` and `--data-urlencode`.

---

## Known limitations

- No authentication yet: anyone can create, edit and delete articles.
- `HomeModel` is really the article model. A rename to `ArticleModel` is pending.
- No pagination on the article list.
- `Model::create()` / `update()` do not restrict column names (see the security note above).
- `Model::updateMany()` uses `transaction()`, which does not support being nested inside another transaction.
- Only one level of layout, with no default layout, and no shared view data.
- Sessions use PHP's default file storage.
- The `image_file` column and the category tables are not used by the application yet.
- No middleware, so there is no way yet to protect a route declaratively.
