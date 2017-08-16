# document.rootScroller User Guide

This document is a guide on how to use the experimental document.rootScroller
API in Chrome.

First of all, as of this writing (August 16, 2017) this is a bleeding-edge API.
document.rootScroller in Chrome versions prior to 62.0.3187.0 may be severely
broken in some ways so make sure you're on a newer version (use Canary
channel).

## Basics

Setting the rootScroller means that the page will constantly try to use it to
perform viewport scrolling actions. However, in order for a rootScroller to
become "effective", it must meet some conditions. It must:

  - Be parented in the DOM in an attached/active Document.
  - Be a `display: block` Element (i.e. &lt;div&gt; or &lt;iframe&gt;.
  - Exactly match the initial containing block rect. That is:
    ```
    width: 100%;
    height: 100%;
    position: absolute;
    left: 0;
    top: 0;
    ```
  - If it isn't an &lt;iframe&gt; it must be potentially scrollable (i.e.
    `overflow: auto`)

If the `document.rootScroller` fails to meet any of these conditions, it will
simply not be "effective". In other words, it won't have any special behavior
and it'll be as if it was never set (and we implicitly treat the root
&lt;html&gt; Element as the "effective"). However, as soon as it does meet the
above criteria it'll become "effective" and take over viewport scrolling
actions.

## Designating a RootScroller

Each document on a page has a rootScroller attribute. It's initial value is
null. If a document doesn't set a rootScroller, we default to using the
&lt;html&gt; Element as the "effective" rootScroller, meaning that scrolling it
will cause viewport actions. This is the existing behavior today without the
API.

To designate a non-&lt;html&gt; rootScroller, we simply need to set the
attribute to an Element in the document's DOM tree.

```
<div id="scroller"></div>
<script>
  document.rootScroller = document.querySelector('#scroller');
</script>
```

That's it! As soon as #scroller meets all the criteria outlined above,
scrolling it will perform viewport actions.

## Composition

The rootScroller can be nested. If an &lt;iframe&gt; becomes the effective
rootScroller in a Document, the "global" rootScroller for the page is
calculated by using the "effective" rootScroller from inside the
&lt;iframe&gt;. This is recursive and works from top down.

For example, to determine which Element's scrolling will perform viewport
actions (i.e. which Element is the "global" rootScroller), we start from the
top-level Document. If it has an effective rootScroller that isn't an
&lt;iframe&gt;, use that as the global. If its effective is an &lt;iframe&gt;,
then we use the effective rootScroller of the Document _inside_ the
&lt;iframe&gt;. We repeat this process in the until we reach a
non-&lt;iframe&gt; effective.

## Example

See https://bokand.github.io/rs/div.html for a simple non-nested example.

See https://bokand.github.io/rs/iframe.html for a simple nested example.

See https://bokand.github.io/rs/nested.html for a multi-level nested example.

Note: Each page above nests the one above it. You can navigate from one to the
next by starting from the first example.
