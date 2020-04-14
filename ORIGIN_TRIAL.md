# Origin Trial in Chrome

The `performance.measureMemory()` API is available as an origin trial starting from Chrome 83.
(It is also available in Chrome 82, but the release of Chrome 82 [was skipped](https://chromereleases.googleblog.com/2020/03/chrome-and-chrome-os-release-updates.html).)
Follow [these instructions](https://web.dev/monitor-total-page-memory-usage/#enabling-support-during-the-origin-trial-phase) to register for the origin trial.

## How Chrome 83 implementation differs from the explainer?

Chrome 83 implementation relies on [Site Isolation](https://developers.google.com/web/updates/2018/07/site-isolation) for security whereas the explainer requires [cross-origin isolation](https://developers.google.com/web/updates/2018/07/site-isolation).
In practice this means that the API will likely be available on desktop Chrome than on mobile Chrome.
(Site Isolate is enabled by default on desktop Chrome).

Chrome 83 implementation measures only JavaScript memory of the main window and all **same-site** iframes and related windows.
(`foo.com`, `a.foo.com`, `b.foo.com` are same-site; `foo.com` and `bar.com` are not same-site).
The API ignores the memory usage of workers and non-JavaScript objects.

Chrome 83 implementation may take up to 20 seconds to complete the memory measurement.
This is because the implementation waits for the next garbage collection to perform the measurement.
The API forces a garbage collection after 20 seconds.
Starting Chrome with the `--enable-blink-features='ForceEagerMeasureMemory'` command-line flag reduces the timeout to zero and is useful for local debugging and testing.