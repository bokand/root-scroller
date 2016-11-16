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

## Proposed API

  * Add a `rootScroller` attribute on `document`.

```
var myScrollerDiv = document.getElementById('myScrollerDiv');
document.rootScroller = myScrollerDiv;
```

If the set element is a valid(\*) scroller, scrolling it should perform all the same actions as the browser performs for document scrolling. E.g. hiding the URL bar, showing overscroll effects, pull-to-refresh, gesture effects, etc. If
no element is set as the `document.rootScroller`, the browser defaults to using the `document.scrollingElement` and
behavior is unchanged from today.

(\*) For some - yet undetermined - definition of valid. Chrome's experimental implementation requires the element to be scrollable and exactly fill the viewport.


### \<iframes\>

When a page sets an \<iframe\> element as the root scroller, e.g:

```
<iframe id="myIframe" src="..."></iframe>
<script>document.rootScroller = document.querySelector('#myIframe');</script>
```

The browser uses whichever element the document in the iframe set as the document.rootScroller (remember, if none
is set it defaults to the `document.scrollingElement`). This nesting works recursively; the iframe could itself set
a child iframe as rootScroller. In this way, the root-most document can delegate root scroller responsibilities to
child documents.


### Example

Here's a common application on the web: a (potentially-infinite) stream of items
that lets you open one up when you click on it. Since the stream is infinite,
navigating to a new page when you open an item means lots of work to recreate the
stream when the user navigates back. What if we could keep the item-view in the
DOM all along? Here's an example of how we'd do that with this proposal:

*Thanks to Dima Voytenko for the 
[MiniApp example](https://docs.google.com/document/d/11kwtjxXelqsIELtHfXDWLWVPrdGJGdy4yvHu-2mGyn4/edit#heading=h.kho1ejnoqhs7),
upon which this is based. This example comes from his experience in trying to make
this work in G+, Google Photos, and the AMP project. See also (with
chrome://flags/#enable-experimental-web-platform-features enabled) my
[conversion of MiniApp](http://bokand.github.io/totese.html) to work with document.rootScroller.*

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
  document.rootScroller = streamView;

  // Clicking on an item will fill the item view with the appropriate content
  // and transition to the item view.
  var itemInStream = document.getElementById('itemInStream');
  itemInStream.addEventListener('click', function() {
    // Populate the DOM in the item container using some helper function.
    replaceContent(document.getElementById('item-container'), itemInStream);

    // Transition to the item view, making it the root scroller.
    transitionView(document.rootScroller, itemView);
  });

  // The back button in the item view will take us back to the stream view.
  document.getElementById('backButton').addEventListener('click', function() {
    transitionView(document.rootScroller, streamView);
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
      document.rootScroller = target;
    });
  }
```
