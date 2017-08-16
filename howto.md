# document.rootScroller User Guide

This document is a guide on how to use the experimental document.rootScroller
API in Chrome.

## Prerequisites

As of this writing (August 16, 2017) this is a bleeding-edge API.
document.rootScroller in Chrome versions prior to 62.0.3187.0 may be severely
broken in some ways so make sure you're on a newer version (use Canary
channel).

You'll also have to turn the feature on manually, either by passing the
`--enable-blink-features=SetRootScroller` flag or by enabling all experimental
Blink features from:
[chrome://flags/#enable-experimental-web-platform-features](chrome://flags/#enable-experimental-web-platform-features)

## Basics

Setting the rootScroller means that the page will constantly try to use it to
perform viewport scrolling actions. However, in order for a rootScroller to
become "effective" for a document, it must meet some conditions. It must:

  - Be parented in the document's DOM. The document must be active and attached
    to a frame.
  - Be a `display: block` element (i.e. &lt;div&gt; or &lt;iframe&gt;).
  - Exactly match the initial containing block rect. That is:
    ```
    width: 100%;
    height: 100%;
    position: absolute;
    left: 0;
    top: 0;
    ```
  - Be potentially scrollable (i.e.  `overflow: auto`) if it isn't an
    &lt;iframe&gt;.

If the `document.rootScroller` fails to meet any of these conditions, it will
simply not be "effective". In other words, reading document.rootScroller will
still return it but scrolling it won't have any special behavior; it'll behave
as if it was never set (and we implicitly treat the root &lt;html&gt; element
as the "effective"). However, as soon as it does meet the above criteria it'll
become "effective" and take over viewport scrolling actions.

_Note: There is no way to read which element is currently the "effective"
rootScroller for a document. It may or may not be the document.rootScroller,
depending on whether it meets the above criteria_

## Designating a RootScroller

Each document on a page has a rootScroller attribute. Its initial value is
`null`. If a document doesn't set a rootScroller, we default to using the
&lt;html&gt; element as the "effective" rootScroller, meaning that scrolling it
will cause viewport actions. This is the existing behavior today without the
API.

To designate a non-&lt;html&gt; rootScroller, we simply need to set the
attribute to an element in the document's DOM tree.

```
<div id="scroller"></div>
<script>
  document.rootScroller = document.querySelector('#scroller');
</script>
```

That's it! As soon as #scroller meets all the criteria outlined above,
scrolling it will perform viewport actions for its document/frame.

## Composition

The rootScroller can be nested. If an &lt;iframe&gt; becomes the effective
rootScroller in a document, the "global" rootScroller for the page is
calculated by using the "effective" rootScroller from inside the
&lt;iframe&gt;. This is recursive and works from top down.

For example, to determine which element's scrolling will affect the URL bar
(i.e. which element is the "global" rootScroller), we start from the
top-level document. If it has an effective rootScroller that isn't an
&lt;iframe&gt;, use that as the global. If its effective is an &lt;iframe&gt;,
then we use the effective rootScroller of the document _inside_ the
&lt;iframe&gt;. We repeat this process in the until we reach a
non-&lt;iframe&gt; effective.

Put another way, the effective rootScrollers _chain_ across iframes.

## Example

See https://bokand.github.io/rs/div.html for a simple non-nested example.

See https://bokand.github.io/rs/iframe.html for a simple nested example.

See https://bokand.github.io/rs/nested.html for a multi-level nested example.

Note: Each page above nests the one above it. You can navigate from one to the
next by starting from the first example.
