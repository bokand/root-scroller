# Root Scroller

This is a proposed feature that would allow an author to
endow an arbitrary scrolling element with the same specialness currently
assigned to the documentElement.

## The Problem

Applications are typically composed of multiple *views* between which a user can
transition. Traditionally, a web app would transition between its views by
navigating to a new page. This causes a round trip to the server and a jarring
experience for the user. To improve performance and allow for pleasing
transitions, the author may wish to keep multiple views within the DOM,
removing the need to navigate to a new document.

Authors can do this today by making each view out of a viewport-sized,
scrollable &lt;div> element. This is the intuitive solution and allows the
author to animate transitions between these views and manage them as independent
components.

On the other hand, browsers have given essential UX features exclusively to one
special element: the documentElement (&lt;html>). Here are some examples of how
&lt;html> is special:

  * URL-bar Hiding - To maximize the user's screen real-estate, the browser's
    URL-bar is hidden when the documentElement element is scrolled down.
  * Overscroll Affordance - To let the user know the page content is fully
    scrolled, further scrolls generate a "glow" or "rubber-banding" effect. In
    Chrome, this only occurs on the documentElement.
  * Pull-to-Refresh - To allow the user to quickly refresh the page, scrolling
    the documentElement up beyond the content top activates a refresh action
    in Chrome. Chrome disables this effect when the documentElement isn't
    scrollable.
  * Spacebar to Scroll - Many browsers use the spacebar as a shortcut to
    scroll down by a page. This often only works for the documentElement.
  * Tapping top of browser chrome - iOS Safari has this to allow the user 
    to quickly scroll back to the top of the page but it targets only the
    documentElement.
  * Anchoring scroll position on rotation

Thus, authors have a choice: use the intuitive method and lose all these
essential UX features, or swap the *content* of each view in and out of the
documentElement. The latter turns out to be surprisingly difficult to
implement in a portable way for all the reasons listed in the
[MiniApp example](https://docs.google.com/document/d/11kwtjxXelqsIELtHfXDWLWVPrdGJGdy4yvHu-2mGyn4/edit#heading=h.kho1ejnoqhs7).
To summarize, swapping content within a single scroller is complicated since
the scroll offset is shared between two conceptual views; it requires tricks to
keep the content of multiple views overlaid and we have to manually keep track
of each view's scroll offset. Likewise, animating transitions becomes tricky
since the animation must be timed carefully with the content swap. Simple
behaviors which emerge naturally from using separate &lt;div>s become difficult
and complicated to implement when forced to share the special documentElement.

Want to see the problem in action? Go to photos.google.com or any AMP article.
Notice that scrolling down doesn't hide the URL bar.

## The Root Scroller

This proposal introduces the concept of a _root scroller_. This generalizes the
special behaviors given today only to the documentElement to a single _root
scroller_ on the page. By default, the documentElement is the page's _root
scroller_, however, by changing which element is the _root scroller_, the
special behaviors can be endowed to scrollers other than the documentElement.

### Changing the Root Scroller

This proposal previously explored an explicit API to let the page designate the
_root scroller_ element explicitly. Due to lack of interop interest, this API
has been dropped in favour of an implicit approach. Should interest arise in an
explicit API, this approach doesn't preclude resurrecting the API.

### Implicit Root Scroller

The alternative to an API is to let the browser automatically decide which
element on a page should be the _root scroller_. We call this approach the
_Implicit Root Scroller_.

There are certain criteria that an element must meet in order to be eligible
for promotion to the _implicit root scroller_. These are designed for two
purposes.

The first is to ensure a consistent UX with existing documentElement
scrolling. We may decide to loosen these restrictions in the future to allow
a greater variety of scrollers to benefit from document scrolling UX.

The second is to determine the semantically most important scroller on the
page. In other words, the set of criteria and heuristics should pick a unique
scroller on the page that's hosting the content the user currently cares about.

The current set of criteria are as follows:

* The Element is connected to the DOM tree, is displayed and is of “box” type
  (essentially, a display: block or <iframe>)
* For non-iframe Elements, it must have an overflow clip
* The Element must exactly fill the viewport. That is, it must be at location
  (0, 0) and it’s size must match the viewport size exactly.
* The Element must be user scrollable, have overflow, and be fully visible (e.g.
  opacity: 1)
* The Element must not have an ancestor that is scrollable or has any kind of
  clipping or masking (e.g. overflow: hidden, clip-path, mask, etc.)

Intuitively: if we notice the main document doesn’t have any scrollable content
itself, but there is a screen-filling scroller that’s not “weird” in any way -
the scroller is promoted to _root scroller_. In the rare case there are 
multiple such candidates, neither is promoted.

These criteria are reevaluated after a style or layout update so that the
_root scroller_ must always meet these criteria.

## Improved Functionality

A major benefit of this feature is that authors can keep logically separate
views in separate parts of the DOM. For example, a common app model is to have
a stream of items from which users can open an item to view more details (e.g.
a news stream). This app has a "stream view" and an "item view".

The intuitive structure of this app would be:

```
<div id="streamView">
 ...
</div>
<div id="itemView" class="hidden">
 ...
</div>
```

When the user opens an item, the #itemView is populated and unhidden, perhaps
with some animated transition.

Without _root scroller_, this application would lose all the document-scrolling
like URL bar hiding. With _implicit root scroller_, the currently in-view
scroller automatically gets the document-scrolling UX (assuming the criteria
above are met) without forcing authors to do DOM surgery.

## Status

This feature is currently implemented in Chrome and on track to ship by default
in M73, see https://www.chromestatus.com/features/5162094739587072

## Examples

See http://bokand.github.io/rs/implicit.html for an example of an application
layout that levarages the _implicit root scroller_.

Note: pre-M73 and in other browsers, scrolling on either the stream or item
view doesn't hide the URL bar or show the overscroll glow affordance. In M73,
on both views the URL bar hides, overscroll glow is shown, and UX is as-if the
view was the documentElement.
