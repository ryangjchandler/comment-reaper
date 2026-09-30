# Language-specific notes

These notes add to the rules in `SKILL.md` for each ecosystem. Wherever they conflict, the "Keep these untouched" rules in `SKILL.md` win.

## PHP / Laravel

- **Framework stub docblocks are fair game.** Remove "Create a new controller instance.", "Run the migrations.", "Reverse the migrations.", "Handle an incoming request.", "Define the model's default state.", "Get the attributes that should be cast.", "Bootstrap any application services.", "Register any application services.", "The attributes that are mass assignable." and similar summaries when all they do is name the method or property.
- **Keep the generic type tag and delete only the prose** on Laravel properties and relations, because Larastan relies on the generic:
  ```php
  // Before
  /**
   * The attributes that are mass assignable.
   *
   * @var list<string>
   */
  protected $fillable = ['name', 'email'];

  // After
  /** @var list<string> */
  protected $fillable = ['name', 'email'];
  ```
  The same applies to `@return BelongsTo<User, $this>`, `@return HasMany<Post, $this>`, `@return array<string, string>` on `casts()`, `@use HasFactory<UserFactory>`, and `@extends Factory<User>`. When a project has no PHPStan/Larastan config at all, you may drop `@var`/`@return` tags on these too, but only if the native type already says the same thing (bare `array` isn't the same as `list<string>`, so keep the tag).
- **Keep all of these:** `@property`, `@property-read`, `@method`, and `@mixin` on models, facades and macroable classes. IDEs and static analysis need them.
- **Laravel's "padded" tag style** (`@param  string  $name`). If you keep a tag, keep the spacing style the file uses.
- **Keep `@throws`** on methods that throw domain exceptions callers are expected to catch. Remove it for exceptions that only bubble up from framework internals.
- **Pest/PHPUnit:** keep `@test`, `@dataProvider`, `@group`, `@covers`, `@depends` and similar annotations. Test names are already descriptive, so a comment that repeats `it('creates a user')` goes.
- **Blade:** `{{-- --}}` comments follow the same rules. HTML comments `<!-- -->` ship to the browser, so removing noisy ones is a small win. Keep conditional comments and anything a JS library reads.
- **Config files** (`config/*.php`): the framework's long explanatory blocks at the top of each config section explain options for people who'll edit them later. **Trim** them to one line each instead of deleting them, unless the option name says it all on its own.

## JavaScript / TypeScript

- In `.ts`/`.tsx`, JSDoc `{type}` annotations are always redundant. Remove `@param {string} name` and `@returns {Promise<void>}`. Keep `@param name` descriptions only if they add meaning, `@deprecated`, `@example` on public API, `@internal` (used by `stripInternal`), and `@see`.
- In `.js` files of a project with `checkJs`, or with no TypeScript at all while an editor or `tsc` reads JSDoc for types, **JSDoc types are the type system.** Keep `@type`, `@param {…}`, `@returns {…}`, `@typedef`, `@template`, and `@import` typedefs. Only delete prose that restates them.
- Keep `/*#__PURE__*/`, `/* webpackChunkName */`, `/* @vite-ignore */`, `// @ts-*`, `// eslint-*`, `// prettier-ignore`, `/* istanbul ignore */`, `/* c8 ignore */`, and `/// <reference>`.
- Vue/Svelte SFC templates: `<!-- -->` follows the same rules as HTML above.

## Python

- Docstrings are runtime-visible (`__doc__`, `help()`, Sphinx, and CLI frameworks like Click/Typer that turn them into `--help` text). Before you delete one, check whether a CLI decorator or `argparse` reads it. If it does, trim it.
- Keep doctest examples (`>>>` blocks), since they're tests.
- Remove `Args:` / `Returns:` / `:param x:` / `:type x:` sections that only restate type hints. Keep argument descriptions that add units, allowed values or side effects.
- Keep `# type: ignore[...]`, `# noqa`, `# pragma: no cover`, `# fmt: off/on`, `# isort: skip`, `# pyright: ...`, and `# mypy: ...`.
- Keep a module docstring on a package's `__init__.py` or on a script entry point if it describes usage.

## Go

- Linters (revive/golint, and `godoc` conventions) expect a doc comment on exported identifiers, starting with the identifier's name. **Trim** these, don't delete them: `// Client talks to the billing API.` is enough.
- Delete doc comments on unexported identifiers that only repeat the name.
- Keep `//go:build`, `//go:generate`, `//go:embed`, `//go:linkname`, `//nolint`, `//export` (cgo), and the cgo preamble comment immediately above `import "C"`.

## Rust

- `///` and `//!` on public items feed rustdoc, and code blocks inside them are doctests. Trim the prose and keep the examples.
- `#![warn(missing_docs)]` / `#![deny(missing_docs)]` means public items need *some* doc. Trim, don't delete.
- Delete `//` narration freely. Keep `// SAFETY:` comments on `unsafe` blocks, because clippy's `undocumented_unsafe_blocks` looks for them and they carry real reasoning. Trim them if they're long.

## Ruby

- Keep `# frozen_string_literal: true`, `# encoding:`, `# rubocop:disable/enable`, `# :nodoc:`, and Sorbet sigils (`# typed: strict`).
- YARD `@param [String] name` tags follow the same rule as PHPDoc. In Sorbet-typed files the `sig` already says it, so remove them.

## C# / Java / Kotlin

- XML doc comments (`///`) and Javadoc/KDoc on public API in libraries: trim to `<summary>` plus meaningful `<exception>` / `@throws` tags. Remove `<param>` / `@param` entries that just repeat the parameter name.
- Keep `#pragma`, `// ReSharper disable`, `@SuppressWarnings` explanations, `//noinspection`, and `// NOSONAR`.

## Shell / SQL / CSS / other

- Shell: keep the shebang, `# shellcheck disable=`, and usage headers on scripts people run by hand.
- SQL migrations: keep comments explaining data backfills or irreversible steps. Delete `-- create users table` above `CREATE TABLE users`.
- CSS/SCSS: keep `/*! */` license banners, `/* stylelint-disable */`, `/* purgecss ignore */`, and comments explaining browser hacks (especially ones with a bug link). Delete section banners.
