# hxt-shorts top-up — run brief

You keep the Hit x Trial Buffer queue topped up from `queue.json` until every video in it has
been posted once to Instagram (Reel) and once to YouTube (Short). You never make or edit
videos. You only create Buffer posts and update `ledger.json`.

**The one rule above all others: never post a video twice to the same channel.** A video
counts as already posted to a channel when EITHER `ledger.json` has a row for that slug and
channel whose status is not `error`, OR Buffer has a post on that channel whose video source
URL equals the queue's `videoUrl` (or the row's old URL in the ledger) with status
`scheduled`, `sending` or `sent`. When in doubt, do not post and say why in the final message.

## Fixed facts
- Buffer organization `6aa7c7f3ac8fd4ea0a97166a`. Instagram **hitxtrial**
  `6aa7e62eea19ca0bde3e0a29`; YouTube **Hit x Trial** `6aa7e507ea19ca0bde3e03a8`.
- Buffer Free plan: **10 scheduled posts per channel**. The hxt-lessons routine also posts on
  these two channels (09:00 and 18:00 IST), so this routine keeps **at most 8 scheduled posts per
  channel** (count every scheduled post on the channel, not only ours).
- Buffer tools are found with `ToolSearch` (query "buffer"); they may be named
  `mcp__Buffer__*`. Use `execute_query` for reads and `create_post` for writes.
- Times: IST is UTC+05:30. **One video in every clock hour, round the clock (24 a day), at a
  random minute.** Pick the minute with `python3 -c "import random; print(random.randint(5, 55))"`;
  in the 09:00 and 18:00 IST hours (when the hxt-lessons routine posts) use
  `random.randint(20, 55)`. Instagram and YouTube for the same video share the same time.
  Never put two queue videos in the same clock hour.

## Steps
1. `git pull --rebase`. Read `queue.json` (ordered list; each item has `slug`, `videoUrl`,
   `coverMs`, `ytTitle`, `ytText`, `igText`) and `ledger.json`.
2. Read Buffer with `execute_query`, paginating with `after` until `hasNextPage` is false:
   ```graphql
   query P($input: PostsInput!, $after: String) { posts(first: 100, after: $after, input: $input) {
     pageInfo { hasNextPage endCursor }
     edges { node { id status channelId dueAt sentAt externalLink error { message } assets { source } } } } }
   ```
   with `input = {organizationId, filter: {channelIds: [both ids], createdAt: {start: "2026-09-24T21:00:00Z"}}}`.
3. Reconcile the ledger: for every Buffer post whose source URL belongs to a queue item or a
   ledger row, set that row's `status`, `dueAt`, `sentAt`, `link` (externalLink) and `error`
   from Buffer; add a row if Buffer has a post the ledger lacks (never drop a row).
4. Capacity: for each channel, `free = 8 - (number of Buffer posts on that channel with
   status scheduled)`. If both are 0, skip to step 7.
5. Next slot: take the latest `dueAt` among ledger rows whose status is `scheduled`,
   `sending` or `sent`; the next video goes in the clock hour after that one, at a random
   minute. If that time is earlier than now + 20 minutes, use the clock hour that contains
   now + 20 minutes instead (random minute no earlier than now + 20 minutes; if that is not
   possible, the following hour). Each further video in this run takes the next clock hour.
6. Walk `queue.json` in order. For each item, find the channels it is still missing (per the
   rule above; a row with status `error` counts as missing, but only once — if the slug and
   channel already has two `error` rows, skip it and report it). For each missing channel with
   `free > 0`, create the post at the item's slot, both channels at the same slot when both
   are missing. Stop when either channel's `free` reaches 0 for a pair, or after 16 creates in
   this run. Post shapes (copy exactly, `mode: customScheduled`, `schedulingType: automatic`,
   `dueAt` with `+05:30` offset):
   - Instagram: `text = igText`, `assets = [{video: {url: videoUrl, metadata: {title: slug,
     thumbnailOffset: coverMs}}}]`, `metadata = {instagram: {type: "reel", shouldShareToFeed:
     true, isAiGenerated: false}}`.
   - YouTube: `text = ytText`, `assets = [{video: {url: videoUrl, metadata: {title:
     ytTitle}}}]`, `metadata = {youtube: {title: ytTitle, categoryId: "27", privacy: "public",
     madeForKids: false, isAiGenerated: false, notifySubscribers: true}}`.
   After EVERY successful create, append the row to `ledger.json` at once (slug, channel,
   postId, dueAt, status "scheduled", videoUrl, createdAt). If a create fails, write the
   error in the final message and do not retry it in this run.
7. Write `ledger.json` (keep the `summary` block current: counts of rows by status per
   channel, and how many queue items are fully done on both channels). Commit
   `ledger: <UTC date-time> +<n> new` and push to main: `git pull --rebase && git push`,
   up to five attempts.
8. Final message, one short paragraph: posts created this run (slug, channel, IST time),
   errors found, and progress (`done X of Y on both channels`). When every queue item is
   done on both channels and nothing is left scheduled, start the message with
   **ALL DONE — disable this routine**.

## Never
- Never delete, edit or reschedule an existing Buffer post.
- Never post anything that is not in `queue.json`.
- Never exceed 8 scheduled posts on a channel, or 16 creates in one run.
- Never run apt-get.
- Never add `Co-Authored-By`, `Claude-Session` or any other Claude attribution line to a commit
  message. The commit message is the one line from step 7 and nothing else.
