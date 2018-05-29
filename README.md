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

We propose to expose the user activation states as an `Activation` object in `window`:
* The field `hasBeenActive` indicates whether a `window` has ever seen a user activation in its lifecycle. Some APIs such
as autoplay are controlled by this sticky boolean. This boolean does not reflect whether the current [task](https://html.spec.whatwg.org/multipage/webappapis.html#concept-task) is triggered by a [user activation](https://html.spec.whatwg.org/multipage/interaction.html#activation) but that the current task has been seen **one**.
* The field `isActive` indicates whether a `window` currently has user activation in its lifecycle.

If in the future we wish to expose an event when `Activation` is changed we are able to change Activation to be an
EventTarget.

## Proposed Window IDL

```WebIDL
interface Activation
{
    readonly boolean hasBeenActive = false;
    readonly boolean isActive = false;
};

partial interface Window {
    [Replaceable, SameObject] readonly Activation activation;
};
```
