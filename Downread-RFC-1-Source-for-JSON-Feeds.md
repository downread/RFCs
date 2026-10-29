# RFC: Source Namespace Extensions for JSON Feed

Oct 29, 2026 · @Tim Jansen

## Summary

This proposal adds five fields from the RSS [source namespace](https://source.scripting.com/) to JSON Feed: who a post replies to, where its comments are, how many comments it has, the author's accounts on other services, and, for comments, the text as Markdown. They live in one extension object, `_source`, so a reader can treat a JSON feed exactly like an RSS feed that uses the namespace.

It also adds one thing the namespace lacks today: a **total** comment count next to the number of direct replies. rss.chat shows only the first-level count, so a thread with 2 replies and 30 answers to them looks quiet.

**Why now.** Social-RSS systems like rss.chat already link posts, replies and comment feeds with `source:inReplyTo`, `source:comments`, `source:account` and `source:markdown`. JSON Feed has no fields for any of this, so a blog that publishes JSON Feed drops out of those conversations. Mirroring the RSS elements one to one keeps a single model for both formats: publishers can generate both from the same data, and readers can use one code path.

## The `_source` object

Everything goes into one object named `_source`, following JSON Feed's [extension rules](https://www.jsonfeed.org/version/1.1/#extensions-a-name-extensions-a): the name starts with an underscore, and the object says which spec it implements in `about`.

```json
"_source": {
  "about": "http://source.scripting.com/"
}
```

- It may appear on an **item** (`inReplyTo`, `comments`, and in comment feeds `markdown`) and on the **feed** itself (`accounts`).
- `about` should be `http://source.scripting.com/`, the namespace URI the RSS elements use. Readers should not reject `https://` or a missing trailing slash.
- Every field is optional. A reader ignores fields it doesn't know and values of the wrong type, so the object can grow later.

## Item: `inReplyTo`

`inReplyTo` names the post this item answers. Usually it is just the permalink of that post, as a string:

```json
"_source": {
  "about": "http://source.scripting.com/",
  "inReplyTo": "https://rss.chat/?id=204"
}
```

When the target has no URL (a Bluesky post, say), use an object with the id and `isPermaLink: false`, like the RSS element's attribute:

```json
"inReplyTo": { "value": "at://did:plc:xyz/app.bsky.feed.post/3k2", "isPermaLink": false }
```

Readers show it as "In reply to …": a link for a permalink, plain text for an id. A post replies to one item; for several, pick the main one.

## Item: `comments`

`comments` says where an item's comments are and how many there are. It has three fields, all optional:

| Field | Type | Meaning |
| --- | --- | --- |
| `feedUrl` | URL | A feed (RSS, Atom or JSON) whose items are the direct replies to this item. Same as `source:comments@feedUrl`. |
| `count` | integer ≥ 0 | Number of **direct** replies: the items in `feedUrl`. Same as `source:comments@count`. |
| `total` | integer ≥ 0 | **New.** Number of replies at **all** levels: direct replies, their replies, and so on. Never less than `count`. |

```json
"_source": {
  "about": "http://source.scripting.com/",
  "comments": {
    "feedUrl": "https://rss.chat/users/dave/comments/205.xml",
    "count": 2,
    "total": 31
  }
}
```

**Why both counts.** A reader shows `count` on the button that opens the comments, because that is what the feed it loads contains. `total` answers "is anything going on here?": with only the first level, a post with 2 replies that drew 29 answers looks like a quiet one. Publishers that only know one number give `count`; a publisher with flat comments (WordPress without threading) has `count` = `total` and may omit `total`.

Replies are posts too, so an item in a comment feed carries its own `comments` for the next level. Counts are a snapshot: readers should not expect them to update with every new comment, and should treat them as a hint, not a promise of what the feed holds.

## Comment item: `markdown`

`markdown` holds a comment's text as Markdown, like `<source:markdown>` in RSS. It is meant for comment feeds only: the items of a feed that a `comments.feedUrl` points to.

```json
{
  "id": "https://rss.chat/?id=205",
  "url": "https://rss.chat/?id=205",
  "authors": [{ "name": "Dave Winer" }],
  "date_published": "2026-07-08T14:42:55Z",
  "content_html": "<p>hey manton, great to see you here.</p><p>this is the <strong>web</strong> btw.</p>",
  "_source": {
    "about": "http://source.scripting.com/",
    "markdown": "hey manton, great to see you here.\n\nthis is the **web** btw."
  }
}
```

- Publishers still write `content_html` or `content_text` (JSON Feed requires one), with the same text, for readers that don't know `markdown`.
- Readers showing a comment prefer `markdown` over `content_html` and `content_text`, as they prefer `<source:markdown>` over `<description>` in RSS. Markdown is easier to render safely: a reader can build the comment from it without trusting the publisher's HTML.
- On items of regular feeds (posts), readers ignore it. A post's body stays `content_html`.

## Feed: `accounts`

`accounts` lists the feed author's accounts on other services, one entry per service, like `source:account` elements on an RSS channel:

```json
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": "Example Blog",
  "_source": {
    "about": "http://source.scripting.com/",
    "accounts": [
      { "service": "socialspec", "name": "@example.com" },
      { "service": "mastodon", "name": "@alice@mastodon.social" }
    ]
  },
  "items": []
}
```

- `service`: the service name, compared case-insensitively (`Mastodon` = `mastodon`).
- `name`: the user name on that service, written the way the service writes it (with the `@` where it has one).
- A reader keeps the first entry per service and may ignore services it doesn't know.
- Readers should also accept a single object under the name `account`, which some early feeds use.

## Mapping to RSS and Atom

Each field has an RSS counterpart, so a publisher can write both formats from the same data and a reader can convert between them without loss. Older WordPress and Atom feeds express some of the same things; readers that support them can fill `_source`'s fields from them.

| JSON `_source` | RSS source namespace | WordPress RSS | Atom (RFC 4685) |
| --- | --- | --- | --- |
| `inReplyTo` (string) | `<source:inReplyTo>` | — | `<thr:in-reply-to href>` |
| `inReplyTo` (object, `isPermaLink: false`) | `<source:inReplyTo isPermaLink="false">` | — | `<thr:in-reply-to ref>` without `href` |
| `comments.feedUrl` | `<source:comments feedUrl>` | `<wfw:commentRss>` | `<link rel="replies" type="application/atom+xml">` |
| `comments.count` | `<source:comments count>` | `<slash:comments>` | `thr:count` on that `replies` link |
| `comments.total` | none yet (see open questions) | — | `<thr:total>` |
| `markdown` (comment items) | `<source:markdown>` | — | — |
| feed `accounts` | `<source:account service>` on the channel | — | — |

When a feed carries both the source elements and an older form, the source elements win.

**Atom counts.** RFC 4685 has two numbers: `thr:count` on a `replies` link counts the entries in that linked feed, `<thr:total>` counts all responses to the entry. Take `thr:count` for `count` and `<thr:total>` for `total`. Blogger writes only `<thr:total>`; its comment feed is flat and holds every comment, so there the total is also the first-level count.

## Notes for publishers and readers

**Publishers**

- Use absolute `http(s)` URLs. Write `_source` only when it has at least one field.
- If you also publish RSS, write the same values there: `inReplyTo`, `comments`, `accounts` and a comment's `markdown` should never disagree between your two feeds.
- A comment feed may be RSS, Atom or JSON Feed; readers should handle all three.

**Readers**

- Feeds are untrusted: drop URLs that are not `http(s)`, counts that are not non-negative integers, and `total` values below `count`.
- If `count` promises comments but `feedUrl` returns an empty feed or an error, say so instead of showing "no comments".
- Comment feeds often live on other sites and send no CORS headers, so browser-based readers need a fallback (a proxy).

## Full example

```json
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": "Example Blog",
  "home_page_url": "https://example.com/",
  "feed_url": "https://example.com/feed.json",
  "_source": {
    "about": "http://source.scripting.com/",
    "accounts": [{ "service": "socialspec", "name": "@example.com" }]
  },
  "items": [
    {
      "id": "https://example.com/posts/42",
      "url": "https://example.com/posts/42.html",
      "content_html": "<p>I disagree, and here is why…</p>",
      "date_published": "2026-10-29T09:00:00Z",
      "_source": {
        "about": "http://source.scripting.com/",
        "inReplyTo": "https://rss.chat/?id=204",
        "comments": {
          "feedUrl": "https://example.com/posts/42/comments.json",
          "count": 3,
          "total": 7
        }
      }
    }
  ]
}
```

## Open questions

- **A `total` attribute for RSS.** The same gap exists in `<source:comments>`. Proposal: an optional `total` attribute, `<source:comments count="2" total="31" feedUrl="…"/>`, so rss.chat can show thread size. This needs the namespace's maintainers to agree.
- **Field names.** `total` mirrors Atom's `thr:total`; `totalCount` would be more explicit. Pick one before anyone ships it.
- **Where `about` points.** Should it name the RSS namespace, as now, or a page for this JSON mapping?
