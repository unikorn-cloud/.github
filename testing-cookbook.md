# Strategies for testing in UNI

## What makes a good test

- detects behaviour changes, not implementation changes
  - so we must be careful not to over-constrain, e.g., by relying on
    validation being done in a specific order
- guards assumptions in the implementation (e.g., the "early warning
  system" [here][reader_test.go])
- can be a reference for how an abstraction is supposed to work

## Almost-always-useful tactics

**Did the fix fix it?**

When you fix a broken behaviour, write a test that would provoke the
broken behaviour without the fix -- then undo the fix and check that
the test fails. This is doubly valuable, because it provides evidence
that you actually fixed it, _and_ it guards against it being broken
again.

**Favour verifying properties, over checking values**

By "properties", I mean things that should be true for every
input. For example, that if you call `convert(generate(input))`, you
get the input back again (see "round-tripping" below).

This gives you a lever, since you can write the verification once, and
test it many times. It can also be helpful by pushing you to
understand what _are_ the properties that should be true -- thinking
this through can clarify how the code under test should work in the
first place.

- often a property is "does not return an error" (or equally
  importantly, _does_ return an error)
- often a property is "it gets the same value back again"

**Make external requirements as narrow as possible**

This is a good principle anyway: [don't depend on methods that you
don't use][solid-isp]. In Go this usually translates to using
interfaces to specify requirements, and only including the methods
that you actually need.

For example, the image handler defines what it needs from the provider
[in its own interface][image-provider].

With respect to testing specifically, it makes it easier to supply
mocks or fakes.

## Testing handlers

The usual shape of API handlers (typically under `pkg/handlers/` in
each repo) is for the HTTP Handler to do authorisation and unmarshal
the request, then hand over to a "client" (e.g.,
uni-region/pkg/handler/network/client.go) to actuate the request.

### Generation and conversion

There's a fair bit of code in UNI that is simply transforming from one
domain into another, similar domain. Many of the API handlers take a
request, and convert it to a Kubernetes resource (often doing
validation on the way), then create or update that resource using a
Kubernetes API client; and on the way out, take a Kubernetes resource
and transform it to a response.

There's a few tactics we can use to put this code under test. Since
the conversions are typically self-contained, they don't need extra
scaffolding.

**How to get started**

 - Write a single test that checks `generate` for a minimal request
   value. Make sure you are happy that the expected value makes sense
   semantically (so you can't, for example, send a valid request that
   creates an invalid resource)
 - Turn it into a table test with one testcase
 - Create helpers and builders for making further test cases, as needed

**Round-tripping (or near round-tripping)**

Usually the request type and the response type are at least similar,
if not the same type. So you can test the property that `convertX` and
`generateX` are (near) inverses, by taking a request and calling
`convertX(generateX(request))`, and comparing the result with the
original request.

The comparison will usually be bespoke, because you'll need to do
things like:

- zero out fields that appear in the response but not in the request,
  e.g., timestamps or generated identifiers;

- compare field-by-field, when the request and response types don't quite
  line up

But you only need to write the comparison once, and you can use it in
a table test. (FIXME: I can't find an example in the UNI codebase)

**Validating validation**

These are easy to write as table tests.

- provide test cases with bad input;

- isolate each test case by using known-good values for the rest of
  the input -- a builder can work well here, or just assigning over
  fields of a fixture;

- fuzzing, or carefully chosen values, will cover off situations where
  the validation could crash (e.g., parsing IP addresses). (NB: not
  currently used in UNI)

### Verifying actuation

Especially in cases where you want to check something more than "it
creates an object". An example is the
[uni-identity/pkg/handler/groups][] tests, which verify some
consistency properties of how the custom resource's fields are
populated.

Some useful tactics in the above example:

- use a fake Kubernetes API client, and seed it with the fixture;
- if there's a lot of complicated setup, [keep fixtures values in a
  struct][fixture-struct]
- use [fake client interceptors][fake-interceptors] to test Kubernetes API error cases

### Testing sagas

Some handlers use sagas -- sequences of reversable actions. These are
a bit more difficult to test, since their purpose is to co-ordinate
calls out to dependencies.

**If you can write a test for the saga, you can very likely write a
test for the API route with not much more work.** The latter is more
valuable, because it will usually put more code under test, and it
will make fewer assumptions about the implementation.

**How to get started**

 - you may need to separate out the HTTP Handler for a set of routes,
   [uni-region/handler/images][image-handler-commit], to make it
   possible to test those routes in isolation
 - define interfaces for the methods that the handler needs to call and mock them
 - write a helper to construct the mock controller and the handler
 - you will probably need to inject things into the `context.Context` -- look for similar tests
 - you can [call the handler methods directly][image-handler-call-method] with a http.NewRequest
 - use http.NewRecorder to capture responses ([ibid][])

[uni-identity/pkg/handler/groups]:  https://github.com/nscaledev/uni-identity/blob/main/pkg/handler/groups/client_test.go
[fixture-struct]: https://github.com/nscaledev/uni-identity/blob/35b19d7600874cd4b02f5b6011b4ad969d15b0a0/pkg/handler/groups/client_test.go#L121
[image-handler-commit]: https://github.com/nscaledev/uni-region/pull/218/changes/eb81aaf81d1185865d30e6eaef1a910eb9a629b8#diff-f0fbac0e227d818487d28a9f5e9c2cd846ac02d996792bb4a39032ffff1bd3a7
[reader_test.go]: https://github.com/nscaledev/uni-region/blob/main/pkg/handler/image/reader_test.go#L27
[solid-isp]: https://en.wikipedia.org/wiki/Interface_segregation_principle
[image-handler-call-method]: https://github.com/nscaledev/uni-region/blob/fb26436b4d6823d5427e0b78d3e942c7b5512a01/pkg/handler/handler_image_test.go#L240
[ibid]: https://github.com/nscaledev/uni-region/blob/fb26436b4d6823d5427e0b78d3e942c7b5512a01/pkg/handler/handler_image_test.go#L238
[image-provider]: https://github.com/nscaledev/uni-region/blob/fb26436b4d6823d5427e0b78d3e942c7b5512a01/pkg/handler/image/image.go#L65
[fake-client-interceptors]: https://github.com/nscaledev/uni-region/blob/63f373f0f1d3b353d9cc225eade636481d9dd0d1/pkg/handler/region/client_test.go
