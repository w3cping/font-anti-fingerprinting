# Font Anti-Fingerprinting

This is an early draft of a proposal to eliminate font-based fingerprinting on
the web. You're welcome to [contribute](CONTRIBUTING.md)!

<!-- TOC -->

- [Introduction](#introduction)
- [Terminology](#terminology)
  - [Locale](#locale)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [Proposal](#proposal)
  - [Allowed system fonts](#allowed-system-fonts)
  - [Aggressively-cached web fonts](#aggressively-cached-web-fonts)
  - [Explicitly provide uncommon but desired fonts to the browser](#explicitly-provide-uncommon-but-desired-fonts-to-the-browser)
- [Key scenarios](#key-scenarios)
- [Detailed design discussion](#detailed-design-discussion)
  - [When to cache the webfonts](#when-to-cache-the-webfonts)
  - [How to support dynamic font subsetting?](#how-to-support-dynamic-font-subsetting)
- [Considered alternatives](#considered-alternatives)
  - [Just define one list of fonts, not depending on locale](#just-define-one-list-of-fonts-not-depending-on-locale)
  - [Define a set of local fonts](#define-a-set-of-local-fonts)
- [Metrics to justify shipping](#metrics-to-justify-shipping)
- [Stakeholder Feedback / Support / Opposition](#stakeholder-feedback--support--opposition)
- [References & acknowledgements](#references--acknowledgements)

<!-- /TOC -->

## Introduction

Differences in the set of fonts users have installed locally contribute many
bits to a unique fingerprint of that user. TODO: link to some quantification.

## Terminology

### Locale

Here a "locale" is the [tag](https://tools.ietf.org/html/bcp47) negotiated by
the `Accept-Language` header. This is often one language+country pair of the form
"en-GB" but other variations are common, as described in BCP 47:

* ["es-419"](https://www.iana.org/assignments/lang-tag-apps/es-419) represents
  Spanish ('es') appropriate to the UN-defined Latin America and Caribbean
  region ('419').
* "sr-Latn-RS" represents Serbian ('sr') written using Latin script ('Latn') as
  used in Serbia ('RS').
* ["zh-Hans-CN"](https://www.iana.org/assignments/lang-tag-apps/zh-Hans-CN)
  represents Chinese ('zh') written in the Simplified script ('Hans') in China
  ('CN').

The locale is *not* the whole `Accept-Language` list.

## Goals

Make font-based queries useless for distinguishing any two users running:

* the same major version of the same browser
* on the same version of the same operating system
* in the same [locale](#locale).

This identifiability should be minimized as much as possible, with as little
harm as possible to the following use cases:

* Web users who have been installing web fonts over cheap connections to avoid
  spending money or time downloading them on expensive or slow connections,
  shouldn't now have to use the expensive or slow connections.
* Websites that depend on a font that's commonly installed among their visitors,
  but uncommon on the web, should have a way to continue working.
* Web developers should be able to use uncommon fonts on their local machines
  even when developing offline.

## Non-goals

Attempts to reduce identifiability based on locale, browser, or OS choice are
out of scope for this explainer. These are plausibly permanent bits in an active
fingerprint because:

* The site needs the locale to show users content in a language they understand.
* The browser and its version are exposed by
  [`Sec-CH-UA`](https://wicg.github.io/ua-client-hints/#sec-ch-ua).
* The OS and its version are exposed by the opt-in
  [`Sec-CH-UA-Platform`](https://wicg.github.io/ua-client-hints/#sec-ch-platform).

This proposal also does not attempt to prevent sites from distinguishing users
who have customized their [generic font
families](https://drafts.csswg.org/css-fonts-4/#generic-font-families) to use
non-default fonts.

## Proposal

Each browser has a map from a [locale](#locale) to a set of fonts that it
ensures are available locally, and which are the only fonts usable with
`src:local()`(https://www.w3.org/TR/css-fonts-3/#font-face-name-value). There
are several options for which fonts are in this set.

### Allowed system fonts

Browsers only allow *pre-installed* fonts in places like the `@font-face` `src:`
[`local()`](https://www.w3.org/TR/css-fonts-3/#font-face-name-value) function
and the [`font-family`
property](https://www.w3.org/TR/css-fonts-3/#font-family-prop). Safari currently
does this by passing a bit to the OS font lookup routines that prevents them
from finding user- or application-installed fonts. Apple intends to expose that
API publicly, and we hope that other operating systems will follow suit.

### Aggressively-cached web fonts

The set of most-commonly-used web fonts for each locale will be derived from
metrics gathered from browser telemetry. We should share a single list across
all browsers, and publicize this list so developers can rely on it. TODO: Figure
out how much usage makes a font one of the most-commonly-used fonts. Is that a
number or disk-size of fonts, a percentage of page loads, or what?

The first time a user visits a page that uses one of these fonts, it's
downloaded and cached until it's no longer in the set of commonly-used fonts,
which could be forever. See [When to cache the
webfonts](#when-to-cache-the-webfonts).

### Explicitly provide uncommon but desired fonts to the browser

The main source of identification in font-based fingerprinting is the long
tail of fonts users have installed (sometimes w/o realizing it) that they
**do not expect or intend websites to access**; fonts-intentionally installed
for web-use are relatively un-identifying (both because many of these use
cases will put users in significant anonymity sets, largely because users
installing fonts needed for less-common-language sites will already overlap
with the "locale" anonymity set).

Browsers can maintain existing important use cases, w/o (in most cases)
increasing the identifiability of the browser by removing the ambiguity
between "fonts that were installed on the machine for non-web purposes" and
"fonts that were installed on the machine for web purposes".  This should
be accomplished through browser settings / "chrome" allowing users to explicitly
describe which fonts the browser should be able to access
**beyond the OS provided fonts**.

This might take the form of selecting fonts the operating system knows about
(e.g. Windows knows about these 30 fonts Windows didn't ship with, select
which fonts you'd like websites to know about), or an alternate "install
a font into the browser" flow.


## Key scenarios

TODO: look through discussion threads to check that this solves the objections.

## Detailed design discussion

### When to cache the webfonts

It would be safest to pre-cache all of these fonts when a new major version of a
browser is installed, but this might waste valuable bandwidth and disk space for
a font that a particular user never happens to need.

I believe it's also safe to cache each font at the point where it's first used,
as long as the cache never evicts fonts. This allows exactly one site to
determine that a user has not visited any site that either uses the font or has
tried to learn this fact about the user.

If the user removes a locale from their `Accept-Language` list, it's plausible to
evict fonts that aren't common for their new set of locales. If the user then
re-adds that locale, it gives one site another chance to learn something about
that user, but changing the Accept-Language list is rare enough that this seems
acceptable.

### How to support dynamic font subsetting?

https://blog.typekit.com/2015/06/15/announcing-east-asian-web-font-support/
describes a piece of Javascript that dynamically requests the particular
characters within a font that a particular page actually uses. This presents
some difficulties for the approach here, depending on how exactly the Javascript
works:

1. If Adobe has defined a separate URL for each character's font subset, and the
   Javascript requests many of those to cover the characters on a page, the set
   of most-commonly-used web fonts may be large, as it will consist of the set
   of all popular characters from all popular fonts.
2. If Adobe has defined a query parameter or similar that specifies the set of
   characters to include, there will probably be too many subsets for any of
   them to appear commonly-used. These fonts will have to be re-downloaded for
   each top-level site.

There may be space for a new CSS specification to help browsers optimize this
better.

## Considered alternatives

### Just define one list of fonts, not depending on locale

This is likely to break the experience for users who read a minority language
and have expensive mobile data. They'll no longer be able to pre-cache the fonts
they need locally, and double-keyed web storage will prevent them from even
keeping a cache of web fonts across multiple sites.

### Define a set of local fonts

If we pick a set of commonly-used local fonts for each locale now, I believe
we'll have a hard time updating that set as new fonts are developed. By
continually collecting metrics on popular web fonts, we'll naturally notice if
developers like a new font enough.

This suggestion to aggressively cache a widely-used subresource has come up for
Javascript libraries too, with the objection that it advantages the
already-winning frameworks and makes it hard to evolve the web. The same
objection is valid for fonts, but it seems less important to encourage font
evolution.

## Metrics to justify shipping

* Within each locale, <= 0.???% of page loads will be "broken" by the new
  restrictions. This criterion is meant to ensure we don't disenfrancise
  minority languages or scripts.

* <= 0.0???% of overall page loads will be "broken" by the new restrictions.

We consider a page load "broken" if it uses a different font before and after
the new restrictions, and the "after" font was selected by a generic-family name
or is the last-resort font.

* A user who fetches all popular fonts for their locale uses no more than ??MB
  of disk space.

* TODO: Something about how much extra network transfer a ??%ile user uses after the
  new restrictions.

## Stakeholder Feedback / Support / Opposition

* CSSWG: No signals
* Browsers:
  * Chrome: Positive
  * Edge: No signals
  * Firefox: No signals
  * Opera: No signals
  * Safari: Support limiting font variation to be based on (browser,OS,locale).
    Concern about aggressively caching web fonts, especially if it's done before
    the user actually needs the font.
  * Samsung: No signals
  * UC: No signals
* Web developers: No signals

## References & acknowledgements

Many thanks for valuable feedback and advice from:

* Pete Snyder
* Safari for showing that a fixed local font list is web-compatible
* Tab Atkins
* The TAG for writing https://w3ctag.github.io/explainers
* Other CSSWG folks on various bug threads
