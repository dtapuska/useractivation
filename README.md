# JS API for querying User Activation
An explainer to define ability to query the state of [user activation](https://html.spec.whatwg.org/multipage/interaction.html#activation).

## Motivation

Suppose a page that contains a cross-origin iframe wants to allow that iframe to request
(say, via a `postMessage` contract) becoming larger, because it's appropriate for that
iframe to become larger when the user interacts with it. However, it doesn't entirely
trust that iframe from trying to grab extra attention, so it wants to do the same check
that the browser does as part of its popup blocking code, which is a test for user
activation. So this API allows the containing page to validate the `postMessage`
from its iframe by only honoring the request to become larger if there is currently
a user activation, that is, if the message appears to have been the result of a real
user interaction with that iframe.

Exposing the primitives that the user agent has are necessary to implement these
specific behaviors correctly. It currently is possible without an API to detect
that a user activation has occurred and one is active. However these approaches
may be inefficient (creating audio and video to check state) or have destructive
properties and require escalated security permissions (clipboard APIs).

## Abstract

Currently there are ways to determine if a user gesture is active:
* Copy and Pasting items to the clipboard
* Requesting play on a media element and checking its status

Some of these actions are destructive to the user (ie. replacing
the clipboard text) and while we cannot necessarily prevent them,
it is advisable to provide functionality to the platform to
discourage their use.

A few examples of use are:
* [Video autoplay](https://github.com/ampproject/amphtml/blob/f7bb404d853df97645bb1a38fffc28b7efac16b8/src/utils/video.js#L26)
* [Audio play](https://github.com/ampproject/amphtml/blob/e32fdddfa38e043cd1df102d50e6d12911e1227e/extensions/amp-iframe/0.1/amp-iframe.js#L675)

We propose to expose the user activation states as a `UserActivation` object in `navigator`:
* The field `hasBeenActive` indicates whether the associated `window` has ever seen a user activation in its lifecycle. Some APIs such
as autoplay are controlled by this sticky boolean. This boolean does not reflect whether the current [task](https://html.spec.whatwg.org/multipage/webappapis.html#concept-task) is triggered by a [user activation](https://html.spec.whatwg.org/multipage/interaction.html#activation) but that the current task has been seen **one**.
* The field `isActive` indicates whether the associated `window` currently has user activation in its lifecycle.

We propose to expose the user activation state as a `UserActivation` object in `MessageEvent` to provide the UserActivation state at the time of posting the message. This attribute will only be set if the `PostMessageOptions` options argument on the `postMessage` has `includeUserActivation` with value `true`.

The newly exposed `postMessage` is a new override. The desire to add a new override is to make `postMessage` look similiar across all targets. Currently there are 7 `postMessage` [interface](https://gist.github.com/domenic/d0ea64893c255445574fd535ca89731f) definitions. In making this consistent it will be of the form `postMessage(message, options)`. To feature detect if the new options are supported an author can test if the postMessage method supports a single argument.

```javascript

var supportsWindowPostMessageOptions = window.postMessage.length < 2;

```

Alternatively if using inside a worker the new support can be detected via:

```javascript

let supported = false
try {
  postMessage("", { get transfer() { throw 1; } })
} catch(e) {
  if(e === 1) {
    supported = true
  }
}
console.log(supported)

```

If in the future we wish to expose an event when `UserActivation` is changed we are able to change UserActivation to be an
EventTarget.


## Example

An example of the iframe use case is [available](https://dtapuska.github.io/useractivation/example.html). It demonstrates the
current state of affairs and the promosed use of the API.

Specifically interesting code in the example is the child is opting in to send the user activation information if it can.

```javascript

  // Check that WindowPostOptions is supported
  if (window.parent.postMessage.length == 1) {
    window.parent.postMessage('resize', {includeUserActivation: true});
  } else {
    window.parent.postMessage('resize', '/');
  }

```

The reading of the userActivation argument on the MessageEvent in the parent iframe.

```javascript
    if (event.userActivation.isActive) {
        return Promose.resolve();
    }
```


## Security Considerations

We explicitly chose an opt-in API (the default for `includeUserActivation` is `false`) so that a message channel doesn't accidentally
leak interaction behavior to another receiver unknowingly. The sender of the message must opt in for each message sent, indicating on the
`PostMesageOptions` that it permits the current user activation state to be cloned and provided to the receiver of the message.
For example communication between an iframe and a parent document is not necessarily mutual. An iframe may wish to provide its
user interaction state to the parent but the parent may not wish to indicate this to an iframe.

## Alternates Explored

We explored an alternative of exposing a UserActivation attribute to be cross origin on a Window object. However this approach had
two drawbacks:
* The interaction state at the time of the message is posted isn't captured.
* Chosing to expose the attribute conditionally to specific origins is complex.

## Proposed IDL

```WebIDL
[
  Exposed=(Window,Worker,AudioWorklet)
] interface UserActivation
{
    readonly attribute boolean hasBeenActive;
    readonly attribute boolean isActive;
};

partial interface Navigator {
    readonly attribute UserActivation userActivation;
};

partial interface MessageEvent {
    readonly attribute UserActivation? userActivation;
};

partial dictionary MessageEventInit {
    UserActivation? userActivation = null;
};

dictionary WindowPostMessageOptions : PostMessageOptions {
  USVString targetOrigin = "/";
};

dictionary PostMessageOptions {
  sequence<object> transfer = [];
  boolean includeUserActivation = false;
};

partial interface Window {
 void postMessage(any message, optional WindowPostMessageOptions options);
};

partial interface Worker {
 void postMessage(any message, PostMessageOptions options);
};

partial interface MessagePort {
 void postMessage(any message, PostMessageOptions options);
};

```
