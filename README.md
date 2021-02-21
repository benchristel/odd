# odd

odd (for _OligoDistributed Data_) is a protocol for
oligodistributed sharing of
[hypermedia](https://en.wikipedia.org/wiki/Hypermedia)
between servers, clients, and peers.

**Oligodistributed** means the protocol relies neither on
the continuous availability of a single authoritative
source for a file (as
[HTTP](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol)
does) nor on the collective memory of a network of peers
(as, say, [IPFS](https://ipfs.io/) does). Instead, odd lets
hypermedia links specify an authoritative source for the
data, along with a content hash that can be used to fetch
the data from a cache, archive, or peer if the authority
can't be reached. This has several advantages:

- odd can be layered on top of HTTP and HTML in a way that
  allows user agents with no awareness of odd to fall back
  to plain old HTTP.
- odd-aware user agents can still use odd to fetch content
  even if the authoritative server for that content is not
  aware of odd.
- Content authors can issue updates to previously-published
  content at the same URL. HTTP-only user agents will simply
  see the updates; odd-aware user agents can offer the user
  a choice between the old and new versions of the content.
- Peer-to-peer file sharing is possible, but odd does not
  _depend_ on the existence of a robust community of peers
  before it can be used practically. This solves the
  "network effect" problem of existing peer-to-peer
  solutions.
- If an authority goes away, fallback to a
  cache/archive/peer is seamless and secure (in the sense
  that you're guaranteed to either get the content that the
  link you followed meant to give you, or get nothing).

## Embedding odd in HTML

odd links in HTML look like this:

```html
<a href="https://example.com/foo/bar.txt"
   x-odd-hash="5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03"
   x-odd-strategy="content-must-match"
>
  Link text
</a>
```

That is, they include a hex-encoded SHA256 hash of the
contents of the linked file. Of course, if the linked file
is modified on the server, the hash won't match it anymore.
If that happens, the user agent has a few different options
for resolving the ambiguity, described in later sections.

Representations of odd links may differ between markup
languages, but they _always_ include a content hash. The
authority and strategy are optional.

## Requesting Content

To request content from a cache/peer/archive over HTTP, an
odd client can make a GET request to a URL like this:

```
https://some.archive.example.com/odd/5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03
```

That is, the path portion of the URL consists of `/odd/`
followed by the content SHA.

The server MUST send content in the HTTP response body that
matches the requested hash. The client MUST reject data that
does not match the hash. The server SHOULD set mime-type
headers, etc. appropriately for the content.

## Requesting a File by Name

To request a file from an authority, an odd client can make
a GET request for the file's path prefixed by `/odd/n` (n
for "name"). For example, to request the file `/foo/bar.txt`
via odd, you'd send this HTTP request:

```http
GET /odd/n/foo/bar.txt HTTP/1.1
```

The server responds with the hash of the file's contents.
This SHOULD match the file currently available at `/foo/bar.txt`:

```
5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03
```

If the content hash does not match the one in the link the
user-agent was following, it means the server has updated
the file since the link was created (or perhaps the link
intentionally referenced an old version of the file).
The user agent can then use one of several strategies to
resolve the ambiguity (see below for details).

To speed up content fetching, the client can optimistically
request `/foo/bar.txt` in parallel with
`/odd/n/foo/bar.txt`. If the response from `/foo/bar.txt`
doesn't match the SHA from `/odd/n/foo/bar.txt`, the client
MUST reject the content (this can happen if the server is in
the middle of an update, and the hashes have been updated
but not the files, or vice versa). The client can then try
fetching the content by hash, e.g. by sending the authority
a GET request for `/odd/5891b5...` or by making the same
request of a cache/archive/peer.

## Query Parameters

A URL fetched with odd can include query parameters, though
if the content is dynamic and changes frequently this may be
somewhere between useless and confusing. The query
parameters are simply appended to `/odd/n` URLs:

```
GET /odd/n/foo/bar?q=123 HTTP/1.1
```

## Lookup Strategies

Links to content can specify a _strategy_ to use when it's
ambiguous how to resolve the content (e.g. because
the authority couldn't be reached, or responded with content
different from the hash that was requested). Strategies
include:

- `content-must-match`: Caches/archives/peers MAY be
  consulted before the authority; if they have the content,
  the authority does not need to be contacted. If the
  authority _is_, contacted, and offers updated
  content, it is refused. In no case does the link resolve
  to anything but the content matching the hash.
- `content-first`: the user agent first fetches the content
  by hash, and meanwhile consults the authority to see if
  there is an updated version. If an update is found, the
  user agent MAY inform the user that an update is available
  and provide a link to it.
- `update-first`: The authority SHOULD be consulted before
  any cache/archive/peer. If it offers updated content, that
  content is accepted. The user agent SHOULD notify the user
  of the update and offer a switch back to the version
  specified in the link.
  If the authority responds to the `/odd/n/...` request with
  the same hash specified in
  the link, the content MAY be fetched from a
  cache/archive/peer, or from the authority. If the
  authority cannot be reached, a cache/archive/peer SHOULD
  be used to fetch the (possibly outdated) content and the
  user agent SHOULD notify the user that the authority was
  unavailable.
- `always-update`: The user agent gets the most up-to-date
  content from the authority without informing or prompting
  the user. If the authority is unavailable, a
  cache/archive/peer can be used as a fallback.

If no strategy is specified, the default depends on the
kind of content being fetched:

- If the user agent is fetching a top-level document (e.g.
  an HTML page or PDF), the default is `update-first`.
- If the user agent is fetching an image, script,
  stylesheet, or other dependency of a top-level
  document, the default is `content-must-match`.

The user agent MAY provide controls for the user to override
any of this behavior.

## If the server doesn't support odd

If a client tries to use odd with a server that doesn't
support it, requests to `/odd/<sha>` and `/odd/n/<filename>`
will fail. This need not be fatal; the client can simply try
requesting `/<filename>` (i.e. falling back to plain old
HTTP).

## TODO: what happens if the authority deletes a file?
