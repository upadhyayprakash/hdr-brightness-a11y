# Explainer: a user preference for limiting HDR brightness

Author: Prakash Upadhyay  
Published: 18 Oct 2026. Feedback welcome: [open an issue](https://github.com/upadhyayprakash/hdr-brightness-a11y/issues).  
Versions: [balanced](./) · this page (technical) · [plain language](plain.md) · [live demo](demo/)  
Related: [csswg-drafts #11790](https://github.com/w3c/csswg-drafts/issues/11790), [CSS Color HDR 1](https://drafts.csswg.org/css-color-hdr/), [Media Queries 5](https://www.w3.org/TR/mediaqueries-5/)  

## The problem

HDR screens can show content much brighter than normal page white. That is great for a movie. It is not great when a single image on an ordinary page uses that headroom and suddenly glows several times brighter than everything around it.

This happens today, by accident. A 100 x 100 company logo in a LinkedIn job alert, viewed in Yahoo Mail in Chrome on a MacBook XDR display, rendered far brighter than the rest of the email. Its JPEG carries an Apple "Rec. ITU-R BT.2100 PQ" color profile, so its white pixels ask for 10,000 nits, about 49 times the 203 nits that ITU-R BT.2408 recommends for graphics white. Nobody meant it to glow. It was most likely exported from an Apple tool with an HDR color space selected.

A screenshot of the page looks normal, because screenshots are saved in SDR and clip anything above normal white. So the problem is invisible in design reviews, bug reports and chat threads. You only see it live, on an HDR screen.

**The harm is the sudden jump, not the overall brightness.** Scrolling a dim inbox at night and hitting one HDR logo is like a TV ad that is much louder than the programme around it.

## What exists today, and why it is not enough

| What | Who controls it | Gap |
|---|---|---|
| `dynamic-range-limit: standard \| constrained \| no-limit` (CSS Color HDR 1) | The developer | Only works if the developer knows about HDR and sets it. Used on 0.66% of Chrome page loads. |
| `@media (dynamic-range: high)` (Media Queries 5) | The hardware | Says the screen *can* show HDR, not whether the person *wants* it. |
| Reduce White Point (iOS, iPadOS) | The user | Dims the whole screen, including the text you need to read. The bright element is still several times brighter than its surroundings. |
| Android "Enhanced HDR brightness" | The user | Reported in beta builds only. Not exposed to apps. |
| Windows HDR on/off per display | The user | All or nothing for the whole display. |
| Native APIs: `UIImageView.preferredImageDynamicRange`, SwiftUI `.allowedDynamicRange(_:)`, `CALayer.DynamicRange`, Android `Window.setDesiredHdrHeadroom()` | The developer | Same as CSS: they let apps cap HDR, but none tells the app what the user prefers. |

The [CSS Color HDR 1 spec, Accessibility Considerations](https://www.w3.org/TR/css-color-hdr-1/#a11y) (Working Draft, 8 October 2026) already says:

> "Some individuals may have a sensitivity to very bright colors, so user agents should provide a mechanism to limit the maximum luminance at user option."

It is a "should", not a "must", and the spec does not define the mechanism. Its only concrete suggestion is that `dynamic-range-limit` "could also be set to standard or constrained in a user stylesheet", which in today's cascade loses to author rules unless marked `!important` (the point the spec editor raised in #11790). I found no browser that offers such a setting today.

Today a developer can cap HDR for everyone on an HDR screen, or for no one. They cannot do it just for the people who need it.

### Usage data (Chrome Platform Status, 16 Oct 2026)

| Feature | Share of Chrome page loads |
|---|---|
| `prefers-reduced-motion` | 58.3% |
| `prefers-contrast` | 27.2% |
| `dynamic-range` (HDR capability) | 6.6% |
| `prefers-reduced-transparency` | 5.5% |
| `dynamic-range-limit` (HDR cap) | 0.66% |

Pages check whether the screen can show HDR ten times more often than they cap it, and the cap appears on about 1/90th as many page loads as reduced-motion. Meanwhile HDR content keeps growing from the camera side: phones save HDR photos by default, and anything users upload (avatars, logos, product photos) can bring HDR into pages whose developers never chose it.

## Goals

- Let a person say "no sudden brightness above normal white" once, and have pages and apps respect it.
- Protect users even on sites whose developers do nothing.
- Let developers who care adapt their design, not just their brightness (an SDR poster instead of an HDR hero video, an SDR logo, no HDR autoplay).
- Fit the existing `dynamic-range-limit` model instead of inventing a new one.

## Non-goals

- Changing how HDR is measured or encoded.
- Replacing OS display settings such as Reduce White Point or night modes.
- Defining flash or seizure thresholds for HDR (a related gap, see "Related work").

## Use cases

**Call for use cases.** The CSS Working Group asks for concrete, real-world use cases before it adds a media feature ([#6757](https://github.com/w3c/csswg-drafts/issues/6757)). If sudden HDR brightness affects you, or people you support, please [share your experience in an issue](https://github.com/upadhyayprakash/hdr-brightness-a11y/issues). Groups this is likely to matter for:

- People with migraine, for whom sudden bright light is a known trigger.
- People with photophobia (light sensitivity), including after concussion or eye surgery.
- People with low vision who rely on high screen brightness for text, and so cannot simply dim the screen to tame HDR spikes.
- Anyone reading in a dark room at night.

## Proposal

Two parts that work together. Shipping both is the strongest option.

### 1. A user-level override (protects everyone, no developer work)

A browser or OS setting that forces `dynamic-range-limit` to `standard` or `constrained` with user-level `!important` priority, so author styles cannot undo it. This is the direction the spec editor suggested in #11790 ("always treat `no-limit` as `constrained-high`").

### 2. A media feature (lets developers adapt their design)

Same model as `prefers-reduced-motion`: the system setting handles rendering, and the media feature lets developers redesign for the people who asked. Two candidate names are offered on purpose, to invite the naming debate rather than fight it.

**Option A: levels**

```css
@media (prefers-dynamic-range: standard) {
  img, video { dynamic-range-limit: standard; }
  .hero-video { display: none; }   /* show the SDR poster instead */
}
```

Values: `standard | constrained | no-preference`.

- Maps directly onto `dynamic-range-limit`'s three values, and onto Apple's three rendering styles (standard, constrained HDR, full HDR).
- Allows a middle level ("a hint of HDR"). Tested in the demo: at about 50% screen brightness, `constrained` and `standard` look clearly different in Chrome, so the middle level is real and useful.
- Risk: easy to confuse with the hardware query. `dynamic-range: standard` and `prefers-dynamic-range: standard` differ by one word, and the wrong one fails silently.
- Precedent: `prefers-contrast: more | less | custom | no-preference`.

**Option B: on/off**

```css
@media (prefers-reduced-peak-brightness: reduce) {
  img, video { dynamic-range-limit: standard; }
}
```

Values: `reduce | no-preference`. Alternative name: `prefers-reduced-highlights`.

- Says exactly what gets limited, and reads immediately as an accessibility preference.
- Cannot be mixed up with the hardware query.
- Risk: unfamiliar terms, no middle level, and the word "brightness" may invite the "OS dimming is enough" reply.
- Precedent: `prefers-reduced-motion`, `prefers-reduced-transparency`, `prefers-reduced-data`.

## A precedent: loudness normalization in audio

Audio solved the same problem: one segment much louder than everything around it.

| | Audio | HDR screens |
|---|---|---|
| Measure | LUFS (ITU-R BS.1770), perceived loudness relative to the programme | Nits and HDR headroom. No agreed measure of a perceived brightness jump. |
| Content standard | EBU R128, ATSC A/85 | ITU-R BT.2408 (graphics white at 203 nits), mostly ignored outside broadcast |
| Law | US CALM Act (2010): ads may not be louder than the programme | None |
| Playback normalization | Spotify, Apple Music and YouTube adjust every track | None. Browsers render HDR as the file asks. |
| User preference | Spotify: Loud, Normal, Quiet, or off | None |

Lessons:

1. Normalize by default and let the user choose the level. Spotify's three levels match `standard` / `constrained` / `no-limit`.
2. Measure the relative jump, not the absolute level.
3. A content standard alone does not work. Enforcement has to happen at playback, which on the web means the browser.
4. Turning down the master volume never fixed loud ads. Normalizing the loud segment did. Reduce White Point is the master volume.

## Objections and answers

**"OS-level dimming is enough."** (Raised in #11790.) For pure rendering, a browser or OS cap is the best protection, which is why part 1 of this proposal exists. But dimming alone falls short:

- It dims everything. The jump is a ratio, so an HDR logo stays several times brighter than the page, while the text gets harder to read.
- It is not everywhere. Reduce White Point is on iPhone and iPad. Mac has no equivalent, Android has a beta toggle, and Windows has only on/off.
- An OS cannot change design decisions. Only the developer can swap an autoplaying HDR video for a still image or pick an SDR logo, and only if they know what the user wants.
- Precedent: iOS and macOS already had a system Reduce Motion setting, and `prefers-reduced-motion` was still added, because the system can slow animations down but only the developer can redesign them.

**"We already have `dynamic-range`."** That query describes the hardware. It is true on an HDR laptop whether the person loves HDR or gets a headache from it. Motion follows the same split: the hardware decides whether animation is possible, and `prefers-reduced-motion` says whether the person wants it.

**Fingerprinting.** Every `prefers-*` feature adds a bit of entropy. `prefers-reduced-motion`, `prefers-contrast` and `prefers-reduced-transparency` all shipped anyway. If the feature only matches on HDR-capable screens (open question 4), it adds nothing on SDR screens.

## Demo

[Live demo](demo/): a recreated inbox with the real logo, a toolbar to switch between "Today (no limit)", "Constrained" and "If my preference were respected", and a side-by-side tile of the same pixels as normal white, full HDR and capped. Needs an HDR screen and Chrome, Edge 136+ or Safari 26+. Screen recordings do not capture the difference.

## Related work

- [csswg-drafts #11790](https://github.com/w3c/csswg-drafts/issues/11790): how a user-level luminance limit fits the cascade. Open, no resolution.
- [csswg-drafts #8651](https://github.com/w3c/csswg-drafts/issues/8651): `prefers-reduced-strobing`, a stalled photosensitivity preference.
- [WCAG 2.3.1](https://www.w3.org/WAI/WCAG22/Understanding/three-flashes-or-below-threshold.html): flash thresholds are defined on sRGB relative luminance, where 1.0 is SDR white. No guidance exists for content that goes far above that. A separate gap worth raising with WCAG 3.

## Open questions

1. Option A or Option B, or another name?
2. Should the override default to `standard` or `constrained`?
3. Should video be in scope from the start, or images first?
4. Should the media feature match only on HDR-capable screens?

## Sources

Specs and discussions
- [Media Queries Level 5](https://www.w3.org/TR/mediaqueries-5/)
- [CSS Color HDR Module Level 1](https://drafts.csswg.org/css-color-hdr/), [Accessibility Considerations](https://www.w3.org/TR/css-color-hdr-1/#a11y)
- csswg-drafts issues: [#11790](https://github.com/w3c/csswg-drafts/issues/11790), [#9074](https://github.com/w3c/csswg-drafts/issues/9074), [#10271](https://github.com/w3c/csswg-drafts/issues/10271), [#11711](https://github.com/w3c/csswg-drafts/issues/11711), [#11698](https://github.com/w3c/csswg-drafts/issues/11698), [#8651](https://github.com/w3c/csswg-drafts/issues/8651), [#6757](https://github.com/w3c/csswg-drafts/issues/6757)
- [HDR headroom limit explainer (Chromium)](https://github.com/ccameron-chromium/hdr-headroom-limit/blob/main/EXPLAINER.md), [blink-dev intent thread](https://groups.google.com/a/chromium.org/g/blink-dev/c/OpUsOWnnN6c)
- [WCAG 2.3.1 Three Flashes or Below Threshold](https://www.w3.org/WAI/WCAG22/Understanding/three-flashes-or-below-threshold.html)

Browser support and usage
- [MDN: dynamic-range-limit](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/dynamic-range-limit), [Can I use](https://caniuse.com/mdn-css_properties_dynamic-range-limit), [Chrome 136 beta](https://developer.chrome.com/blog/chrome-136-beta)
- Chrome Platform Status: [CSS popularity](https://chromestatus.com/data/csspopularity), [feature popularity](https://chromestatus.com/data/featurepopularity)

Native platforms
- Apple: [Supporting HDR images in your app](https://developer.apple.com/documentation/uikit/supporting-hdr-images-in-your-app), [NSImage.DynamicRange](https://developer.apple.com/documentation/appkit/nsimage/dynamicrange), [allowedDynamicRange(_:)](https://developer.apple.com/documentation/swiftui/view/alloweddynamicrange(_:)), [CALayer.DynamicRange](https://developer.apple.com/documentation/quartzcore/calayer/dynamicrange), [Adopting advancements in HDR image rendering](https://developer.apple.com/documentation/coregraphics/adopting-advancements-in-hdr-image-rendering)
- Android: [HDR and user interfaces](https://android-developers.googleblog.com/2025/09/hdr-and-user-interfaces.html), [ActivityInfo](https://developer.android.com/reference/android/content/pm/ActivityInfo), [Display Ultra HDR](https://developer.android.com/media/grow/ultra-hdr/display), [Android HDR settings report](https://www.androidauthority.com/android-canary-hdr-settings-3576420/), [Ultra HDR](https://en.wikipedia.org/wiki/Ultra_HDR)
- [Working with HDR content in Pixelmator Pro / Photomator](https://support.pixelmator.com/pixelmator-support/advanced-workflows/working-with-hdr-content-in-photomator-and-pixelmator-pro)

Graphics white and loudness
- ITU-R BT.2408 graphics white: [public-colorweb](https://lists.w3.org/Archives/Public/public-colorweb/2019Sep/0003.html), [public-png](https://lists.w3.org/Archives/Public/public-png/2023Dec/0058.html), [comfort levels for graphics](https://daejeonchronicles.com/2023/12/07/power-budget-comfort-levels-graphics/)
- [Spotify loudness normalization](https://support.spotify.com/artists/article/loudness-normalization/), [LKFS and the CALM Act](https://www.tvtechnology.com/opinions/lkfs-amp-the-calm-act), [ATSC A/85](https://www.forasoft.com/learn/audio-for-video/glossary/terms-audio/atsc-a85)
