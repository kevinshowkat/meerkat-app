# Read-time privacy enforcement

*Shipping a retroactive privacy toggle with no migration, no backfill — and the canary that watches whether visibility rules bend the data.*

## The feature

Meerkat's rating-visibility rules came from the product side; I built the enforcement. They boil down to: participation is public, exact scores are discoverable but never pushed to the rated author, and there's one escape hatch — **"Rate privately."** Flip it and your avatar leaves rater stacks and your rating history goes private, *including everything you rated before flipping it*. Your scores still count toward every aggregate.

"Retroactive" is the word that usually turns a toggle like this into a project: scrub jobs, backfills, cache invalidation, and a week of "why is my avatar still showing" bugs.

## The implementation

The whole feature is **read-time**. Nothing about historical data changes when the toggle flips:

- The setting rides an existing enum column on accounts — the value was defined but enforced nowhere, so shipping it required **zero migrations**.
- **Rater stacks:** the preview builder excludes private raters from the avatar array; the total count still includes them. Aggregates are never reduced by privacy.
- **Rating history:** the profile-reactions query returns an empty list to any viewer who isn't the owner. Same response envelope, so clients need no special casing.
- **The no-delivery invariant:** the notifications pipeline must never pair a rater's identity with their exact score for the author. That's enforced by audit plus a test that asserts the invariant, so a future notification type can't quietly break it.

Because enforcement lives in the read path, the toggle is instant, retroactive by construction, and impossible to get out of sync — there's no denormalized "privacy state" to reconcile.

Every filter lands twice: the backend runs a **dual store** (Postgres and a behavior-identical in-memory implementation), so the full ~11k-line API suite exercises this logic with zero infrastructure — no containers, no test database, sub-second startup. The Postgres store is then covered by the same contract tests against real SQL.

## The canary

Visibility rules shape behavior, so the system measures whether they're working: a weekly report over the trailing eight weeks of *given* scores — mean, median, spread, band distribution — excluding seed and content accounts, on both the raw internal scale and the user-facing ten-point scale.

It exists to answer one question: **are people going soft?** If mean given-score drifts up more than +0.5 (ten-point scale) after a visibility feature ships, exact scores demote to score bands on public surfaces. The measurement shipped with the features; the demotion is a trigger waiting on evidence.

The guardrail that matters most: below a minimum distinct-rater count, the report marks itself directional-only and the trigger is suspended. A handful of enthusiastic early raters can move a mean a full point; small samples don't get to make policy. The same small-N rules apply in-product — sparse aggregates render differently, so a "10 from one rating" never masquerades as consensus.

## The takeaway

Privacy features want to be read-time. If you can afford the query-path cost, you get instant, retroactive, always-consistent behavior for free — and the test story collapses from "verify the scrub job ran everywhere" to "assert one filter in one place," twice, against both stores.
