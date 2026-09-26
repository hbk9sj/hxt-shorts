# hxt-shorts

Queue and posting record for Hit x Trial Shorts, posted to Instagram (Reel) and YouTube (Short)
through Buffer.

- `queue.json` — the videos to post, in order, with titles and captions. Video files live in
  [hxt-shorts-media](https://github.com/hbk9sj/hxt-shorts-media) (Buffer fetches them by raw URL).
- `ledger.json` — every Buffer post made so far: slug, channel, post id, time, status, link.
  This is the record that stops a video being posted twice.
- `routine/brief.md` — what the scheduled top-up routine does on each run.

Each video ends with a one-second card showing its cover image; the Instagram post's cover
(`thumbnailOffset`) points at that card, because Buffer does not accept custom thumbnails.
