# Shimmer-free scrolling

*How Meerkat swaps live SwiftUI cards for baked bitmaps mid-scroll without a visible pixel of difference.*

## The setup

Meerkat's home feed is expensive to render live: every card carries an illustrated surface, a gradient rating rail, an animated score coin, avatars, and glass-material chrome. To hold a 120Hz scroll, cards render **live at rest** and swap to a **pre-rasterized snapshot** while the feed is moving. Classic technique; the pipeline maintains a snapshot cache with budgets scaled to the device's render scale.

The problem is the seam. Users kept reporting the feed felt "unstable" — and they were right. At swap boundaries you could intermittently catch a one-frame artifact: text that changed size almost imperceptibly, a card edge moving ~1px, an avatar flashing from photo to monogram, media blinking through a placeholder gradient. Each defect was tiny. Together they read as shimmer, which is the exact opposite of what a snapshot pipeline is for.

The bar we set (verbatim from the spec): the swap must be **pixel-invisible**. Not "subtle." Not "brief." Invisible.

## Eyeballing is not evidence

The first lesson was that casual observation misdiagnoses this class of bug. We recorded scroll sessions and stepped them frame by frame. Most "flashes" people reported were actually ordinary 1px scroll steps — the whole screen translating uniformly, no swap involved. And the two scroll-settle swaps in the first recording were *clean*; the defect was intermittent, dependent on card state, not present at every swap.

That reframed the work: a fix couldn't be verified by scrolling and squinting. It had to be verified across the **state space** of cards — media loaded vs. loading, avatar resolved vs. fallback, coin score present vs. lagging, glass chrome settled vs. animating — because any single scroll only samples a few states.

## The failure taxonomy

Frame forensics plus a source audit produced a ranked root-cause inventory. The recurring shapes:

1. **Size-mismatch bakes.** A snapshot baked at a stale or estimated size, then drawn at the card's true layout size — guaranteed subpixel text and edge drift. Fix: bake at the exact final pixel size or don't swap.
2. **Incomplete cache signatures.** The snapshot key didn't include every input that changes pixels (avatar resolution state, score availability, material phase). Two different visual states shared one cache entry, so the "same" snapshot was sometimes wrong. Fix: the signature covers *everything* that draws.
3. **Swapping before ready.** If any dependency (image, score, avatar) hadn't resolved when the bake happened, the snapshot froze the placeholder — and the swap *revealed* it. Fix: readiness gates, and re-minting the snapshot when a dependency lands.
4. **Projection drift.** Cards positioned against map-projected geometry could disagree with the snapshot's captured offset by a pixel. Fix: reconcile both paths to one source of truth for placement.

Plus a rule that closed the loop at rest: once settled, a card that has a valid snapshot **stays on the snapshot** until the next live interaction requires it — no gratuitous swap back to live at settle, because every swap is another chance to be visibly wrong.

## Where it landed

Swaps are bit-identical for a card in a given state; the remaining motion at settle is the score coin's own animation, which is supposed to move. The durable takeaways:

- "Fast but flickery" is a worse trade than it looks; users experience micro-inconsistency as *the product being broken*, even when they can't name it.
- Perceptual bugs need forensic verification. Record, step frames, classify — then fix categories, not sightings.
- Cache keys for rendered UI are contracts. If a pixel input isn't in the key, it will eventually be wrong — and it'll be wrong mid-scroll, on someone else's device.
