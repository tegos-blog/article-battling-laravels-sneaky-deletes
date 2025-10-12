# Battling Laravel's Sneaky DELETEs: How I Got ORDER BY and LIMIT to Play Nice with Joins

![Battling Laravel's Sneaky DELETEs](assets/poster.jpg)

A casual dive into fixing Laravel's MySQL grammar to support full DELETE queries with JOINs, ORDER BY, and LIMIT-preventing accidental data wipes during batch operations.

## What's Inside
- The problem: Silent stripping of LIMIT/ORDER BY in JOIN deletes.
- The fix: PRs [#57138](https://github.com/laravel/framework/pull/57138) (draft) and [#57196](https://github.com/laravel/framework/pull/57196) (merged).
- Lessons from the review: Debates on exceptions vs. letting MySQL complain.
- Pro tip: Use `FOR UPDATE` + `WHERE IN` for MySQL workarounds.

Read the full post: [Battling Laravel's Sneaky DELETEs](https://dev.to/tegos/battling-laravels-sneaky-deletes-how-i-got-order-by-and-limit-to-play-nice-with-joins-ng9).