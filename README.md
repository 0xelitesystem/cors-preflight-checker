# CORS Preflight Checker

Enter a cross-origin request's method, headers and credentials mode plus the server's preflight and response headers, and see whether the Fetch Standard sends a preflight, which step fails, the header change that fixes it, and where shipping browsers differ from the spec.

**Live demo:** https://0xelitesystem.github.io/cors-preflight-checker/

## Use

1. Fill in the left panel the way the request is written in your code: the page origin, the target URL, the method exactly as typed (case matters), the mode, the credentials mode, one `Name: value` request header per line, and the kind of body.
2. Paste what the server answered into the right panel: the preflight (`OPTIONS`) response, with an optional `HTTP/1.1 204` first line, and the headers of the actual response.
3. Read the verdict. Every verdict is labelled "per the Fetch Standard". You get the first failing step with its section number, every further step that would still fail once that one is fixed, both sides of the handoff side by side (what the browser sends, what the server answered), and the minimal fix: the response headers to add, plus a client side change where one exists.
4. If a sourced engine divergence applies, a separate "Shipping browsers differ here" panel says so and names the source file. The page never says "the browser will block it" without that qualifier.

The page opens on the lead example: a lowercase `patch` against a server that allows `PATCH`.

### When this is useful

- A Chromium-based browser printed a console error containing "blocked by CORS policy" and you have the request and the response headers from the network panel, but not the reason.
- You are about to add a custom header (a tracing ID, an API version) to a cross-origin call and want to know whether that alone introduces a preflight, and what the API has to answer.
- You are reviewing an API gateway or reverse proxy CORS config before deploy: paste the preflight it returns and check it against the requests your frontend makes.
- A teammate's `fetch` with `method: "patch"` fails while the same call from curl works, and you need to show the backend team exactly why (see the lowercase patch section below).
- You are switching a call to `credentials: "include"` and need the full set of response headers that change with it.

## What it models

Each step is labelled on the page with its section of the [Fetch Standard](https://fetch.spec.whatwg.org/), implemented from the edition last updated 21 September 2026.

- **A. Request constructor** ([5.4](https://fetch.spec.whatwg.org/#request-class), [2.2.1](https://fetch.spec.whatwg.org/#methods)). A method that is not a token, or `CONNECT`, `TRACE` or `TRACK` in any case, throws a TypeError. Only `DELETE`, `GET`, `HEAD`, `OPTIONS`, `POST` and `PUT` are upper-cased; every other method goes out verbatim. `no-cors` with anything but `GET`, `HEAD` or `POST` throws, and so does a body with `GET` or `HEAD`. A body with no explicit `Content-Type` gets its implicit type: a string gives `text/plain;charset=UTF-8`, `URLSearchParams` gives `application/x-www-form-urlencoded;charset=UTF-8`, `FormData` gives `multipart/form-data` with a boundary, and a `Blob` gives its type if non-empty. So a JSON string sent with no `Content-Type` does not preflight.
- **B. Headers** ([5.1](https://fetch.spec.whatwg.org/#headers-class), [2.2.2](https://fetch.spec.whatwg.org/#forbidden-request-header)). A forbidden request-header is dropped silently and reported as "never sent". That covers the fixed list in 2.2.2, every name starting `proxy-` or `sec-`, and `X-HTTP-Method`, `X-HTTP-Method-Override` or `X-Method-Override` when a listed method is forbidden.
- **C. Safelist** ([2.2.2](https://fetch.spec.whatwg.org/#cors-safelisted-request-header)). The 128 byte value limit, the per-name rules for `Accept`, `Accept-Language`, `Content-Language`, `Content-Type` (strict [MIME Sniffing](https://mimesniff.spec.whatwg.org/#parsing-a-mime-type) parse, then the three allowed essences) and `Range`, and the [1024 byte total](https://fetch.spec.whatwg.org/#cors-unsafe-request-header-names) after which every safelisted entry counts as unsafe. That last rule is reachable only with repeated entries, so repeated names stay separate entries in the model. The page prints the exact `Access-Control-Request-Headers` value: unique names, lowercased, sorted, joined by a comma with no space.
- **D. Whether a preflight is needed** ([4.1](https://fetch.spec.whatwg.org/#main-fetch)): the method is not exactly `GET`, `HEAD` or `POST`, or at least one header name is CORS-unsafe.
- **E. The preflight** ([4.8](https://fetch.spec.whatwg.org/#cors-preflight-fetch)): the CORS check on the preflight using the request's credentials mode, an [ok status](https://fetch.spec.whatwg.org/#ok-status) (200 to 299, redirects are not followed), list parsing of `Access-Control-Allow-Methods` and `Access-Control-Allow-Headers` ([3.3.4](https://fetch.spec.whatwg.org/#http-new-header-syntax)), the case-sensitive method match, the `Authorization` rule, and the case-insensitive header match. `*` is a wildcard only when credentials mode is not `include`.
- **F. The actual response** ([4.10](https://fetch.spec.whatwg.org/#cors-check)): `Access-Control-Allow-Origin` must be `*` (only without `include`) or the serialized origin byte for byte. A trailing slash, an uppercase host, an explicit default port, or two `Access-Control-Allow-Origin` lines (which combine into one value with a comma) all fail. With `include`, `Access-Control-Allow-Credentials` must be exactly `true`. The actual response's status does not matter.

The checker assumes an empty preflight cache. Not modelled: Local Network Access and private network checks (browser specific, not in the Fetch Standard), redirects of the actual request (a cross-origin redirect taints the origin to `null`), service workers, the use-CORS-preflight flag (for example XHR upload listeners), the [CORS protocol exceptions](https://fetch.spec.whatwg.org/#cors-protocol-exceptions) to the `Content-Type` safelist, `Access-Control-Expose-Headers` beyond one note, mixed content, CORP and COEP.

## The lowercase patch handoff

This is the page's lead example, and it is a case where every party is following its own rules.

- The Fetch Standard [normalizes a method](https://fetch.spec.whatwg.org/#concept-method-normalize) only if it is `DELETE`, `GET`, `HEAD`, `OPTIONS`, `POST` or `PUT`, so `patch` goes out as `patch`. The standard says so in its own note: "Using `patch` is highly likely to result in a `405 Method Not Allowed`. `PATCH` is much more likely to succeed."
- The preflight step in 4.8 checks whether the method is "in" `Access-Control-Allow-Methods`, which is an exact, case-sensitive match. A server answering `Access-Control-Allow-Methods: PATCH` therefore fails a `patch` request.
- HTTP agrees with the server: [RFC 9110 section 9.1](https://www.rfc-editor.org/rfc/rfc9110#section-9.1) says "The method token is case-sensitive".
- Chromium's network service compares case-sensitively by default: `network_context.mojom` declares `bool acam_preflight_spec_conformant = true;` and [`preflight_result.cc`](https://raw.githubusercontent.com/chromium/chromium/main/services/network/cors/preflight_result.cc) still carries a legacy upper-casing path marked for deprecation (crbug.com/1228178). Firefox's [`nsCORSListenerProxy.cpp`](https://raw.githubusercontent.com/mozilla-firefox/firefox/main/netwerk/protocol/http/nsCORSListenerProxy.cpp) uses `mPreflightMethod.Equals(method)`, and WebKit's [`CrossOriginPreflightResultCache.cpp`](https://raw.githubusercontent.com/WebKit/WebKit/main/Source/WebCore/loader/CrossOriginPreflightResultCache.cpp) uses `m_methods.contains(method)`.

The server half: a stock Node http server answers a lowercase patch request line with 400 Bad Request before any handler runs. Reproduce it with this script, which was run on Node v25.9.0 on 2026-10-01 and printed `HTTP/1.1 400 Bad Request`; the same script with `PATCH` printed `HTTP/1.1 200 OK`.

```js
import http from 'node:http';
import net from 'node:net';
const srv = http.createServer((req, res) => { res.end('handler ran: ' + req.method); });
srv.listen(0, '127.0.0.1', () => {
  const s = net.connect(srv.address().port, '127.0.0.1', () => s.write('patch / HTTP/1.1\r\nHost: x\r\nConnection: close\r\n\r\n'));
  s.once('data', d => { console.log(process.version, String(d).split('\r\n')[0]); s.destroy(); srv.close(); });
});
```

So allowing `patch` in `Access-Control-Allow-Methods` only moves the failure to the server's router. The page's Minimal fix panel therefore leads with the client change, write `PATCH`, whenever the method is a lowercase or mixed-case form of `PATCH` or of a method already in `Access-Control-Allow-Methods`. The server-side headers come after it, labelled as the answer to use only if the request has to stay as written.

## Shipping browsers differ here

The tool models the spec. These two panels appear only when they apply, and each cites the source it rests on. All sources were read on 2026-10-01.

**Authorization under `*`.** The Fetch Standard makes `Authorization` the only [CORS non-wildcard request-header name](https://fetch.spec.whatwg.org/#cors-non-wildcard-request-header-name): under 4.8 a request carrying it fails unless `Access-Control-Allow-Headers` names it explicitly, even when the list contains `*`. Shipping engines currently let `*` cover it when credentials mode is not `include`:

- Chromium: [`services/network/public/cpp/features.cc`](https://raw.githubusercontent.com/chromium/chromium/main/services/network/public/cpp/features.cc) declares `kCorsNonWildcardRequestHeadersSupport` as `FEATURE_DISABLED_BY_DEFAULT`, and Blink's [`deprecation.json5`](https://raw.githubusercontent.com/chromium/chromium/main/third_party/blink/renderer/core/frame/deprecation/deprecation.json5) carries an `AuthorizationCoveredByWildcard` deprecation for this case.
- Firefox: [`modules/libpref/init/StaticPrefList.yaml`](https://raw.githubusercontent.com/mozilla-firefox/firefox/main/modules/libpref/init/StaticPrefList.yaml) sets `network.cors_preflight.authorization_covered_by_wildcard` to `true`, and `nsCORSListenerProxy.cpp` logs a deprecation warning and continues.
- WebKit: `validateCrossOriginHeaders` in `CrossOriginPreflightResultCache.cpp` treats `*` as covering every header, with no `Authorization` exception.
- [MDN browser-compat-data](https://github.com/mdn/browser-compat-data/blob/main/http/headers/Access-Control-Allow-Headers.json), subfeature `authorization_not_covered_by_wildcard`: chrome `false`, safari `false`, firefox `115` only with that preference set to `false`.

The page renders this as a split verdict: "Fetch Standard: fails at 4.8. Shipping browsers: currently allowed, with a deprecation warning in Chromium and Firefox." The fix it gives is to list `Authorization` explicitly, which passes under both readings. The split headline appears only when the `Authorization` rule is the one thing standing between the request and a pass. If another step that browsers enforce also fails, such as a missing `Access-Control-Allow-Origin`, the headline is a plain FAIL that names that step, and this panel stays as a secondary note. Under `credentials: "include"` the `*` is a literal name, and the request fails in the spec and in the engines alike.

**Chromium's extra safelisted names.** Chromium's [`services/network/public/cpp/cors/cors.cc`](https://raw.githubusercontent.com/chromium/chromium/main/services/network/public/cpp/cors/cors.cc) (`IsCorsSafelistedHeader`) accepts these names on top of the spec's five: `intervention`, `device-memory`, `dpr`, `width`, `viewport-width`, `rtt`, `downlink`, `ect` and `save-data` (plus `sec-ch-*` names, which scripts cannot set). Chromium applies its own value checks to them; for `save-data` the lowercased value must be `on`. The spec says such a header forces a preflight, so when one appears the page notes that Chromium-based browsers may skip the preflight the spec requires, and that Firefox follows the spec.

**Preflight cache ceilings.** The spec default when `Access-Control-Max-Age` is absent is 5 seconds (4.8). Engines cap larger values: Chromium at 2 hours (`kMaxTimeout = base::Hours(2)` in `services/network/cors/preflight_result.cc`), Firefox at 86400 seconds (`nsCORSListenerProxy.cpp`), WebKit at 600 seconds (`maxPreflightCacheTimeout = 600_s` in `CrossOriginPreflightResultCache.cpp`). A cached earlier answer can make a fix look like it did nothing for that long.

## Your browser, right now

A panel on the page constructs `Request` objects (with an absolute URL; constructing one sends nothing) and shows what your engine did next to what the spec expects: method casing for `patch`, `Patch`, `delete` and `foo`, TypeErrors for `CONNECT` and `track`, silent dropping of `Cookie`, `Origin`, `Sec-Foo` and `X-HTTP-Method-Override: TRACE` while `X-HTTP-Method-Override: PATCH` is kept, a TypeError for a GET with a body, the implicit `text/plain;charset=UTF-8` Content-Type of a string body, the `no-cors` limits, and the default credentials and mode. A mismatch is flagged, never hidden.

## Self test

The page runs its self test on load and prints a live pass and fail count. It covers the method cases (including the lowercase patch, normalized `delete` and `put`, and the `*` method wildcard), the `Authorization` split, the credentials cases, the preflight triggers (`Content-Type` parse, the 128 byte limit on either side, `Content-Language`, simple and suffix ranges, `Save-Data` and `DPR`), duplicate and trailing-slash `Access-Control-Allow-Origin`, preflight status and list parsing, the 1024 byte total and its control, a 302 preflight, and all seven rows of the credentials table in Fetch Standard [3.3.5](https://fetch.spec.whatwg.org/#cors-protocol-and-credentials), encoded verbatim. Rows the author never measured in a browser are marked "spec only". The last row is a positive control written to fail; if it ever holds, the suite is not running.

## Why this exists

The console error for a CORS failure names a symptom. The cause sits in an exchange between two parties who each follow their own rules, and you only see it once the browser's half and the server's half are laid side by side with the step that rejected them. This page does that, with the spec section next to every step, and it is honest about the places where shipping browsers do something else.

Single file, no build step, no network call, no tracking, MIT licensed. Paste production headers without wondering where they go.

## Privacy

Everything runs in your browser. The page makes no network call of any kind: no code, font, analytics or telemetry request, and nothing you paste is transmitted. A Content-Security-Policy in the page forbids network connections. The only thing it stores is your light or dark theme choice, in your browser's local storage. Save the file and it works offline from `file://`.

## Run locally

```
git clone https://github.com/0xelitesystem/cors-preflight-checker.git
cd cors-preflight-checker
```

Open `index.html` in a browser. Or serve it:

```
python -m http.server 8000
```

then open `http://localhost:8000/`.

## Build

No build. One self-contained `index.html` with inline CSS and JavaScript. No dependencies, no package manager, no bundler, no CDN, no web font.

## Independence and provenance

This is an independent tool. It is not affiliated with, endorsed by, or connected to the WHATWG, Google, Mozilla, Apple, Microsoft or any other browser vendor, or MDN. Behaviour was implemented from the specification text and engine source was read for the divergence notes; no code is reproduced here.

## Third-party notices

**WHATWG Fetch Standard** (https://fetch.spec.whatwg.org/) and **MIME Sniffing Standard** (https://mimesniff.spec.whatwg.org/). Holder line, copied from the footer of each spec: "Copyright © WHATWG (Apple, Google, Mozilla, Microsoft)." Licensed under the Creative Commons Attribution 4.0 International License (CC BY 4.0). What was taken: the short rule wording quoted on the page and in this README, including the 2.2.1 note on `patch`, and the rows of the 3.3.5 credentials table used as self test fixtures. Implemented against the Fetch Standard as last updated 21 September 2026 and the MIME Sniffing Standard as last updated 17 July 2026.

**MDN browser-compat-data** (https://github.com/mdn/browser-compat-data). Licensed CC0 1.0 Universal. What was taken: the support values of the subfeature `authorization_not_covered_by_wildcard` under `http.headers.Access-Control-Allow-Headers`, cited above.

**Chromium** (https://source.chromium.org/chromium). Holder line, copied from https://raw.githubusercontent.com/chromium/chromium/main/LICENSE: "Copyright 2015 The Chromium Authors". Licensed BSD-3-Clause. What was taken: the single phrase "blocked by CORS policy", quoted in the Use section above as a search phrase, from `third_party/blink/renderer/platform/loader/cors/cors_error_string.cc`. The identifiers named in the divergence notes are cited as references, not copied code.

## Related

- https://github.com/0xelitesystem/cors-basics-reference (this page is the hands-on answer to its `05-common-errors` chapter)
- https://github.com/0xelitesystem/http-header-analyzer
- https://github.com/0xelitesystem/curl-command-builder

## License

MIT. Copyright (c) 2026 0xelitesystem. See [LICENSE](LICENSE).
