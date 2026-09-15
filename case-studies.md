---
title: Case Studies
description: Short application-specific notes that apply system-design components and patterns to real products.
---

# Case Studies

These compact notes show how system-design choices change with the application.
They are not complete architectures; use them to remember an important
bottleneck, trade-off, or hybrid solution during an interview.

## News Feed

A news feed collects posts from accounts a user follows and orders them for
display. The central trade-off is whether to build each user's feed when posts
are created or when the feed is requested.

```{note} Fan-out on write vs. fan-out on read
**Fan-out on write (push):** When an ordinary account posts, background workers
copy the post ID into each follower's precomputed feed. Reading is fast, but a
popular account can turn one post into millions of writes.

**Fan-out on read (pull):** When a user opens the app, fetch recent posts from
the accounts they follow and merge or rank them. This avoids write amplification
but makes reads expensive, especially when the user follows many accounts.

**Hybrid:** Fan out writes from accounts with manageable audiences, but do not
copy posts from celebrity or other high-fan-out accounts into every follower's
feed. At read time, merge those high-fan-out posts into the precomputed feed.
Choose the cutoff using follower count, posting frequency, and active followers—not
just a celebrity label.
```

The hybrid design spends background writes to keep ordinary reads fast while
preventing a single popular post from creating an enormous write spike. It
requires the read path to merge, rank, deduplicate, and paginate results from
both sources.
