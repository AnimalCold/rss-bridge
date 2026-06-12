# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

RSS-Bridge is a PHP web application that generates RSS/Atom (and other) feeds for
websites that don't offer them. Each supported site is implemented as a "bridge".
The same `index.php` entry point serves both as a web server endpoint and as a CLI
tool. Minimum PHP version is 7.4 (the Composer platform is pinned to 7.4, so do not
use newer language features).

## Commands

Dev dependencies must be installed first: `composer install`.

- Run all tests: `./vendor/bin/phpunit` (or `composer test`)
- Run a single test class: `./vendor/bin/phpunit --filter UrlTest`
- Lint (coding style): `composer lint`
  (`./vendor/bin/phpcs --standard=phpcs.xml --warning-severity=0 --extensions=php -p ./`)
- Auto-fix lint violations: `./vendor/bin/phpcbf --standard=phpcs.xml ./`
- PHP version compatibility check: `composer compat`
- Run a local dev server: `php -S 127.0.0.1:9001` then browse http://127.0.0.1:9001/
- Invoke a bridge from CLI: `php index.php action=display bridge=<Name> format=Atom`
- Clear / prune cache: `bin/cache-clear` / `bin/cache-prune`

Coding style is PSR-12 (enforced via `phpcs.xml`). Two commits are excluded from
git blame in `.git-blame-ignore-revs` (the PSR-12 reformat).

## Request lifecycle (the big picture)

`index.php` is the single entry point for both web and CLI. It:
1. Loads `lib/bootstrap.php` (sets up autoloading + procedural helper files),
   `lib/config.php` (reads `config.ini.php`, falling back to `config.default.ini.php`),
   and `lib/dependencies.php` (a small DI `Container`).
2. Builds a `Request` from globals or from CLI args (`Request::fromCli`).
3. `RssBridge::main()` maps the `action` query param to an Action class
   (e.g. `action=display` -> `DisplayAction`), then runs it through a fixed
   middleware stack before invoking the handler.

Middleware stack (see `lib/RssBridge.php`, applied in `array_reverse` order):
`BasicAuthMiddleware`, `CacheMiddleware`, `ExceptionMiddleware`, `SecurityMiddleware`,
`MaintenanceMiddleware`, `TokenAuthenticationMiddleware`.

## Core directories

Autoloading is split: `lib/bootstrap.php` `require`s a fixed list of *procedural*
helper files (`lib/utils.php`, `lib/http.php`, `lib/html.php`, `lib/contents.php`,
`lib/url.php`, vendored `parsedown`/`simplehtmldom`/`php-urljoin`, etc.), and
registers an `spl_autoload_register` that resolves *classes* by bare class name from
`actions/`, `bridges/`, `caches/`, `formats/`, `lib/`, and `middlewares/`. Class name
must equal the file name. There are no PHP namespaces in app code (tests do use the
`RssBridge\Tests\` namespace).

- `bridges/` — ~500 bridge classes, one per supported site. This is where the vast
  majority of changes happen.
- `actions/` — request handlers (`Display`, `Detect`, `Findfeed`, `Frontpage`,
  `List`, `Connectivity`, `Health`). Each implements `ActionInterface` and is
  `__invoke`-able.
- `formats/` — output serializers extending `FormatAbstract` (`Atom`, `Mrss`, `Json`,
  `Html`, `Plaintext`, `Sfeed`). Selected via the `format` query param.
- `caches/` — cache backends implementing `CacheInterface` (`File`, `SQLite`,
  `Memcached`, `Array`, `Null`). Chosen by `[cache] type` in config.
- `middlewares/` — request middlewares (auth, caching, security, maintenance).
- `lib/` — framework code: `RssBridge`, `Container` (DI), `Configuration`,
  `BridgeAbstract`, the `*Factory` classes, `FeedItem`, `FeedExpander`, `Request`/
  `Response`, `XPathAbstract`, `WebDriverAbstract`.
- `templates/` — PHP HTML templates rendered via the `render()` helper.
- `tests/` — PHPUnit tests (see below).

## How bridges work

A bridge extends `BridgeAbstract` (or `FeedExpander` for bridges that augment an
existing upstream feed, or `XPathAbstract`/`CssSelectorBridge` for scraping). The
contract:

- Class name ends in `Bridge` and lives in `bridges/<Name>Bridge.php`.
- Metadata via class consts: `NAME`, `URI`, `DESCRIPTION`, `MAINTAINER`,
  `CACHE_TIMEOUT` (seconds), and `PARAMETERS` (user-facing input fields, grouped by
  "context"). `CONFIGURATION` declares instance-level config (e.g. API keys).
- Implement `collectData()` to populate `$this->items[]` with feed items.
- Read user inputs with `$this->getInput('name')`; current context is
  `$this->queriedContext`.

Expected feed item shape (keys are optional unless noted):
```php
$this->items[] = [
    'uri'        => 'https://example.com/blog/hello', // link to the item
    'title'      => 'Hello world',
    'timestamp'  => 1668706254,    // unix timestamp or strtotime-parsable string
    'author'     => 'Alice',
    'content'    => 'item html',
    'enclosures' => ['https://example.com/foo.png'], // media attachments
    'categories' => ['news', 'tech'],
    'uid'        => 'e7147580c8747aad', // stable globally-unique id
];
```

Fetch helpers (defined in the procedural `lib/contents.php` / `lib/html.php`, so
available globally without `use`): `getContents()`, `getSimpleHTMLDOM()`,
`getSimpleHTMLDOMCached()`. Prefer these over raw curl — they honour the configured
HTTP client, proxy, and caching.

New code files MUST start with:
```php
<?php

declare(strict_types=1);
```

## Tests

PHPUnit config is `phpunit.xml`, bootstrapped from `lib/bootstrap.php`; the test
suite is the `tests/` directory. Notable suites:
- `BridgeImplementationTest` / `BridgeFactoryTest` — structural validation that every
  bridge in `bridges/` is well-formed (correct consts, valid `PARAMETERS`, etc.). Run
  these after adding or editing a bridge.
- `FormatTest`, `CacheImplementationTest`, `FeedItemTest`, `UrlTest`, `UtilsTest`,
  `ParameterValidatorTest`, `ConfigurationTest`.

Most bridges are NOT covered by network tests (they would hit live sites), so the
structural tests plus manual CLI invocation are the practical verification path.

## Configuration

Runtime config is read from `config.ini.php` if present, otherwise
`config.default.ini.php`. Access values via `Configuration::getConfig('section', 'key')`.
Key sections: `[system] env` (`dev` turns warnings into thrown exceptions and enables
debug logging), `[cache] type`, `[authentication]`, `[error] output`/`report_limit`,
`[proxy]`. Bridges are gated by `enabled_bridges[]` (`*` enables all).

## License

Public domain (UNLICENSE). Bundled third-party libs under `lib/` (Parsedown,
simplehtmldom, php-urljoin) keep their own MIT licenses — don't relicense them.
