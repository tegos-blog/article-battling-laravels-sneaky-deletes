Hey folks, if you've ever stared at your terminal in horror as Laravel cheerfully deletes half a million rows instead of the tidy batch of 500 you asked for, you're not alone. That's exactly what kicked me into gear a few weeks back. As a Laravel enthusiast, I dove into the framework's MySQL grammar to sort out this quirky behavior. Spoiler: it ended in a merged PR and a few lessons in database drama. Let's break it down.

### The Sneaky Problem

Picture this: You're cleaning up a massive `cross_product_references` table by joining it with `products` and nuking invalid entries in chunks. Your code looks solid-LIMIT(500), ORDER BY `id`, and a tidy `while` loop to keep things batched. But nope. Laravel's MySQL grammar? It _silently_ strips the ORDER BY and LIMIT when JOINs are in the mix. Boom: one query, all 500k+ rows gone. Poof.

Here's the culprit in action (straight from my console command):

```php
$batchSize = 500;
$totalDeleted = 0;

do {
    $deleted = DB::table('cross_product_references as cpr')
        ->leftJoin('products as p1', fn($join) => $join
            ->on('cpr.article', '=', 'p1.article')
            ->on('cpr.brand_id', '=', 'p1.brand_id'))
        ->leftJoin('products as p2', fn($join) => $join
            ->on('cpr.cross_article', '=', 'p2.article')
            ->on('cpr.cross_brand_id', '=', 'p2.brand_id'))
        ->where(fn($query) => $query
            ->whereNull('p1.id')
            ->orWhereNull('p2.id'))
        ->limit($batchSize)
        ->orderBy('cpr.id')
        ->delete();
    $totalDeleted += $deleted;
} while ($deleted > 0);

$this->line("Deleted <info>$totalDeleted</info> rows.");
```

Expected: Batches of 500, safe and sound. Actual: one massive, spectacular (and slightly horrifying) purge.
Turns out, this "feature" was there to dodge MySQL's native gripes about ORDER BY/LIMIT in DELETE...JOIN queries. But silent failures? Not cool.

### The Fix (and a Side of Debate)

I fired up a PR [#57138](https://github.com/laravel/framework/pull/57138) overriding `compileDeleteWithJoins` in `MySqlGrammar`. Now? Laravel spits out the _full_ SQL-JOINs, ORDER BY, LIMIT and all. Let the database (MySQL, MariaDB, or even Vitess) decide if it's happy. On vanilla MySQL? You'll get a crisp QueryException. Check mysql docs for more info ([Mysql: Multi-Table Deletes](https://dev.mysql.com/doc/refman/8.4/en/delete.html#id246705)). On PlanetScale or Vitess? It just works, thanks to their query-rewriting magic.

But oh, the comments! Taylor Otwell and Graham Campbell chimed in, and we had a mini philosophy sesh: Throw a framework exception for safety, or let MySQL yell? I pushed for explicit errors to avoid "surprise deletions," but we landed on compiling the full query anyway-best of both worlds. A quick branch swap to target `master`, some test tweaks, and... PR [#57196](https://github.com/laravel/framework/pull/57196) was born. Fast-forward a few days, and yeah, it got merged. 🎉

So, why does this actually matter? Predictability, my friend. You don't want your maintenance scripts nuking the database behind your back. If you're batch-deleting with joins, jump on the latest Laravel and double-check those LIMITs. Old-school MySQL folks, here's a hack: snag the IDs with FOR UPDATE, then do your DELETE WHERE IN. Works like a charm.

## TL;DR

Laravel now respects `ORDER BY` and `LIMIT` in `DELETE ... JOIN` queries for MySQL. If the DB doesn't support it - you'll get a proper exception instead of a silent data wipe.

💡 PRs in question: [#57138](https://github.com/laravel/framework/pull/57138) (draft) and [#57196](https://github.com/laravel/framework/pull/57196)

🚀 **Merged into:** Laravel 13.x

Grateful for the Laravel crew's sharp eyes-it was a reminder that open-source magic happens in those comment threads. Got a similar gotcha? Hit reply or ping me [@tegos](https://www.linkedin.com/in/ivan-mykhavko/). What's your wildest query story?