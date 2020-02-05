# performance.measureMemory API

Last updated: 2020-02-05

## tl;dr
We propose a new `peformance.measureMemory` API that estimates memory usage of a web page including all its iframes and workers. The API is available only for [cross-origin isolated](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/crossOriginIsolated) web pages that opt in using [the COOP+COEP headers](https://docs.google.com/document/d/1zDlfvfTJ_9e8Jdc8ehuV4zMEu9ySMCiTGMS9y0GU92k/edit) preventing cross-origin information leaks.

Example:
```JavaScript
const result = await performance.measureMemory();
console.log(result);
// Console output:
{
  bytes: 100*MB,
  breakdown: [
    {bytes: 40*MB, globals: 2, type: 'js/window', origins: ['foo.com']},
    {bytes: 30*MB, globals: 1, type: 'js/window', origins: ['bar.com']},
    {bytes: 20*MB, globals: 1, type: 'js/worker', origins: ['foo.com']},
    {bytes: 10*MB, globals: 3, type: 'dom', origins: ['foo.com', 'bar.com']},
  ]
}
```

## Problem
As shown in [this collection of use cases](https://docs.google.com/document/d/1u21oa3-R1FhHgrPsh8-mpb8dIFVj60wcFiM5FFrfIQA/edit#heading=h.6si74uwp7sq8) there is a need for an API that measures memory footprint of web pages in production.
The use cases include a) analysis of correlation between memory usage and user metrics, b) detection of memory regressions, c) evaluation of feature launches in A/B tests, d) memory optimization.

Currently developers resort to the non-standard `performance.memory` API that [is used in 20%](https://www.chromestatus.com/metrics/feature/timeline/popularity/884) of page loads in Chrome.
The API reports the size of the JS heap and thus has two major drawbacks:
1) it may greatly _overestimate_ the actual memory usage if other large web pages share the same heap.
2) it may greatly _underestimate_ the actual memory usage if the web page spawns workers and/or embeds cross-site iframes that are allocated in separate heaps.

In this proposal we aim to fix these drawbacks and set the following requirements for the new API.

### Requirements
- The API does not leak cross-origin information.
- The API measures memory usage of the web page including all its iframes and workers.
- The API accounts only the objects allocated by the web page. Other web pages do not affect the result.
- The API provides breakdown of the result by type and origin with implementation-specific granularity.

### Non-Goals
- Precise memory measurement. Implementations are allowed to return an estimate because computing the precise result may be computationally expensive.
- Complete memory measurement. Implementations are free to account as many memory types (JS, DOM, CSS) as possible, but they are not required to account all memory types. Memory types that cannot be isolated per page can be omitted.
- Comparing memory usage across browsers. The API results are browser implementation specific.
- Synchronous memory measurement before and after a specific action. The API has asynchronous interface to allow folding the measurement into garbage collection and perform necessary interprocess communication. It may take seconds or minutes until the result is available.

## Related Work
### JavaScript agent memory API
The current proposal is generalization and extension of the previous [JavaScript agent memory API proposal](https://github.com/ulan/javascript-agent-memory/blob/master/explainer.md)  that was presented at [WebPerf WG F2F June 2019](https://docs.google.com/document/d/12ANc7fbKpjs__Qw_0DxM74u49276vTwRwCPyBxUkfBw/edit#heading=h.nraz045xllk0) meeting.

The main difference is that the current proposal relies on COOP+COEP for security.
This allows us to increase the scope of the API beyond a single JavaScript agent to cover cross-origin iframes, workers as well as other kinds of memory.

### Process memory API
There was a proposal for [process memory API](https://github.com/WICG/performance-memory/blob/master/explainer.md) that estimates memory footprint of the site and accounts different types of memory: JS, DOM, CSS, Web Workers spawned by the page, etc.
The proposal [was initially blocked](https://github.com/mozilla/standards-positions/issues/85#issuecomment-426382208) by information leak of opaque cross-origin resources and is currently abandoned.

Our proposal combines this process memory API with the JS agent memory API.
The main difference between our proposal and the process memory API is that the scope of the process memory API is limited to the current [JS agent cluster](https://html.spec.whatwg.org/multipage/webappapis.html#integration-with-the-javascript-agent-cluster-formalism) (The proposal uses the old term -- "related similar-origin browsing contexts").
This means that the API measures only same-site memory and does not measure cross-site iframes.
In contrast to that, our API measures all JS agent clusters of the current [browsing context group](https://html.spec.whatwg.org/multipage/browsers.html#browsing-context-group).
Besides that, our API provides breakdown of the result by type and origin.

### Memory pressure API
There is a proposal for [memory pressure API](https://github.com/WICG/memory-pressure/blob/master/explainer.md) that notifies the application about system memory pressure events.
This gives the application an opportunity to change its behavior at runtime to reduce its memory usage if possible e.g. by freeing up caches and unused resources.
Our proposal has different [use cases](https://docs.google.com/document/d/1u21oa3-R1FhHgrPsh8-mpb8dIFVj60wcFiM5FFrfIQA/edit#heading=h.6si74uwp7sq8) such as collecting telemetry data and detecting regressions.
Thus the two proposals are orthogonal.

## API Proposal
The API consists of a single asynchronous method `performance.measureMemory` that estimates memory usage of the web page and provides breakdown of the result by type and origin.

```JavaScript
const result = await performance.measureMemory();
console.log(result);
// Console output:
{
  bytes: 100*MB,
  breakdown: [
    {bytes: 40*MB, globals: 2, type: 'js/window', origins: ['foo.com']},
    {bytes: 30*MB, globals: 1, type: 'js/window', origins: ['bar.com']},
    {bytes: 20*MB, globals: 1, type: 'js/worker', origins: ['foo.com']},
    {bytes: 10*MB, globals: 3, type: 'dom', origins: ['foo.com', 'bar.com']},
  ]
}
```
The `bytes` field contains the total estimate of web page's memory usage.

Each entry of the `breakdown` array describes some portion of the memory.
It lists all `origins` that use the memory portion.
We expect implementations to differ in the granularity of origin attribution.
Some implementations will provide accurate per-origin attribution so that the `origins` lists are singletons.
Implementations that do not support per-origin attribution are allowed to list all origins of the web page.

Additionally we consider the number of `globals` (i.e. JS realms or frames) to be a useful signal for web pages
that create iframes dynamically and want to estimate average iframe sizes.

The `type` field contains implementation specific description of the memory portion.
Alternative design would be to have separate `types` and `context` fields:
```JavaScript
  {bytes: 40*MB, globals: 2, types: ['js'], context:'window', origins: ['foo.com']},

```

## Security Considerations
### Cross-origin information leak
Only [cross-origin isolated](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/crossOriginIsolated) web pages that opt in using [the COOP+COEP headers](https://docs.google.com/document/d/1zDlfvfTJ_9e8Jdc8ehuV4zMEu9ySMCiTGMS9y0GU92k/edit) can use the API.
This prevents cross-origin information leaks because all iframes and resources have to explicitly opt in using CORP/CORS.
Additionally it is guaranteed that a cross-origin isolated web page is not colocated with web page in the same address space.

An attempt to invoke the API when
`crossOriginIsolated === false` leads to promise rejection with a `SecurityError` DOM exception.
```JavaScript
try {
  const result = await performance.measureMemory();
} catch (error) {
  console.assert(!crossOriginIsolated);
  console.assert(error instanceof DOMException);
  console.assert(error.name === 'SecurityError');
  console.assert(error.message === 'crossOriginIsolated is required');
}
```

### Fingerprinting
Implementations need to be careful to avoid leaking information about the underlying browser instance.
Ideally the API returns `0` (or some other constant) for empty web pages such that the result depends only on the objects allocated by the web page.

Some information leak is unavoidable though. The web page can deduce:
- the bitness of the system (32-bit vs 64-bit).
- the version of the browser to some extent.

Note that such information can be obtained from the existing APIs (`navigator.userAgent`, `navigator.deviceMemory`).

## Performance Considerations
The API with coarse-grained breakdown can be implemented efficiently provided that the browser does not collocate a  cross-origin web page with other web pages.
In such case the API can return the sizes of the heaps (JS, DOM, CSS, worker, etc) and list origins on each heap.

Fine-grained per-origin breakdown requires more expensive computation because objects from different origins may be allocated on the same heap.
One possible implementation is to infer the object's origin while traversing the object graph during garbage collection.
This was implemented in Chrome/V8 and introduces 10%-20% overhead to garbage collection.
An alternative implementation is to segregate objects by origin during allocation.

## API Usage
The API is well-suited for A/B testing, regressions detection, and general analysis of aggregate memory usage data from production.
Results of individual API calls are less useful because they are sensitive to timing w.r.t web page and user actions.
We recommend calling the API periodically every `N` minutes.
Even better would be to use statistical sampling such as Poisson sampling to avoid the bias of a fixed sampling interval.

Note that due to [COOP](https://docs.google.com/document/d/1u21oa3-R1FhHgrPsh8-mpb8dIFVj60wcFiM5FFrfIQA/edit#heading=h.6si74uwp7sq8) restriction only the main frame of the web page can call the API.
(The `crossOriginIsolated` flag is always `false` for iframes.)

## See Also

Links for [the previous version](https://github.com/ulan/javascript-agent-memory/blob/master/explainer.md) of the API that did not rely on COOP+COEP for security:
- [WebPerf WG F2F June 2019 presentation](https://docs.google.com/document/d/1uQ7pXwuBv-1jitYou7TALJxV0tllXLxTyEjA2n1mSzY/edit#heading=h.x0ih8vnl6qwo)
- [TAG design review](https://github.com/w3ctag/design-reviews/issues/386)
- [WICG discussion thread](https://discourse.wicg.io/t/proposal-javascript-agent-memory-api/3586)
