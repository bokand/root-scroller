# Non-Document Root Scrollers

Non document root scrollers is a proposed feature that would allow an author to
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

## Proposed Solution



We need to provide authors some way to make the intuitive "each view is a div"
solution workable. This means giving an arbitrary scroller the same powers as
the documentElement; anointing it the *root scroller*. Lets make the documentElement
less special. The web is the only UI framework that gives special powers to one, and
only one, special scrolling element.

I propose an explicit mechanism to allow the author to specify which element should
be treated as the root scroller and, importantly, *when*. This will go some way to
*explaining* how the page interacts with browser UI and give authors a some ways to
control those interactions.

## document.scrollingElement

The web platform recently introduced
[document.scrollingElement](https://drafts.csswg.org/cssom-view/#dom-document-scrollingelement).
The read-only scrollingElement attribute was added as a helper to ease
transition for sites while WebKit-based/derived browsers fixed an
[age-old interop issue](https://dev.opera.com/articles/fixing-the-scrolltop-bug/).
Basically, different browsers designate different elements as the root
scrolling element (aka "viewport"). WebKit based browsers (including Chrome
and Opera) apply root scrolling to the &lt;body> element while IE and Firefox
comply with the specification and apply scrolling to the root element (&lt;html>).
To ease the transition for when Blink and WebKit fix the bug, document.scrollingElement
returns the element that's used as the root scroller: &lt;body> on WebKit/Blink
and &lt;html> in the rest.

This sounds an suspiciously like what was described above: different elements
acting as the root scroller. What if we allowed *setting* the scrollingElement?
This would be conceptually compatible with the meaning of scrollingElement. It
should represent to root-most scroller so that scrolls can be set and read from
script in a uniform way.

*Note: We'd likely want a setter method, rather than making the attribute
writable, for easier feature detection and failure ergonomics.*

### Example

Here's a common application on the web: a (potentially-infinite) stream of items
that lets you open one up when you click on it. Since the stream is infinite,
navigating to a new page when you open an item means lots of work to recreate the
stream when the user navigates back. What if we could keep the item-view in the
DOM all along? Here's an example of how we'd do that with this proposal:

*Thanks to Dima Voytenko for the 
[MiniApp example](https://docs.google.com/document/d/11kwtjxXelqsIELtHfXDWLWVPrdGJGdy4yvHu-2mGyn4/edit#heading=h.kho1ejnoqhs7),
upon which this is based. This example comes from his experience in trying to make
this work in G+, Google Photos, and the AMP project.*

Here's the markup:

```html
  <html>
    <style>
      body, html {
        width: 100%;
        height: 100%;
      }
      .view {
        position: absolute;
        top: 0px;
        left: 0px;
        width: 100%;
        height: 100%;
        z-index: 1;
        overflow: auto;
      }
      .invisible {
        visibility: hidden;
        z-index: -1;
      }
      .transitioning {
        z-index: -1;
      }
    </style>
    <body>
      <!--Notice: The body element itself has no scrolling, only the views scroll-->
      <div id="streamView" class="view">
        <div>
          <!--CONTENT GOES HERE-->
        </div>
      </div>

      <div id="itemView" class="view invisible">
        <div id="backButton" class="button"></div>
        <div id="itemContainer">
          <!--CONTENT GOES HERE-->
        </div>
      </div>
    </body>
  </html>
```

And the script to make the transitioning happen:

```javascript
  var streamView = document.getElementById('streamView');
  var itemView = document.getElementById('itemView');

  // When the page loads, start with the stream view being the "root scroller".
  document.setScrollingElement(streamView);

  // Clicking on an item will fill the item view with the appropriate content
  // and transition to the item view.
  var itemInStream = document.getElementById('itemInStream');
  itemInStream.addEventListener('click', function() {
    // Populate the DOM in the item container using some helper function.
    replaceContent(document.getElementById('item-container'), itemInStream);

    // Transition to the item view, making it the root scroller.
    transitionView(document.scrollingElement, itemView);
  });

  // The back button in the item view will take us back to the stream view.
  document.getElementById('backButton').addEventListener('click', function() {
    transitionView(document.scrollingElement, streamView);
  });

  // Note how simple this function is; most of it is details about how to fade
  // and display the views. Since the views are siblings in the DOM and we don't
  // have to move any DOM around it's a straightforward matter.
  function transitionView(current, target) {
    target.classList.remove('invisible');
    target.classList.add('transitioning');
    target.style.opacity = 0;
    
    var animation = new SequenceEffect([
      new KeyframeEffect(current, [ {opacity: 1}, {opacity:0} ], {fill: 'forwards', duration: 500}),
      new KeyframeEffect(target, [ {opacity: 0}, {opacity:1} ], {fill: 'forwards', duration: 500}),
    ]);

    document.timeline.play(animation).finished.then(function() {
      // When the transition is complete, make the "current view" visible.
      current.classList.add('invisible');
      target.classList.remove('transitioning');
      current.style.opacity = 1;
      target.style.opacity = 1;

      // Mark the newly current view as the root scroller so it gets all the
      // nice browser UX features.
      document.setScrollingElement(target);
    });
  }
```
