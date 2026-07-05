# The coin at 120Hz

*Cartoon drag physics on the rating dial without dropping a ProMotion frame.*

## The brief

The spec's governing principle for the rating gesture: **light, fun, not quantitative** — flicking an opinion at a wall, not filling in a survey. This note is about what it costs to build that at 120Hz.

## The thumb becomes the coin

The rating dock is a gradient rail — salmon through gold to green — and the drag thumb *is* the score coin, the same object that will land on the post when you commit. That continuity is the trick: you're not moving an abstract handle that produces a number somewhere else; you're holding the verdict in your thumb.

While you drag:

- **Rubber-hose physics.** The coin is gesture-coupled with a hint of cartoon lag and stretch — enough personality to feel alive, never so much that it feels detached from your finger. (The name is a nod to 1930s rubber-hose animation, which is the app's whole visual mood.)
- **Mood preview.** The coin's face previews the band you're hovering over — queasy at the bottom, side-eye through the contested middle, heart-eyes up top — so the emotional meaning of the number reads before the number does.

## The commit beat

Release is a three-beat animation: **anticipation → spring → land.** A brief gather, an overshoot spring, and the coin sets onto the card with its final score. The beat structure comes from classical animation, and it does real product work — rate-to-reveal means the crowd's score appears only after you commit, so the commit needs to feel like *a moment*: you spent your coin, now the jury comes back. Undo exists, but as a deliberate follow-up action, never a mid-animation take-back — the beat resolves, then you can change your mind.

## Paying for it at 120Hz

All of this runs inside a 120Hz interaction budget on ProMotion devices, which is where whimsy gets expensive:

- The gradient rail, glass-material dock, and animated coin must never contend with feed scrolling. The feed's snapshot pipeline (see [shimmer-free scrolling](shimmer-free-scrolling.md)) exists partly so the live rendering budget is available where it matters — under your thumb.
- Glass materials smear when they're re-rendered mid-transform; containers that hold animating content get their material phases pinned so the blur never crawls during the spring.
- Masked text (the "?" on an unrated coin) turned out to be its own rendering trap — pre-composited rather than live-masked, after it produced one-frame artifacts during handoffs.

The lesson that generalizes: juice is a performance feature. An interaction can survive being plain; it cannot survive stuttering. Every animation earned its frame cost or it shipped simpler.
