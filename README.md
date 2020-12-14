# performance.measureMemory API

[Draft specification](https://wicg.github.io/performance-measure-memory/)

[Origin trial and how it differs from the specification](ORIGIN_TRIAL.md)

## tl;dr
We propose a new `peformance.measureMemory` API that estimates memory usage of a web page including all its iframes and workers. The API is available only for [cross-origin isolated](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/crossOriginIsolated) web pages that opt in using [the COOP+COEP headers](https://docs.google.com/document/d/1zDlfvfTJ_9e8Jdc8ehuV4zMEu9ySMCiTGMS9y0GU92k/edit) preventing cross-origin information leaks.

Example:
```JavaScript
async function run() {
  const result = await performance.measureMemory();
  console.log(result);
}
run();
// Console output:
{
  bytes: 2300000,
  breakdown: [
    {
      bytes: 1000000,
      attribution: [
        {
          url: "https://example.com",
          scope: "Window",
        },
      ],
      userAgentSpecificTypes: ["JS", "DOM"],
    },
    {
      bytes: 500000,
      attribution: [
        {
          url: "https://example.com/iframe.html"
          container: {
            id: "example-id",
            src: "redirect.html?target=iframe.html",
          },
          scope: "Window",
        }
      ],
      userAgentSpecificTypes: ["DOM", "JS"],
    },
    {
      bytes: 800000,
      attribution: [
        {
          url: "https://example.com/worker.js",
          scope: "DedicatedWorkerGlobalScope",
        },
      ],
      userAgentSpecificTypes: ["JS"],
    },
    {
      bytes: 0,
      attribution: [],
      userAgentSpecificTypes: [],
    },
  ],
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
- The API provides breakdown of the result by type and owner with implementation-specific granularity.
- The API has no overhead when it is not used.
- The API is defined based on the standard concepts from HTML and ECMAScript specifications and does not assume any particular process model.

### Non-Goals
- Precise memory measurement. Implementations are allowed to return an estimate because computing the precise result may be expensive.
- Complete memory measurement. Implementations are free to account as many memory types (JS, DOM, CSS) as possible, but they are not required to account all memory types. Memory types that cannot be isolated per page can be omitted.
- Comparing memory usage across browsers. The API results are browser implementation specific.
- Synchronous memory measurement before and after a specific action. The API has an asynchronous interface to allow folding the measurement into garbage collection and perform necessary interprocess communication. It may take seconds or minutes until the result is available.

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
Besides that, our API provides breakdown of the result by type and owner.

### Memory pressure API
There is a proposal for [memory pressure API](https://github.com/WICG/memory-pressure/blob/master/explainer.md) that notifies the application about system memory pressure events.
This gives the application an opportunity to change its behavior at runtime to reduce its memory usage if possible e.g. by freeing up caches and unused resources.
Our proposal has different [use cases](https://docs.google.com/document/d/1u21oa3-R1FhHgrPsh8-mpb8dIFVj60wcFiM5FFrfIQA/edit#heading=h.6si74uwp7sq8) such as collecting telemetry data and detecting regressions.
Thus the two proposals are orthogonal.

## API Proposal
The API consists of a single asynchronous method `performance.measureMemory` that estimates memory usage of the web page and provides breakdown of the result by type and owner.

```JavaScript
async function run() {
  const result = await performance.measureMemory();
  console.log(result);
}
run();
```

### Example 1
For a simple page without iframes and workers the result might look as follows:
```JavaScript
{
  bytes: 1000000,
  breakdown: [
    {
      bytes: 1000000,
      attribution: [
        {
          url: "https://example.com",
          scope: "Window",
        },
      ],
      userAgentSpecificTypes: ["JS", "DOM"],
    },
    {
      bytes: 0,
      attribution: [],
      userAgentSpecificTypes: [],
    },
  ],
}
```
*Note: the format of the result has changed recently and the origin trail running in Chrome 82-87 uses the old format. See [Origin Trial](ORIGIN_TRIAL.md) for details.*
The entry with `bytes: 0` is present in the breakdown list to encourage processing of the result in a generic way without hardcoding specific entries.
Such an entry is inserted at a random position if the list is not empty.

Providing only the total memory usage is also a valid implementation:
```JavaScript
{
  bytes: 1000000,
  breakdown: [],
}
```

Similarly, the implementation might return empty `attribution: []` and/or empty `userAgentSpecificTypes: []`.

The top-level `bytes` field contains the total estimate of the web page's memory usage.
Each entry of the `breakdown` array describes some portion of the memory and attributes it to a set of windows and workers.
The entries are disjoint and their sizes sum up to the total `bytes`.

The `userAgentSpecificTypes` field lists memory types associated with the memory portion.
As the name suggests each memory type is entirely implementation specific.
In other words, memory types are not comparable across different browsers and may even change between different versions of the same browser.
The order of memory types in the list is not significant and also depends on the implementation.
An implementation may (but is not required to) use the following as a memory type:

- the name of a JavaScript object type or constructor: `Function`, `SharedArrayBuffer`, `String`, etc.
- the name of a WebIDL interface: `HTMLCanvasElement`, `HTMLElement`, etc.
- implementation specific names: `JS`, `DOM`, `GPU`, `Detached`, `Code`, etc.

### Example 2
For a page that embeds a same-origin iframe the result might attribute some memory to that iframe and provide diagnostic information for identifying the iframe.

```HTML
<html>
  <body>
    <iframe id="example-id"
            src="redirect.html?target=iframe.html">
    </iframe>
  </body>
</html>
```

```JavaScript
{
  bytes: 1500000,
  breakdown: [
    {
      bytes: 0,
      attribution: [],
      userAgentSpecificTypes: [],
    },
    {
      bytes: 1000000,
      attribution: [
        {
          url: "https://example.com",
          scope: "Window",
        },
      ],
      userAgentSpecificTypes: ["JS", "DOM"],
    },
    {
      bytes: 500000,
      attribution: [
        {
          url: "https://example.com/iframe.html"
          container: {
            id: "example-id",
            src: "redirect.html?target=iframe.html",
          },
          scope: "Window",
        }
      ],
      userAgentSpecificTypes: ["DOM", "JS"],
    },
  ],
}
```

Note how the `url` and `container.src` fields differ for the iframe. The former reflects the current location.href of the iframe whereas the latter is the value of the src attribute of the iframe element.

It is not always possible to separate iframe memory from page memory in a meaningful way. An implementation is allowed to lump together some or all of iframe and page memory:

```JavaScript
{
  bytes: 1500000,
  breakdown: [
    {
      bytes: 0,
      attribution: [],
      userAgentSpecificTypes: [],
    },
    {
      bytes: 1500000,
      attribution: [
        {
          url: "https://example.com",
          scope: "Window",
        },
        {
          url: "https://example.com/iframe.html",
          container: {
            id: "example-id",
            src: "redirect.html?target=iframe.html",
          },
          scope: "Window",
        },
      ],
      userAgentSpecificTypes: ["JS", "DOM"],
    },
  ],
};
```

### Example 3
For a page that spawns a web worker the result includes the URL of the worker.
```JavaScript
{
  bytes: 1800000,
  breakdown: [
    {
      bytes: 1000000,
      attribution: [
        {
          url: "https://example.com",
          scope: "Window",
        },
      ],
      userAgentSpecificTypes: ["DOM", "JS"],
    },
    {
      bytes: 0,
      attribution: [],
      userAgentSpecificTypes: [],
    },
    {
      bytes: 800000,
      attribution: [
        {
          url: "https://example.com/worker.js",
          scope: "DedicatedWorkerGlobalScope",
        },
      ],
      userAgentSpecificTypes: ["JS"],
    },
  ],
};
```
An implementation might lump together worker and page memory. If a worker is spawned by an iframe, then the worker’s attribution entry has a container field corresponding to the iframe element.

Memory of shared and service workers is not included in the result.

### Example 4
To get the memory usage of a shared/service worker, the performance.measureMemory() function needs to be invoked in the context of that worker. The result could be something like:
```JavaScript
{
  bytes: 1000000,
  breakdown: [
    {
      bytes: 1000000,
      attribution: [
        {
          url: "https://example.com/service-worker.js",
          scope: "ServiceWorkerGlobalScope",
        },
      ],
      userAgentSpecificTypes: ["JS"],
    },
    {
      bytes: 0,
      attribution: [],
      userAgentSpecificTypes: [],
    },
  ],
}
```

### Example 5
If a page embeds a cross-origin iframe, then the URL of that iframe is not revealed to avoid information leaks. Only the container element (which is already known to the page) appears in the result. Additionally, if the cross-origin iframe embeds other cross-origin iframes and/or spawns workers, then all their memory is aggregated and attributed to the top-most cross-origin iframe.

Consider a page with the following structure:
```
example.com (1000000 bytes)
  |
  *--foo.com/iframe1 (500000 bytes)
       |
       *--foo.com/iframe2 (200000 bytes)
       |
       *--bar.com/iframe2 (300000 bytes)
       |
       *--foo.com/worker.js (400000 bytes)
```

A cross-origin iframe embeds to other iframes and spawns a worker. All memory of these resources is attributed to the first iframe.
```HTML
<html>
  <body>
    <iframe id="example-id"
            src="https://foo.com/iframe1">
    </iframe>
  </body>
</html>
```

```JavaScript
{
  bytes: 2400000,
  breakdown: [
    {
      bytes: 0,
      attribution: [],
      userAgentSpecificTypes: [],
    },
    {
      bytes: 1000000,
      attribution: [
        {
          url: "https://example.com",
          scope: "Window",
        },
      ],
      userAgentSpecificTypes: ["DOM", "JS"],
    },
    {
      bytes: 1400000,
      attribution: [
        {
          url: "cross-origin-url",
          container: {
            id: "example-id",
            src: "https://foo.com/iframe1",
          },
          scope: "cross-origin-aggregated",
        },
      ],
      userAgentSpecificTypes: ["JS", "DOM"],
    },
  ],
}
```

Note that the `url` and `scope` fields of the cross-origin iframe entry have special values indicating that information is not available.

### Example 6
If a cross-origin iframe embeds an iframe of the same origin as the main page, then the same-origin iframe is revealed in the result. Note that there is no information leak because the main page can find and read location.href of the same-origin iframe.

```
example.com (1000000 bytes)
  |
  *--foo.com/iframe1 (500000 bytes)
       |
       *--example.com/iframe2 (200000 bytes)
```

```HTML
<html>
    <body>
      <iframe id="example-id"
              src="https://foo.com/iframe1">
      </iframe>
    </body>
  </html>
```

```JavaScript
{
  bytes: 1700000,
  breakdown: [
    {
      bytes: 1000000,
      attribution: [
        {
          url: "https://example.com",
          scope: "Window",
        },
      ],
      userAgentSpecificTypes: ["JS", "DOM"],
    },
    {
      bytes: 0,
      attribution: [],
      userAgentSpecificTypes: [],
    },
    {
      bytes: 500000,
      attribution: [
        {
          url: "cross-origin-url",
          container: {
            id: "example-id",
            src: "https://foo.com/iframe1",
          },
          scope: "cross-origin-aggregated",
        },
      ],
      userAgentSpecificTypes: ["DOM", "JS"],
    },
    {
      bytes: 200000,
      attribution: [
        {
          url: "https://example.com/iframe2",
          container: {
            id: "example-id",
            src: "https://foo.com/iframe1",
          },
          scope: "Window",
        },
      ],
      userAgentSpecificTypes: ["JS", "DOM"],
    },
  ],
}
```

### Alternatives
Adding a `userAgentSpecific` prefix to the `bytes` and `attribution` fields would emphasize that the API result is implementation dependent:
```JavaScript
  {userAgentSpecificBytes: 1000000, ...},
```
Our preference however is to communicate the message in documentation and keep the API concise and consistent with other Performance APIs.

### Scope
The API is available for windows, shared workers, and service workers.
When invoked in a window context, the API estimates memory usage of all [JS agent clusters](https://html.spec.whatwg.org/multipage/webappapis.html#integration-with-the-javascript-agent-cluster-formalism) of the [browsing context group](https://html.spec.whatwg.org/multipage/browsers.html#browsing-context-group) of the window.
(The scope will be restricted to the current address space for security. See Issues #5 and #20.)
Note that the result accounts for all iframes and nested dedicated workers.
In the shared/service worker case, the API estimates memory usage of the worker's JS agent cluster that includes all nested dedicated workers.

The API is not available in the context of a dedicated worker.
There are two reasons for this design choice.
First, the intended use of the API is to have a global memory monitor for a web page that performs statistical sampling of memory usage and sends the samples to the server for A/B testing and regression detection.
Since the top-level context (a window or a shared/service worker) already measures all nested dedicated workers, invoking the API in a dedicated worker is likely a misuse of the API.
The second reason is to simplify implementation of the API by ensuring that at most one thread in a JS agent cluster can request memory measurement for the cluster.

## Security Considerations
### Cross-origin information leaks
The URLs and other string values that appear in the result are guaranteed to be known to the origin that invokes the API.

The only information that is exposed cross-origin is the size information provided in the `bytes` fields.
The API relies on the [cross-origin isolation](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/crossOriginIsolated) mechanism to mitigate cross-origin size information leaks.
Specifically, the API relies on the invariant that all loaded resources have opted in to be embeddable and legible by their embedding origin.

### Fingerprinting
The result of the API depends only on the objects allocated by the web page itself and does not include unrelated memory such as the baseline memory usage of an empty web page.
This means the same user agent binary running on two different devices should produce the same results for a fixed web page.

A web page can infer the following information about the user agent:
- the bitness of the user agent (32-bit vs 64-bit).
- the version of the user agent to some extent.

Similar information can be obtained from the existing APIs (`navigator.userAgent`, `navigator.platform`).
The bitness of the user agent can also be inferred by measuring the runtime of 32-bit and 64-bit operations.

## Performance Considerations
The API with coarse-grained breakdown can be implemented efficiently provided that the browser does not collocate a cross-origin web page with other web pages in the same process.
In such a case the API can return the sizes of the heaps (JS, DOM, CSS, worker, etc) with empty attributions.

Fine-grained attribution requires more expensive computation because objects from different frames may be allocated on the same heap.
One possible implementation is to segregate objects by frame during allocation.
Alternative implementation is to infer the object's frame while traversing the object graph during garbage collection.
This was implemented in Chrome/V8 and introduces 10%-20% overhead to garbage collection (only if there is a pending memory measurement request).
See [Implementation Notes](IMPLEMENTATION_NOTES.md) for more details.

## API Usage
The API is intended for A/B testing, regressions detection, and general analysis of aggregate memory usage data from production.
The results of individual calls are less useful because they are sensitive to the timing of web page events, user actions, and garbage collection.
We recommend calling the API periodically every `N` minutes.
Even better would be to use statistical sampling such as Poisson sampling to avoid the bias of a fixed sampling interval as shown below.

```JavaScript
// Starts statistical sampling of the memory usage.
function scheduleMeasurement() {
  if (!performance.measureMemory) {
    console.log('performance.measureMemory is not available.');
    return;
  }
  let interval = measurementInterval();
  console.log('Scheduling memory measurement in ' +
      `${Math.round(interval / 1000)} seconds.`);
  setTimeout(performMeasurement, interval);
}

async function performMeasurement() {
  // 1. Invoke performance.measureMemory().
  let result;
  try {
    result = await performance.measureMemory();
  } catch (error) {
    if (error instanceof DOMException &&
        error.name === 'SecurityError') {
      console.log(`Cannot measure memory: ${error.message}.`);
      return;
    }
    throw error;
  }
  // 2. Record the result.
  console.log(`Memory usage: ${result.bytes} bytes`);
  console.log('Memory breakdown: ', result.breakdown);
  // 3. Schedule the next measurement.
  scheduleMeasurement();
}

// Returns a random interval in milliseconds that is
// sampled with a Poisson process. It ensures that on
// average there is one measurement every five minutes.
function measurementInterval() {
  const MEAN_INTERVAL_IN_MS = 5 * 60 * 1000;
  return -Math.log(Math.random()) * MEAN_INTERVAL_IN_MS;
}
```

## See Also

Links for [the previous version](https://github.com/ulan/javascript-agent-memory/blob/master/explainer.md) of the API that did not rely on COOP+COEP for security:
- [WebPerf WG F2F June 2019 presentation](https://docs.google.com/document/d/1uQ7pXwuBv-1jitYou7TALJxV0tllXLxTyEjA2n1mSzY/edit#heading=h.x0ih8vnl6qwo)
- [TAG design review](https://github.com/w3ctag/design-reviews/issues/386)
- [WICG discussion thread](https://discourse.wicg.io/t/proposal-javascript-agent-memory-api/3586)
