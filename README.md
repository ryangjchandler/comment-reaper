# Comment Reaper

An agent skill that harvests code comments. It deletes comments and docblocks that restate what the code already says, trims useful comments down to the minimum context, and leaves the code itself untouched.

## Install

```bash
npx skills add ryangjchandler/comment-reaper
```

## Usage

Ask your agent to harvest or reap comments, or invoke the skill directly with an optional target:

```
/comment-reaper                      # whole repository
/comment-reaper app/Models/User.php  # a single file
/comment-reaper src/Http             # a directory
/comment-reaper billing              # a subsystem, resolved automatically
```

The skill runs on its own from start to finish. It doesn't commit anything. Review the result with `git diff`.

## Example

Before:

```php
/**
 * The attributes that are mass assignable.
 *
 * @var list<string>
 */
protected $fillable = ['user_id', 'total_cents'];

/**
 * This method is responsible for calculating the total of the order.
 * We loop through all line items and add them. Note that we must round
 * each line individually because Stripe rounds per line and totals must
 * match to the cent. See https://github.com/acme/shop/issues/412.
 *
 * @param  array  $lines  The lines
 * @return int  The total
 */
public function total(array $lines): int
{
    // initialise the total
    $total = 0;

    // loop through the lines
    foreach ($lines as $line) {
        // $total += $line['price'];
        $total += (int) round($line['price'] * $line['qty']); // @phpstan-ignore-line
    }

    // return the total
    return $total;
}
```

After:

```php
/** @var list<string> */
protected $fillable = ['user_id', 'total_cents'];

/**
 * Rounds each line individually because Stripe rounds per line and totals must match to the cent.
 *
 * @see https://github.com/acme/shop/issues/412
 */
public function total(array $lines): int
{
    $total = 0;

    foreach ($lines as $line) {
        $total += (int) round($line['price'] * $line['qty']); // @phpstan-ignore-line
    }

    return $total;
}
```

The generic `@var` tag, the reason for rounding per line, the issue link and the PHPStan directive all survive. Everything that repeated the code is gone.

## What it removes

- Comments that narrate the code (`// loop through users`)
- Docblocks that only repeat the signature (`@param string $name The name`, `@return void`)
- Framework boilerplate summaries ("Create a new controller instance.")
- `@author`/`@since`/`@package` metadata, commented-out code, and banner dividers

## What it keeps

- Comments explaining *why*: workarounds, invariants, trade-offs, issue links (trimmed to the essentials)
- Type information richer than the native signature (generics, array shapes, `@template`, `@property`/`@method`)
- Tool directives and runtime annotations (`@phpstan-ignore`, `eslint-disable`, `# noqa`, `//go:build`, Doctrine attributes-in-comments, test annotations)
- License headers, and `@deprecated`/`@throws`/`@internal`/`@see`

## License

MIT
