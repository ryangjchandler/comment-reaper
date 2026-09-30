---
name: comment-reaper
description: Harvests code comments across a file, directory, subsystem or whole codebase. Deletes comments and docblocks that restate what the code or its type signatures already say, trims comments that are worth keeping down to the minimum context, and leaves the code itself untouched. Use when the user asks to clean up, trim, prune, reap or harvest comments, remove redundant or noisy docblocks (PHPDoc, JSDoc, docstrings), or reduce comment bloat. Accepts an optional target path, glob or subsystem name. With no target it runs on the whole repository.
argument-hint: "[file | directory | glob | subsystem]"
---

# Comment Reaper

Go through the target code and **harvest** its comments. Every comment gets one of three verdicts:

- **Delete**: it tells the reader nothing the code, names or types don't already tell them.
- **Trim**: it holds real information but uses more words than it needs. Cut it down to the smallest wording that keeps that information.
- **Keep**: it's already tight and useful. Leave it alone.

Run on your own from start to finish. Don't stop to ask for confirmation, don't show a plan and wait for approval, and don't ask about individual comments. Git is the undo button. The only input is the target. When a case is unclear, follow the tie-breaker rules below and keep going.

## The one hard rule

**Change comments only. Never change code.** That covers identifiers, formatting outside the comment itself, imports and logic. Also leave alone comments that the tooling reads, because those count as code (see "Comments that are really code"). If a comment makes you think the code is wrong, don't fix the code. Keep the comment and flag it in the report.

## Step 1: Resolve the target

The argument can be:

| Input | Resolution |
| --- | --- |
| Nothing | The whole repository. |
| A file path | That file. |
| A directory or glob | Every source file matched. |
| A subsystem name (e.g. `billing`, `auth`, `queue workers`) | Find it yourself. Search directory names, namespaces and packages, route and config files, and class name prefixes. Take the union of the obvious matches (e.g. `app/Billing/**`, `app/Http/Controllers/Billing/**`, `tests/**/Billing/**`). Mention the resolved paths in the final report. Don't ask the user to confirm them. |

List candidate files with `git ls-files` (plus untracked, non-ignored files if the target is untracked) so that `.gitignore` is respected. **Always skip:**

- Dependency and build output: `vendor/`, `node_modules/`, `dist/`, `build/`, `public/build/`, `target/`, `.next/`, caches, compiled views.
- Generated files: anything with a header like `@generated`, `DO NOT EDIT`, `auto-generated`, or `Code generated ... DO NOT EDIT`, plus IDE helper files (`_ide_helper*.php`, `*.d.ts` emitted by tools).
- Minified files, lock files, snapshots, and fixture or test-data files where comments may be part of the data under test.
- Non-source files (Markdown, JSON, YAML, env files) unless the user named that file explicitly.

If the target is empty after filtering, say so and stop.

## Step 2: Learn the project's conventions (once, before editing)

Spend a minute on this. It decides how aggressive you can be.

1. **Static analysis that reads docblocks.** Look for PHPStan/Larastan (`phpstan.neon*`), Psalm (`psalm.xml`), TypeScript `checkJs`/`allowJs` with JSDoc types, mypy/pyright, and similar tools. If any are present, docblock type information that goes *beyond* the native signature is load-bearing and must stay (see the type rules below).
2. **Lint rules that require comments.** Look for phpcs sniffs such as `Squiz.Commenting.*` or `SlevomatCodingStandard.Commenting.*`, eslint `jsdoc/require-*`, pydocstyle/ruff `D` rules, golint/revive `exported`, and clippy `missing_docs`. If a rule requires a docblock somewhere, **trim** that docblock to the minimum the rule accepts instead of deleting it.
3. **Library or application?** A package with a public API (a published Composer/npm/PyPI/crates package, or generated API docs through phpDocumentor, TypeDoc, Sphinx, rustdoc or godoc) should keep short docs on its **public** surface. Internal and private code gets the full treatment either way.
4. **The house style of surviving comments.** Match it: sentence case or not, trailing periods or not, `//` or `#`. Keep your trimmed comments consistent with it.

If static analysis is configured and fast enough, run it on the target now and save the output as a baseline for Step 4.

Don't write any of this down for the user. Just apply it.

## Step 3: Harvest file by file

For each file: **read the whole file first**, then judge each comment in the context of the code around it, then make the edits. Never judge a comment from a grep match alone.

For large targets (roughly more than 40 files), work through them in batches by directory. If your environment supports parallel subagents, give each one a directory and a copy of these rules, then combine their reports.

### Delete these

- **Narration of the code.** `// loop through users`, `// return the result`, `// increment counter`, `# check if valid`, `// call the API`.
- **Docblocks that only repeat the signature.**
  ```php
  /**
   * Get the user's name.
   *
   * @param  int  $id  The id
   * @return string
   */
  public function getUserName(int $id): string
  ```
  Delete the whole block. The name, parameter types and return type already say all of it.
- **Tags that duplicate native types**: `@param string $name` when the parameter is `string $name`, `@return void`, `@return bool` on a `: bool` method, `@var int` on a typed `int` property, and JSDoc `@param {string}` in TypeScript files.
- **Boilerplate summaries**: "Create a new instance.", "Constructor.", "Class Foo", "Interface for bar", "Handle the request.", "Run the migrations.", "Getter for x", "The name of the thing" above `$name`. Delete these even when a framework's generated stubs ship with them.
- **Metadata that version control already tracks**: `@author`, `@since` / `@version` in application code, `@package` / `@subpackage` that just repeat the namespace, "Created by X on DATE", and change logs kept in comments.
- **Commented-out code.** Git has it.
- **Decorative noise**: banner dividers (`// ===== Helpers =====`), region markers with no tooling behind them, empty `/** */` blocks, and lone `//`.
- **Section labels that name the next line**: `// Relationships` above one relation method, `// Constructor` above the constructor.
- **Comments that are clearly stale** and describe behaviour the code no longer has, *when* the code is plainly correct and the comment is simply outdated. (If you can't tell which one is wrong, keep it and flag it.)

### Trim these to the minimum

Keep the *information*, cut the words. A good trimmed comment is usually one line and explains **why**, not what.

- **Reasons behind non-obvious decisions**: workarounds, ordering constraints, performance or security trade-offs, "don't change this because…".
- **Invariants and contracts the types can't express**: units ("milliseconds"), ranges, "must be called inside a transaction", thread-safety or idempotency, side effects, what `null` means.
- **Links to issues, tickets, RFCs, specs and upstream bugs.** Always keep the link, even if you cut everything else.
- **Explanations of regexes, bit twiddling, magic numbers and dense algorithms.**
- **`TODO` / `FIXME` / `HACK` / `XXX`** that carry actionable context. Trim the prose but keep the marker and any owner or ticket reference. Delete a bare `// TODO` with no content.
- **Short public-API docs** in libraries (Step 2.3), and docblocks that a lint rule requires (Step 2.2). Cut them to a single summary line plus only the tags that add information.

Ways to trim:
- Drop throat-clearing: "This method…", "Here we…", "Basically", "Note that", "Please be aware", "In order to".
- Drop the half that restates the code and keep the half that explains it.
- Turn several paragraphs into one sentence, and a sentence into a fragment when that's still clear.
- Don't reword a comment that's already tight just to make it yours.

```php
// Before
/**
 * This method is responsible for calculating the total price of the order.
 * We need to loop through all of the line items and add them up. Note that
 * we have to round each line individually before summing because the
 * payment provider rounds per line and totals must match to the cent.
 * See https://github.com/acme/shop/issues/412 for more details.
 *
 * @param  Order  $order  The order
 * @return int  The total
 */
public function total(Order $order): int

// After
/**
 * Rounds per line before summing to match the payment provider's totals.
 *
 * @see https://github.com/acme/shop/issues/412
 */
public function total(Order $order): int
```

### Keep these untouched

- **Comments that are really code.** Tooling reads them, so removing one changes behaviour:
  - Suppressions and directives: `@phpstan-ignore*`, `@psalm-*`, `phpcs:ignore`, `@codeCoverageIgnore`, `eslint-disable*`, `prettier-ignore`, `@ts-expect-error`, `@ts-ignore`, `@ts-nocheck`, `/// <reference>`, `# noqa`, `# type:`, `# pragma`, `# fmt: off`, `//go:build`, `//go:generate`, `//nolint`, `#[allow]`-style comment directives, `istanbul ignore`, `c8 ignore`, `@vite-ignore`, webpack magic comments, `@license` / `@preserve` / `/*!` banners.
  - Runtime annotations: Doctrine `@ORM\…`, Symfony `@Route`, JMS/Serializer, and anything else parsed through reflection.
  - Test metadata: `@test`, `@dataProvider`, `@depends`, `@group`, `@covers`, and doctest examples in Python/Rust docs.
  - Shebangs, encoding declarations, and `declare`-style headers.
- **Type information richer than the native signature.** Generics and collections (`array<string, int>`, `list<User>`, `Collection<int, Post>`, `BelongsTo<User, $this>`), array shapes, `class-string<T>`, literal and union types the language can't express natively, `@template`, `@extends` / `@implements` generics, `@property` / `@method` / `@mixin` on magic classes such as Eloquent models and facades, and `@var` on untyped properties you can't type natively. **Rule:** remove a type tag only when the native type says exactly the same thing. When you strip a docblock down, keep these tags and drop the rest. If no prose is left, a tags-only docblock is fine.
- **`@throws`** when it documents a meaningful exception the caller should handle, **`@deprecated`** (keep the replacement hint), **`@internal`**, **`@api`**, and **`@see`** pointing at something useful.
- **License and copyright headers.**
- **Comments that contradict the code** where you can't tell which one is right. Keep them unchanged and flag them.

### Tie-breakers

- If deleting a comment would lose any fact a new maintainer couldn't recover from the code in under a minute, **trim**, don't delete.
- If the choice is between trimming and keeping, prefer the shorter version. Trimming can reword, but it may only use facts the original comment already states. Never add or guess at a reason. If a shorter version would need a fact you'd have to infer, leave the original.
- If a comment is used by tooling, or you're not sure whether it is, **keep** it. That includes directives for tools the project doesn't seem to configure.
- Leave file-level docs in entry points (a package's main module, a CLI's usage header) alone unless they're pure boilerplate.

See [references/languages.md](references/languages.md) for language-specific notes on PHP/Laravel, JS/TS, Python, Go, Rust, Ruby and others.

### Tidy up after each edit

- Remove blank lines left behind where a comment was, so you don't leave double blank lines or a blank line right after an opening brace.
- Collapse a docblock with a single remaining line to one line (e.g. `/** @var list<string> */`). Don't do this if the project clearly always uses multi-line docblocks, for example because a lint rule enforces them.
- Keep indentation and line endings exactly as they were.

## Step 4: Verify

1. **Diff audit.** Run `git diff -U0 -- <touched files>` and check that every removed or added line is a comment or a blank line. If any code line changed, revert that part.
2. **Syntax check** each touched file with the cheapest available tool: `php -l`, `node --check` / `tsc --noEmit` on the project, `python -m py_compile`, `gofmt -l`, `ruby -c`, or `cargo check`.
3. **Static analysis**, if the project has it and it runs in reasonable time on the target paths (PHPStan/Larastan, Psalm, `tsc`, mypy/pyright). Run it once on the target *before* you edit anything to get a baseline, then again afterwards. Any new error means a type tag you removed was load-bearing. Restore that tag and re-run.
4. **Formatter / linter** in check mode on touched files only (e.g. `vendor/bin/pint --test <files>`, `npx prettier --check <files>`, `ruff check <files>`). Fix only the problems your edits introduced. Don't reformat anything else.

Skip any tool that isn't installed or configured. Don't install anything.

## Step 5: Report

Don't commit. Leave the changes in the working tree for the user to review with `git diff`. Finish with a short report:

```
Comment Reaper: <target> (<resolved paths if a subsystem>)

Files scanned: N   Files changed: M
Comments deleted: D   Trimmed: T   Kept: K

Notable:
- app/Billing/Invoice.php: collapsed 40-line class docblock to 1 line
- …up to ~10 of the most significant changes

Flagged for human review:
- app/Queue/Retry.php:88: comment says "max 3 attempts", code allows 5
- …stale or contradictory comments you kept, and anything you were unsure about

Verification: php -l ✓  phpstan ✓ (no new errors)  pint --test ✓
```

When counting, one docblock or one contiguous run of `//`/`#` lines counts as one comment. Leave out the "Flagged" section if nothing was flagged. Keep the report short, because the diff is the real output.
