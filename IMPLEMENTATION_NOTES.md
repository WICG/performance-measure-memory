# Implementation Notes

## Approach 1: process isolation of web pages
A fast implementation is possible if the browser puts a cross-origin isolated web page into a separate process.
Such an implementation can return the sizes of the relevant heaps with empty attributions.
Inter-process communication may be required if cross-origin iframes are hosted in different processes.

## Approach 2: heap segregation by JS realm
If the browser puts multiple web pages in the same process and the same heap, then it needs a way to distinguish objects belonging to different pages.
An object that is allocated by JavaScript code can be attributed to the JS realm of [the running execution context](https://www.ecma-international.org/ecma-262/10.0/index.html#running-execution-context).
Segregating objects on the heap by realm during allocation provides a way to tell objects of different pages apart and additionally enables fine-grained per-frame attribution.
This comes at the cost of a more complex allocator and heap organisation with a separate space/partition for each realm.
Note that attribution is not precise for objects shared between multiple realms.
Another subtlety is that the realm that keeps an object alive (i.e. retains the object) is not necessarily the same realm that allocated the object.
Moreover, both realms may differ from [the realm of the object's constructor](https://tc39.es/ecma262/#sec-getfunctionrealm).
This is because realms of the same JavaScript agent can synchronously script with each other and can pass objects to each other.
The implementation can sidestep this problem by making attribution more coarse-grained and merging the memory usage of all realms of the same JavaScript agent.

## Approach 3: accounting during garbage collection
An alternative to heap segregation is dynamic attribution of object during garbage collection.
The following algorithm shows how to carry out per-realm memory measurement in the marking phase of a Mark-Sweep garbage collector.

Setup:

1. Assume that there is a partial function `InferRealm(object)` that uses implementation dependent heuristics to quickly compute the realm of the object or fails if that is not possible.
For example, the function could return [the realm of the object's constructor](https://tc39.es/ecma262/#sec-getfunctionrealm).
2. Let `realms` be the set of realms present on the heap at the start of garbage collection.
3. For each `realm` in `realms` create a marking worklist `worklist[realm]`.
4. Create a special marking worklist `worklist[unknown]` for shared/unattributed objects.
5. Iterate roots and push the discovered objects onto `worklist[unknown]`.

Marking worklist draining:
1. Pop an `object` from one of the non-empty worklists `worklist[realm]`, where `realm` can also be `unknown`.
2. If `InferRealm(object)` succeeds, then change `realm` to its result.
3. If `realm` is not `realms`, then it was created after the start of garbage collection and it does not have a worklist. In that case change `realm` to `unknown`.
4. Account the size of `object` to `realm`.
5. Iterate the reference in the object and push newly discovered to `worklist[realm]`.

The algorithm precisely attributes objects with known realms.
Additionally, objects that are not shared between multiple realms are accounted for precisely.
However, attribution of shared objects that do not have known realms is non-deterministic.
Shared strings and code objects are likely to be affected, so it might be worthwhile to add step 3a for such objects:

3.a. if `object` can be shared between multiple realms, then change `realm` to `unknown`.

The algorithm [was implemented](https://bugs.chromium.org/p/chromium/issues/detail?id=973627) in Chrome/V8 and it adds 10%-20% overhead to garbage collection.
The overhead applies only if there is a pending memroy measurement request.
