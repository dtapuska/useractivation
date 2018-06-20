# JS API for querying User Activation
An explainer to define ability to query the state of [user activation](https://html.spec.whatwg.org/multipage/interaction.html#activation).

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

We propose to expose the user activation state as a `UserActivation` object in `MessageEvent` to provide the UserActivation state from the window posting the message. This attribute will only be set if the `WindowPostMessageOptions` options argument on the `postMessage` has `includeUserActivation` with value `true`.

The newly exposed `postMessage` is a new override. The desire to add a new override is to make `postMessage` look similiar across all targets. Currently there are 7 `postMessage` [interface](https://gist.github.com/domenic/d0ea64893c255445574fd535ca89731f) definitions. In making this consistent it will be of the form `postMessage(message, transfer, options)`. To feature detect if the new options are supported an author can test if the postMessage throws an exception when attempting to send a message that will go nowhere.

```javascript

var supportsWindowPostMessageOptions = false;
try {
  window.postMessage(null, [], {targetOrigin: "ftp://example.com"});
  supportsWindowPostMessageOptions = true;
} catch(e) {
}

if (supportsWindowPostMessageOptions) {
  window.postMessage("exampleMessage", [], {includeUserActivation: true});
} else {
  window.postMessage("exampleMessage", "/");
}

```


If in the future we wish to expose an event when `UserActivation` is changed we are able to change UserActivation to be an
EventTarget.

## Proposed Window IDL

```WebIDL
interface UserActivation
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

dictionary WindowPostMessageOptions {
  USVString targetOrigin = "/";
  bool includeUserActivation = false;
};

partial interface Window {
 void postMessage(any message, optional sequence<object> transfer = [],
                                optional WindowPostMessageOptions options);

};

```
