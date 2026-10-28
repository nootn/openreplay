# OpenReplay Tracker

The main package of the [OpenReplay](https://openreplay.com/) tracker.

## Development & Contribution

Please use [bun](https://bun.sh/) to install and build this library. Any submitted pull request must pass **all tests** and should have positive test coverage diff %.

## Documentation

For launch options and available public methods, [refer to the documentation](https://docs.openreplay.com/en/sdk/methods)

## Installation

```bash
npm i @openreplay/tracker
```

## Usage

Initialize the package from your codebase entry point and start the tracker. You must set the `projectKey` option in the constructor. Its value can can be found in your OpenReplay dashboard under [Preferences -> Projects](https://app.openreplay.com/client/projects).

```js
import Tracker from '@openreplay/tracker'

const tracker = new Tracker({
  projectKey: YOUR_PROJECT_KEY,
})
tracker
  .start({
    userID: 'Mr.Smith',
    metadata: {
      version: '3.5.0',
      balance: '10M',
      role: 'admin',
    },
  })
  .then((startedSession) => {
    if (startedSession.success) {
      console.log(startedSession)
    }
  })
```

Then you can use OpenReplay JavaScript API anywhere in your code.

```js
tracker.setUserID('my_user_id')
tracker.setMetadata('env', 'prod')
```

## Additional capture sanitization

`captureSanitizer` is an optional synchronous filter for recorded DOM attributes,
resources, CSS and FontFace metadata. It does not change the live DOM, application
requests or native font construction. Omitting it retains the existing collection
policy, with intentional image privacy-floor fixes: Hidden/Obscured error and
mutation paths suppress original sources, placeholders clear recorded srcset, and
oversized sources stop after the placeholder rather than falling through.

```ts
import Tracker, { type CaptureSanitizer } from '@openreplay/tracker'

const resources = new Set(['https://static.example.org/ui.css'])
const styles = new Set(['.panel { display: block; }'])
const captureSanitizer: CaptureSanitizer = data => {
  switch (data.kind) {
    case 'attribute':
      if (data.name === 'class' && data.value === 'panel') return data.value
      if (data.name === 'aria-expanded' && ['true', 'false'].includes(data.value)) return data.value
      if (data.name === 'hidden' && data.value === '') return data.value
      if (data.name === 'style' && styles.has(data.value)) return data.value
      if (['href', 'src'].includes(data.name) && resources.has(data.value)) return data.value
      return null
    case 'resource':
      return resources.has(data.value) ? data.value : null
    case 'css':
      return styles.has(data.value) ? data.value : null
    case 'font':
      return null
  }
}
const tracker = new Tracker({ projectKey: 'YOUR_PROJECT_KEY', privateMode: true, captureSanitizer })
```

A string replaces the recorded value, `null` suppresses it. Throwing, returning a
Promise or any other non-string value suppresses it without logging raw data.
The callback is fixed at tracker construction and runs for each emission; there
is no verdict cache. Existing masking, truncation and blanked ordinary links run
first. Do not use the callback to reconstruct masked values from the DOM. This is
trusted application code, not an untrusted/customer-authored policy sandbox.

`CaptureData` has these semantic contexts (no generated wire tuple dependency):

- `attribute`: `name`, `value`, original `node` (or null). Checked once before the
  value can enter the string dictionary or a URL-based attribute message. A
  rejected attribute emits a removal so a prior recorded value does not linger.
- `resource`: `resourceType` (`image`, `stylesheet`, `svg`, `timing`, `base`),
  `value`, `node`. Stylesheet/SVG URLs are checked before SDK fetch/parsing; CSS
  text fetched afterward receives its own check. Image `timing` covers the
  independent image-error URL path, not the general performance timing options.
  `base` is the replay resource base URL; rejection records an empty base. Permit
  a safe static base when approved CSS needs relative URL resolution.
- `css`: `source` (`attribute`, `text`, `rule`, `adopted`), `value`, `node`. Includes
  inline/dynamic/adopted styles and replacements. Rewriting or dropping CSS can
  reduce replay fidelity; this is not a CSS parser or automatic URL scrubber.
- `font`: `field` (`family`, `source`, `descriptors`), `value`, null `node`. A font
  tuple is omitted if any field is rejected. Only approved fields are buffered;
  replay revalidates those approved values. A null or changed result suppresses the
  tuple: an approved value is never rewritten twice. Callbacks should accept their own
  safe replacements unchanged; mutable external policy can revoke a tuple on replay.
  Original constructor arguments are untouched.

A finite set of genuinely SDK-generated structural values bypass the callback:
image placeholder attribute and generated dimensions, hidden-root width/height-only
style, dialog mode, `orloaded`, resolved colour scheme. Matching DOM attribute
names/values alone do not receive this exemption. A hidden root clone may retain
width/height-only geometry (including identical caller-provided geometry); no other
style content is exempt. The placeholder still passes the image resource callback, which may replace it
with a fixed non-network data URI or reject it. Without this callback it uses
`https://static.openreplay.com/tracker/placeholder.jpeg`; its asset hosting/egress
must still fit your deployment policy. The hook is not proof that every replay
asset is locally hosted or that playback works without network access.

Cross-domain child batches/debug cannot carry this callback and are dropped with
one value-free warning when a policy is active. A separately instantiated
cross-domain child tracker with this option is refused. Same-App DOM observation
uses the normal source boundaries; this does not promise arbitrary plugin/raw
worker messages are safe. Avoid relying on internal `App.send` monkeypatches.

This option is **not** a universal privacy or lifecycle policy. Text/input values,
page URL/title/referrer, click labels/selectors, network/console/exceptions,
performance timing, identity/storage, canvas/frames and custom events have
separate existing controls. Configure and test those too. It does not implement
Stop/discard, delete backend data, certify clinical capture, or replace decoded
outbound and replay-asset tests for your actual application.


### Terminal discard (local candidate API)

`tracker.discard(): void` retires that Tracker permanently. It drops pending main-thread buffers, terminates its worker without the flushing Stop/closing messages, detaches owned collection and removes only configured SDK identity keys. A new recording requires a new Tracker. Repeated discard is harmless; identity getters and capture delegates no longer access the retired App. Plugins invoked through `use()` afterwards receive null, as for an unavailable App.

Ordinary `stop()` is unchanged and still flushes. Discard does not retract dispatched requests (keepalive batches can finish after worker termination), remove server-accepted data or prove a network-wide no-packets-after-call guarantee. Default storage is not inherently memory-only. Global instrumentation wrappers are not all uninstalled; retired instances stop collecting/emitting, including cached FontFace tuples, but wrapper depth can grow across repeated instances.

For the Clin6 synthetic controller use a fresh memory-only storage pair per instance, forceSingleTab true, debug-report endpoint null, and disable analytics, network capture, canvas, crossdomain/frames, offline/cold saving and in-memory CSS scanning. Existing caller consent/principal/policy checks remain required. This source API is not production/privacy acceptance and has not yet been exercised by the bounded runtime harness.

With analytics enabled, discard drops its pending batch and autosend/retry work; ordinary analytics Stop remains flushing. Previously persisted analytics profile properties are not erased by this API. Retired iframe listeners cannot restore session keys.
