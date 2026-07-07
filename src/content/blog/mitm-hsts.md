---
title: "Why My MITM Proxy Worked on Digikala but Not Maktabkhooneh: A Lesson in HSTS"
description: "I tried to intercept HTTPS traffic with mitmproxy. Some sites let me through, one refused completely. Here's what I learned about why."
pubDate: 'Jul 07 2026'
lang: en
heroImage: '../../assets/mitm-hsts.png'
---

I've been studying for the CEH, and one of the topics is intercepting traffic — the classic man-in-the-middle setup. So I decided to actually try it on my own machine instead of just reading about it. What I expected to be a quick lab turned into a small mystery that taught me more than the lab itself. This is a write-up of what happened, why it happened, and the wrong assumption I made along the way.

> A note before we start: everything here was done inside my own Kali VM, against my own browser, purely to understand how the mechanism works. Intercepting other people's traffic without permission is illegal. This post is about the *defensive* concept — why some sites can't be intercepted — not a how-to for attacking anyone.

## The setup

My lab was simple:

- A Kali Linux VM.
- `mitmproxy` running in the terminal, listening on port 8080.
- Chrome inside Kali, with the **Proxy SwitchyOmega** extension pointing all traffic at `localhost:8080`.

The idea: route the browser through mitmproxy so I could watch the requests go by. For plain HTTP this is trivial. For HTTPS it's more interesting, and that's where the story is.

## What I saw

I opened a few Iranian sites, all served over HTTPS.

On **digikala.com**, the browser threw up a "your connection is not private" warning. In the mitmproxy terminal I could see the data flowing and mitmproxy generating its own certificate on the fly. Chrome didn't trust that certificate — but at the bottom of the warning page there was the familiar option to **continue anyway (unsafe)**. I clicked it, and I was in.

On **snapp.ir**, exactly the same thing: warning, then a "continue unsafely" escape hatch.

Then I tried **maktabkhooneh.org**, and everything changed. There was no warning-with-a-way-through. There was no "proceed anyway" button at all. The browser just refused. It got weirder: even typing a plain search term like `water` into the address bar wouldn't go through while the proxy was intercepting. The site — and eventually the browser itself — simply would not talk to me through the proxy.

## My first (wrong) theory

My initial guess was that it had something to do with the **domain**. Maybe `.ir` sites behaved differently, or maybe the site needed a specific TLD to load. I even wondered if I had to feed it a direct link for it to work.

But the evidence didn't fit. Digikala is a `.com` and it loaded (with the warning). Maktabkhooneh is a `.org` and it refused completely. If the TLD were the deciding factor, these two should have behaved the same way — and they didn't. So the domain theory was dead.

This is the part I want to emphasize, because it's the actual lesson: **when your explanation predicts two things should behave the same and they don't, the explanation is wrong.** Time to find a better one.

## Why HTTPS interception triggers a warning at all

To understand the real cause, you first have to understand *why* there's a warning in the first place.

When mitmproxy sits in the middle of an HTTPS connection, it can't just read the encrypted traffic — that's the whole point of TLS. So instead, it does something clever: it terminates the TLS connection itself, and to your browser it presents a certificate it generated on the spot, claiming to be the site.

The problem is *who signed that certificate*. A real site's certificate is signed by a trusted Certificate Authority (CA) that your browser already trusts. Mitmproxy's certificate is signed by mitmproxy's own CA, which your browser has never heard of. So the browser looks at it and says, correctly: "this certificate isn't trusted — someone might be intercepting you." That's the "not private" warning.

So far, this is identical for every site. It explains the warning on Digikala and Snapp. It does **not** yet explain why Maktabkhooneh behaved differently. For that, we need one more piece.

## The real cause: HSTS

The difference comes down to **HSTS** — *HTTP Strict Transport Security*. (I confirmed this afterwards; the query output is at the end of the post.)

HSTS is a policy a website can send to your browser (via a response header) that says, roughly: "From now on, only ever talk to me over *valid* HTTPS. If the certificate is broken or untrusted, do not show the user a way to click through. Just refuse."

Once a browser has seen that header from a site, it remembers it. After that, a bad certificate on that site isn't a warning you can dismiss — it's a hard stop, by design. There is deliberately no "continue unsafely" button.

That lines up perfectly with what I saw:

- **Digikala and Snapp**: no strict HSTS policy in effect for my browser, so the untrusted mitmproxy certificate produced a *warning* — with the option to proceed.
- **Maktabkhooneh**: an HSTS policy in effect, so the browser rejected mitmproxy's fake certificate outright, with no way through.

And the "even `water` won't search" detail fits too: Chrome sends your address-bar searches over HTTPS as well. With the proxy mangling every TLS connection, those requests were failing for the same reason.

The deciding factor was never `.ir` vs `.com`. It was **each site's own security policy.**

## Confirming it (don't just trust me)

I didn't want to present a guess as a fact, so I verified it instead of assuming. In Chrome, open:

```
chrome://net-internals/#hsts
```

Under **Query HSTS/PKP domain**, type the domain and hit query. For `maktabkhooneh.org`, here's the part of the result that matters:

```
dynamic_sts_domain: maktabkhooneh.org
dynamic_upgrade_mode: FORCE_HTTPS
```

That's the confirmation. The site has an active HSTS policy, and the browser is enforcing `FORCE_HTTPS` for it — which is exactly why there was no "proceed anyway" button.

One detail worth noticing: the result came back under `dynamic_sts_domain`, not `static_sts_domain`. That distinction matters:

- **Static (preloaded) HSTS** means the domain ships inside Chrome's built-in preload list, so the browser enforces HTTPS from the very first visit, before it has ever talked to the site.
- **Dynamic HSTS** means the browser learned the policy *after* visiting the site once and receiving its HSTS header, then stored it (with an expiry).

Maktabkhooneh's entry was dynamic — my browser had picked up the policy from an earlier visit and remembered it. So the protection wasn't baked into Chrome; it came from the site telling my browser "always HTTPS," and my browser honoring that afterwards.

## What I took away from this

Three things stuck with me:

1. **A failed intercept can teach you more than a successful one.** The site I *couldn't* touch is the one that taught me what actually protects users.
2. **HSTS is a genuinely strong defense.** It closes the exact "just click continue" gap that makes so many TLS warnings useless in practice, because most people click through them.
3. **Check your assumptions against the evidence.** My domain theory felt reasonable until two data points killed it. Following the contradiction is what led to the real answer.

I'm still early in this — I'm learning as I go, and I'm sure there's more nuance here (certificate pinning, preload lists, and so on) that I haven't dug into yet. But that's kind of the point of writing these down: next time I hit something like this, I'll have a map.

If you're studying the same material and spot something I got wrong, I'd genuinely like to hear it.
