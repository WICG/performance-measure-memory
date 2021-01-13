# Origin Trial in Chrome

The `performance.measureUserAgentSpecificMemory()` API is available as [an origin trial](https://developers.chrome.com/origintrials/#/view_trial/1281274093986906113) in Chrome 82 to 87.
The origin trial ends on January 13, 2021.
Follow [these instructions](https://web.dev/monitor-total-page-memory-usage/#enabling-support-during-the-origin-trial-phase) to register for the origin trial.

Since the specification of the API is evolving and not finalized yet, the origin trial implementation differs from the specification in some aspects.

## Result differences
The `attribution` field of the result has evolved from a simple URL string to an object with the `url`, `container`, and `scope` fields.

The origin trial implementation uses the old format:
```
attribution: ['https://example.com/iframe.html']
```

The new format defined in the specification provides more information:
```
attribution: [
  {
    url: "https://example.com/iframe.html"
    container: {
      id: "example-id",
      src: "redirect.html?target=iframe.html",
    },
    scope: "Window",
  }
]
```

Chrome 89 will start using the new format.

## Security differences
The origin trial implementation relies on [Site Isolation](https://developers.google.com/web/updates/2018/07/site-isolation) for security whereas the explainer requires [cross-origin isolation](https://developers.google.com/web/updates/2018/07/site-isolation).
In practice this means that the API will likely be available on desktop Chrome than on mobile Chrome because Site Isolation is enabled by default on desktop Chrome.

Chrome 89 will be gated behind cross-origin isolation.

## Scope differences
The implementation measures only JavaScript memory of the main window and all **same-site** iframes and related windows.
(`foo.com`, `a.foo.com`, `b.foo.com` are same-site; `foo.com` and `bar.com` are not same-site).
The API ignores the memory usage of non-JavaScript objects.

Chrome 87 additionally measures memory usage of dedicated workers, but does not provide their URLs.
Chrome 89 will add support for cross-site iframes and will provide URLs for workers.

## Other caveats
The implementation may take up to 20 seconds to complete the memory measurement.
This is because the implementation waits for the next garbage collection to perform the measurement.
The API forces a garbage collection after 20 seconds.
Starting Chrome with the `--enable-blink-features='ForceEagerMeasureMemory'` command-line flag reduces the timeout to zero and is useful for local debugging and testing.