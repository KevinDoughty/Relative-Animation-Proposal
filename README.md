# Relative Animation Proposal


## Introduction

Relative animation is an animation usage pattern built on additive animation
now in part possible on the web using Web-Animations.
This is an explanation of its approach and a proposal for new features and amendments to multiple specs.
Some features have been implemented as proof-of-concept in an unlanded Firefox Nightly patch
[https://phabricator.services.mozilla.com/D46887].
Proposed changes to specifications are incomplete but available at
[https://github.com/w3c/csswg-drafts/compare/main...KevinDoughty:master]


Animations are additive and run from the old value minus the new value to zero.
They run relative to a base value provided by traditional layout mechanisms instead of persistent animations.
Relative value conversion for non-additive animation was popularized among web developers as FLIP,
but there are more benefits when additive.
Transitions only ever need specified values,
rather than computed values for beginning where a previous animation left off.
It was designed for optimizations that run off the main thread.
On interruption, animations are not replaced like with CSS Transitions.
Computed values are not required for beginning where a previous animation left off.
Blending is provided by the timing function.
Blending of keyframe animation gives rise to simple emergent physics
outperforming a more "real" but main thread bound physics sim.
The pattern is best suited for user interface animation triggered by interaction and events.

### History

It was initially conceived as a Core Animation usage pattern for animating laid out glyph runs and inline grid content.
Bug reports and feature request radars,

and conversation over IRC contributed to the contents of at least two WWDC videos,
the second half of
[https://developer.apple.com/videos/play/wwdc2014/236/]
and a few minutes of
[https://developer.apple.com/videos/play/wwdc2017/230/]
A derivative pattern became the default animation of UIView class methods starting in iOS 8,
[https://developer.apple.com/documentation/uikit/uiview/1622418-animatewithduration?language=objc]
using bug workarounds (but not the feature requests).
[http://www.openradar.me/12085417]
It was not implemented at the Core Animation level and thus does not seem to be present in SwiftUI.
On AppKit for desktop, implicit view animtion with Core Animation was never possible without resorting to method swizzling hacks.

The Firefox Nightly patch is proof of concept.
It may have conceptual flaws and significant bugs, but an implementation in code informs and strengthens this proposal.
Years ago, a less matured but similar proof of concept was implemented in Webkit,
[https://github.com/KevinDoughty/webkit/commit/759cb04793ed339e78964af54560c17b2a6bf47a]
but this and my vague proposals made to the css mailing list and public-fx did not gain any traction.
Other techniques of relative animation that have been overlooked by many will be addressed here.
I previously wrote about it in a now defunct blog
[https://kxdx.org]
which barely received any recognition except for the generous Dan Abramov and a few others
https://medium.com/@dan_abramov/my-react-list-862227952a8c
Early examples using the Web-Animations javascript shim for the blog and the public-fx mailing list broke when a re-write to the shim unfortunately removed additive animation.


## The pitch, or, why this is so awesome


Relative animation excels when triggered from user events and interaction,
in particular the special case of interrupting in-progress animations,
even repeatedly from continuous events like dragging.
It is a pattern best suited for user interface animation,
as opposed than cartoons or animation with narrative.
Animations need only use discrete source and destination values, even on iterruption,
rather than values that begin where a previous left off.
Developers can ignore current calculated animated values,
where the animation is at any particular moment, and keep a simpler temporal mental model.

Animations blend together on interruption,
with a limitation being the timing function must be used to achieve this blending.
In most cases all calculations can be made when animations start,
without the need for cleanup or setting new values in callbacks or completion handlers,
which should now be considered an anti-pattern.
It is easy to compose complex animations on a single property comprised of multiple parts,
for instance simultaneous transforms.
Keyframe animations can be used to create an emergent physics that,
unlike other techniques for spring animation, do not need to be bound to the main thread.
Designing transient animations from discrete base state instead of having to deal with current animated values,
to use an old cliche, makes things easier to reason about.

### What is possible on the web

Web-Animations brings the `add` and `accumlate` composite modes which open many possibilities, but have to be manually created.

### What is not possible on the web

The feature requests in this proposal notwithstanding,
another hurdle is the lack of avilability of discrete, non-animated metrics.
Functions like `getBoundingClientRect` return animated values only.
Admittedly, sibling element position can be animated using offsetLeft and offsetTop,
but would become more complicated when reparenting.

### What could be possible on the web

It could make possible an arguably superior solution to transition reversal than the current implementation.
[https://www.w3.org/TR/css-transitions/#reversing]
It could make possible interruption of CSS Animations giving transition-like behavior, but with smoothing, without needing to know current animated values.
It could make possible significant opimization opportunites in the browser, 
because fewer style recalculations are needed if only discrete non-animated values are used.



### A simple example

To explain how to manually construct a relative animation by example,
animating from 2 to 10 would be accomplished by having the specified value instantly set to 10,
then adding an animation from the old value minus the new value, which is from -8 to 0.
Added to the base value of 10,
the animation would put the computed value back at 2 and proceed to 10.
If interrupted midway at 7 for the current computed value, and with a new destination value of 1,
the first animation would not be removed but instead continue on from -3 to 0,
while a new animation would be added from 9 to 0.
The discrete base value would be instantly changed from 10 to 1,
which when summed with both animations would remain visualy consistent.
If a nice easing is used, the change in direction would be seamless, smooth, and beautiful.

### Easing is pretty much required

When considering the same simplified example but using a linear timing function instead,
its behavior would look erroneous.
Animating from 2 to 10 would be accomplished in the same way,
instantly setting the base value to 10 and adding an animation from -8 to 0.
A linear timing function may not produce very natual motion,
but the seams would really show on interruption.
The new destination base value would be set 1, and an animation added from 9 to 0.
The first animation running from -3 to 0 and the second from 9 to 0 would exactly counteract the other,
because the signs of their from values are opposite.
All motion would stop at 7 until the first animation elapsed.
Then motion would start again, as the second animation continued running from 6 to 0,
added on top of the base value of 1,
resulting in the apparent motion from 7 to 1.


### Emergent Physics

Limited spring physics can be emulated using keyframes.
Additive animation is not a requirement for this.
However, interruption would otherwise be handled in the traditional way.
The animation would suddenly stop and a new one would begin where the previous left off.

Despite never removing previous animations on interruption,
this works surprisingly well when additive and relative.
Information is never destroyed.
This has unexpected and wonderful consequences but with reasonable values,
very natural motion can be designed.
Layout state gets converted from matter to energy, the matter of layout metrics and the energy of animation.
A better metaphor might be using force and acceleration, or inertia.

A purist or comedian might argue (but the author does not) to stop in place all running animations,
equal and opposite animations should be added, instead of removed.



### How this is different

This pattern differs from previous intended uses of additve animation in that animations counterintuitively end at zero.
When complete, animations can be removed without further action because they no longer have any effect.

Applied to transitions, the pattern differs from traditional implementations in that it does not depend on current calculated animated values.
Interruption works with any number of states rather than just two,
unlike the Faster reversing of interrupted transitions section of the CSS Transitions 1 spec.

On Core Animation in OS X, forward fill modes were meant to persist one animation so a second additive animation starting from zero could run concurrently without disruption.
GSAP might be considered a forward-filling animation library,
its additive animation plugin also begins at zero instead of ending there.
User interface layout should not be performed by forward-filling animations.
Layout should be perfomed by the tools specially built for the task.
Animations should run relative to discrete values set using CSS, the appropriate styling technique.
The relative pattern obviates fill modes, for both layout and interruption.
They and their memory leaks should now be considered an anti-pattern.


### Benefits

Animation behavior is a superior way to handle interruption of an in-progress animation.
This has consequenses from simple interactions to more complex animations,
as well as overall design aesthetic.
Animating inline elements on container resize is not common in user interfaces because of the difficulty.
When relative, elements don't bunch up in the center but rather flow gracefully.
Perceived performance is much higher this way.
Even simple hover or toggle animation reversal behavior

Fewer style recalculations are the result of the pattern only ever needing discrete values.
It is not a premature optimization.
This pattern is also meant for GPU composited properties, an optimization in itself.

Only needing discrete values for animation is a convenience for the developer.
Physical and temporal spaces of the user interface can be envisioned more easily,
in regards to handling animation interruption or throughout user events.
In other words, more easy to reason about.



### Don't call it FLIP

The primary (only?) benefit of non-additive FLIP is not needing animation completion handlers to reset transform animations.
FLIP only works with non-additive transform animations because element frame is determined by layout,
the value to which it can run relative.
It is possible to animate an opacity fade out from zero to one using non-additive FLIP, but not a fade in.
FLIP replaces existing animations with a new one that begins where the previous left off.
It lacks blending.
Spring physics could be created with keyframes but the seams would show on interruption.
Yes of course it was a fantastic implementation, my hat is off to Paul Lewis.



### Failures of the pattern

"Spooky animation at a distance" is a phrase coined to describe curious artifacts when a stagger animation is interrupted.
[https://twitter.com/KvnDy/status/741727269228544000]
Without discrete values, layout metrics using `getBoundingClientRect` are incorrect.

Copying animations and applying them from one element to another is a commonly needed technique but unwieldy in practice.
If an animating element is to be split in two,
the animations from the original would need to be copied to the new element to maintain visual consistency.

Reparenting a node removes all animations, due to supposed security concerns.
The workaround is copying animations, which is needed regardless if an animation is additive or not.

A `relative` CSS animation or transition should not be removed when reparenting but instead run its full lifecycle, which would solve:
https://github.com/w3c/csswg-drafts/issues/5334
and hopefully obviate:
https://github.com/w3c/csswg-drafts/issues/5524
related:
https://github.com/w3c/csswg-drafts/issues/3309
and:
https://github.com/whatwg/dom/issues/808
affecting:
https://github.com/facebook/react/issues/19406
https://github.com/preactjs/preact/issues/2637
https://github.com/infernojs/inferno/issues/1519
https://github.com/MithrilJS/mithril.js/issues/2612
from:
https://twitter.com/isiahmeadows1/status/1284726730574315522
This is not implemented in D46887. Watch out for memory leaks. No don't, this will not be implemented.




## The Request quick rundown

### Essential features that have a proof-of-concept implementation in an unlanded Firefox Nightly patch
* A `subtraction` procedure to convert values from absolute to relative.
* A `relative` CompositeOperation for Web-Animations which behaves like `accumulate` for lists but with automatic conversion from absolute to relative values.
* A `transition-composition` property which takes the keywords `replace` (the default) or `relative`.
* The `animation-composition` property with an additional `relative` keyword.
* A `perfect` timing function that resolves to `cubic-bezier(0.5, 0, 0.5, 1)`.
* A `discrete-metrics` CSS property that takes the keywords `none` (the default), `mixed`, or `strict`.


### Additional features that I haven't thought much of yet:
* Relative animation for certain motion-path properties

### Lower-priority features for `transition-interrupt` : `special`
* A default timing function of `cubic-bezier(0.5, 0, 0.5, 1)` or `perfect`.
This may be difficult to fit in the spec and codebase, and might not gain enthusiastic support, like adding new color names.

### Lower-priority features for `animation-composite` : `relative`
* Use of a single timing function spanning all CSS Animation keyframes.

### Lower-priority features for `animation-composite` : `relative`
* A default fill mode of `backwards` when run forward or `forwards` when run in reverse (to prevent flicker)

## Features which should be considered for implementation but are not fully realized yet:
### It should be possible to:
* Prevent animations and transitions from affecting inherited values, so elements inherit specified rather than computed animated values.
* Prevent animations and transitions from affecting intrinsic sizing, or contribute specified rather than computed animated values.
* Prevent animations and transitions from affecting scrollable overflow, or contribute specified rather than computed animated values.
* Use destination values rather than computed animated values when performing hit testing.
* Ensure that elements positioned off-screen due to animation are not occluded.

### In addition, it might make sense if it were easier to
* Distinguish between animated and non-animated computed values in the spec.
[https://www.w3.org/TR/css-cascade-4/#value-stages]
It may be complicated already, but this means a `discrete value` as well as a `computed value`
### Plus:
* A better easing spec could allow for push animations, which provide a solution to dragging animation latency

### Outlandish features that might fulfill an animator's wildest dreams
* An "Animated OM" for animating Javascript numbers and arrays of discrete objects.
* A type of discrete object "fake set" animation for collections that applies a given sort function to its results


Many things remain unimplemented like the various motion-path properties, and there are many bugs in animating certain transforms.


## Essential features that absolutely should be implemented

### A `relative` CompositeOperation for Web-Animations

A `relative` CompositeOperation for Web-Animations behaves similarly to `accumulate` except automatically converts from absolute to relative values.
Relative values refers to converting values from the old value minus the new value and animating to zero.
For keyframes, `A, B, C, D` become `A-D, B-D, C-D, D-D`.
[https://www.w3.org/TR/web-animations-1/#the-compositeoperation-enumeration]
A (private) `subtract` operation to perform the conversion would be needed in the css-values-4 spec as well as a `relative` operation.
[https://www.w3.org/TR/css-values-4/#combining-values]
Proof of concept is implemented in an unlanded Firefox Nightly patch D46887
[https://phabricator.services.mozilla.com/D46887]
but not without bugs.
An example video
[https://twitter.com/KvnDy/status/1103100842247364608]
and source
[https://codepen.io/kvndy/pen/RdGgap]
show its usage


While it can be done manually, relative conversion of values is for the convenience of developers.
A Javascript animation framework to enable this pattern would be difficult to implement.
A framework for providing automatic relative conversion would have to use the Typed OM and perform conversion manually ("FLIP" for arbitary properties),
which would be unnecessarily cumbersome.

Relative conversion was required for NSView animation but the feature request to Apple was ignored,
making relative view animation in AppKit impossible without method swizzling.
(Also, view animation on OS X only allows one animation per property at a time, which is its other failure.)
The very first relative animation library was for Core Animation on Mac OSX addressed these issues  
[https://github.com/KevinDoughty/Seamless]


Single keyframe behavior should use specified values when relative,
otherwise computed to begin where a previous animation left off
[https://github.com/web-animations/web-animations-js/issues/14]



### More specific implementation details
Private subtract and public relative modes, relativeEndpoints, etc.

### To be considered:
The D46887 patch has issues related to transform matrices, lists, and CSS filters which need to be resolved.
Does it build on the correct operation, should it be `add` or `accumulate`?
Or does it need both?
Do there need to be separate modes similar to the distinction between `add` and `accumulate`?



### Related:

The additive CSS proposal is interesting but would not have a corresponding relative operation,
because it applies to only one value, not old and new values to calculate a difference.
[https://github.com/w3c/csswg-drafts/issues/1594]




## A `transition-interrupt` property

A `transition-interrupt` property for CSS Transitions would take the keywords `regular`, which is the default, or `special`.
If `special`, property transitions would be additive and animate from the old value minus the new value to zero.
The CompositeOperation of the transition animation would be `relative`.
This would provide blending on interruption, and an alternative to faster reversing of interrupted transitions as described in
[https://www.w3.org/TR/css-transitions-1/#reversing]

The shorthand property should allow these keywords last, for example, `transition: transform 1s cubic-bezier(0.5, 0, 0.5, 1) 0s special;`.
This would benefit from a different default easing, something other than linear to provide smoothing on interruption.
Well chosen defaults would permit these improvements by merely writing `transition: transform 1s special;`.


In the current implementation, changing this property while a transition is in progress will result in visual inconsistencies.
The value will jump, but I am not sure if this would be considered a bug.
Fixing it would make the code somewhat more complex.
Another example video
[https://twitter.com/KvnDy/status/1102807177302065152]
and source
[https://codepen.io/kvndy/pen/RdGVvR]
show its usage, and also will not work in any browsers.

### Changing this value
Switching from special to regular and vice versa requires additional effort.

When switching from `regular` to `special`, the existing transition animation should not be replaced.
Instead, it should be mutated (or removed and re-created) with a `CompositeOperation` of `relative`.
This ensures the underlying value is the discrete, specified, non animated value, and not the animated value of the existing transition.

When switching from `special` to `regular` when there is only one in-progess transition animation, the rules do not need to be altered.

When switching from `special` to `regular` when there are multiple in-progess transition animations, the from value of the new transition needs to be the composited value of all exisiting transitions.
Then all of the running transition animations must be removed, instead of just one.

### Faster reversing of interrupted transitions
[https://www.w3.org/TR/css-transitions-1/#reversing]
should be entirely bypassed when `transition-interrupt` is`special`, with extra care taken when switching back and forth between it and `regular`.
These reversal rules are somewhat convoluted and mercifully need only apply to a single transition.
The apparent asymmetry of animation speed on reversal is one of the many problems relative animation solves.






## A `relative` keyword for the CSS Animations 2 `animation-composite` property

If `relative`, animation values are converted to their relative conterparts. 
The CompositeOperation of the animation would be `relative`.

There are some potential default values for developer convenience when using relative animation,
but many of them are difficult to add to both the spec and the codebase.
There is however one that is not merely for convenience but rather necessary for preventing the impression of buggy behavior.
A backwards fill mode (or forwards if the animation is running in reverse) would prevent occasional flicker when a second animation is running.
It is an artifact of being async not understood by the author.
It seems sometimes animations are added without affecting an element's computed value.
Layout places an element in a new destination before the animation's effect gets applied.
A backwards fill mode applies an animation's effect to times earlier than its start time to prevent any flicker.
It should only be applied when the animation's delay is zero.
The danger is many unknowing web developers filling the internet with flicker that makes a browser's own rendering look flawed when it is not.
It is possible there is a fix in the browser, perhaps


### Additional features that I haven't personally implemented yet:

The most challenging obstacle for this pattern on the web is that it relies on non-animated values,
which are not readily accessible.
The functions `getComputedStyle`, `getBoundingClientRect`, and `getClientRects` return animated values which breaks the pattern.
https://drafts.csswg.org/cssom-view/#dom-element-getboundingclientrect



### getDiscreteComputedStyle, getDiscreteBoundingClientRect, and getDiscreteClientRects
The term could be "Unanimated" instead of "Discrete".
This clumsily expands the API footprint.
Worse, the implementation would be expensive.
While a discrete implementation of `getComutedStyle` would be more or less trivial to implement,
a discrete `getBoundingClientRect` and `getClientRects` would either require a lot more memory or many more style recalculations.

Better might be adding an optional `discrete` argument to each function,
that accepts a boolean and defaults to false.
This would still be expensive.


### Brutally Discrete

A radical proposition which radically changes behavior radically,
this may have vast unforseen benefits and consequences.
A better alternative might be a new CSS property that would make all calls to `getBoundingClientRect` and the like return discrete, non-animated values.
The developer might be accepting trade-offs like seemingly inaccurate hit testing.
Alternately, it could be a list of properties like `will-change`
For the compositor properties transform and opacity, animation-only style recalculations would never occur,
which would be a huge performance optimization.
Animation-only style recalculations may be signficantly optimized compared to their regular style recalcuation counterparts.
But layout information is only stored in one place, which requires overwriting with animated or non-animated values as needed.
Preventing these back-and-forth repeated style recalculations is the improvement.

The rest of the (non-composited) properties might actually perform slightly worse,
if they have to do an extra non-animation style recalculation for functions like `getBoundingClientRect`.

A significant flaw is there is no way to manually animate scroll height.
There is no scroll view hierarchy like on OS X.
That might not be true, it may be possible to animate, or a way could be proposed.




### Better defaults for `transition-interrupt` and `animation-composite` : `relative`

The default timing function for `transition-interrupt` and `animation-composite` : `relative`
should be `cubic-bezier(0.5, 0, 0.5, 1)`, or a new keyword `perfect` giving the equivalent, if possible.
Admittedly, it would be difficult to fit into the spec for breaking simpler concepts like only having an initial value of `linear`.
It is not only for personal preference.  
The original need was an apparent bug or misunderstanding of behavior when animating scaling matrices in Core Animation that cannot be readily reproduced.
A slight wobble was introduced, perhaps because of a lack of perfect smoothness.
More investigation is needed.



### Use of a single timing function spanning all CSS Animation keyframes.
Web-Animations allows use of a single timing function that applies to the entire iteration duration rather than keyframe-specific timing functions.
The `relative` keyword of `animation-composition` should be overloaded to also permit this, exclusively if necessary.
[https://www.w3.org/TR/web-animations-1/#the-effecttiming-dictionaries]
It would bring emergent physics to keyframe animations that behaved like springs or motion paths.
An example using Web-Animations shows one such usage. One rocketship, relatively animated with a single keyframe, gains angular velocity the more times the mouse is clicked. A second rocketship and subject of this has keyframes that create a semicircle path and uses relative animation to merge and shorten that path on multiple clicks.
[https://codepen.io/kvndy/pen/zeMMrJ]
As CSS Animations these would be difficult to create,
almost certainly requiring a preprocessor,
so the usefulness of this feature is debatable.



## Not fully realized

This usage pattern might benefit from the following ambitious changes,
which are not implemented in the experimental patch nor admittedly are their ramifications fully understood.
One possibility to enable this mode is a new CSS property similar to `will-change` that takes a list

### Prevent animations and transitions from supplying values for inheritance.
https://www.w3.org/TR/css-cascade-3/#inheriting
There would give a solution to the problem exposed by this example.
[https://dbaron.org/css/test/2015/transition-starting-1]
Mentioned in this message
[https://lists.w3.org/Archives/Public/www-style/2015Jan/0444.html]

Instead of Firefox behavior, or Chrome behavior which differs, 
the simplest solution is not inheriting calculated values for transitions,
rather inheriting specified values for transitions.
Parent and child each would animate only once on hover, and each only once when reversing, a benefit not restricted to relative animation.

It is not clear if this means only specified values are what is inherited,
or only if specified values are what is used in the case of animations and transitions.

Do any composited properties inherit?

This was discussed by L. David Baron, but I've misplaced the hyperlink:

  6. We don't want ridiculous numbers of transition events firing
     when an inherited property (e.g., color) is transitioned.  This
     should be handled just fine because we we get at most one for
     each element that actually has a 'transition-property' property
     set on it to transition that property.
D. Transitioning values are inherited by descendants just like any
   other values.  In other words, explicit 'inherit', or for
   inherited values, lack of any cascaded value on a descendant,
   leads to the transitioning value being inherited.  If, from (C),
   there is a transition simultaneously running on the descendant,
   that overrides the inherited value as any other specified value
   does.  This is the remainder of the explanation for how point (6)
   works, and also addresses point (3) in that the result of the
   transition inherits just like anything else does.
   
An interesting test is in Nightly layout/style/test/test_transitions.html
// Test that transitions on descendants start correctly when the
// inherited value is itself transitioning.  In other words, when
// ancestor and descendant both have a transition for the same property,
// and the descendant inherits the property from the ancestor, the
// descendant's transition starts as specified, based on the concepts of
// the before-change style, the after-change style, and the
// after-transition style.


### Prevent animations and transitions from contributing to intrinsic sizing.
https://www.w3.org/TR/css-sizing-3/#intrinsic
It should be possible that an animated layout change of children would not continuously update the size of a parent containing element.
Instead, the parent should instantly jump to a new size, using the destination values of the children.
If the containing element is to be animated, explicitly or by transition, it should happen based on that instant change, in hopes to avoid any layout thrashing.

### Prevent animations and transitions from contributing to scrollable overflow.
https://www.w3.org/TR/css-overflow-3/#scrollable
This is similar to the above but not just useful for relative animation.
Traditional animation would also benefit from not producing or extending scroll bars when a spring or bounce animation extends a child beyond its parent's edges.
This might require more granular control of the scroll bars, hopefully something exists, I'd rather not expand the API.

### Use destination values rather than calculated animated values when performing hit testing.
I am not sure if and where in the spec this is defined, but this would also be beneficial to non-relative animation.
Wobbly spring animations, when quickly toggled multiple times, can make elements move over top of others and potentially receive inadvertent clicks.
This might also be a solution to annoying hover transition animations that get trapped bouncing between two states which are never able to settle.

### Distinguish between animated and non-animated computed values in the spec.
I do know it is important to be able to use discrete non-animated values throughout the platform, wherever possible,
so animated interactions can designed and easily reasoned about.
I am not sure if this is necessary, I just think it might be helpful, at least to me.
The section on value processing might need to be altered for this.
https://www.w3.org/TR/css-cascade-3/#value-stages




### Relative animation for certain motion-path properties
There is plenty of discussion around different types.


## Features that for various reasons may not receive approval:


### A `perfect` timing function that resolves to `cubic-bezier(0.5, 0, 0.5, 1)`.

A timing function, `perfect` that resolves to `cubic-bezier(0.5, 0, 0.5, 1)` would also belong to the "pit of success" category.
Resistance to this might be similar to that of adding a new named color, which is explicitly mentioned... somewhere
[LINK]
Transition blending is accomplished by the timing function, which should not be considered a limitation as it utilizes existing primitives to accomplish emergent behavior.

A linear timing function may look bad with relative blending, but would also look bad without.
More dramatic timing functions may show their seams, and if they are required, then perhaps `special` interruption is not the right choice.

This was especially useful for Core Animation matrix scale operations.
It is not clear if it was needed because of a bug, but there was a visible discrepancy that the author has not been able to reproduce in the browser, and cannot be verified because the author no longer uses a Mac.






## Wildest dreams

### The Animated OM
It should be possible to animate objects in javascript using Web-Animations, instead of just values of CSS properties.
A limited subset of Web-Animations could be exposed, along with some Javascript specific additions.
The following is not well developed, and heavily influenced by Apple's Core Animation API.
Just spitballin':
`const obj = {};`
`Object.animator(obj)` to return an Animator object associated with `obj`
`obj` can't be an Animator or any object returned by Animator methods:
`specified` would be a getter that returns the model layer, a copy with all non-animated properties
`computed` would be a getter that returns the presentation layer, a copy with all animated properties
`animations` or `getAnimations()` a getter for all animations
`addAnimation( new KeyframeEffect(...), name );` add by optional name
`animate( new KeyframeEffect(...), name );` alternate add by optional name
`animationNamed(name)` animation getter by name
`removeAnimationNamed(name)` remove by name
`removeAnimation(animation)` remove by instance
`removeAllAnimations()` remove them all
This somehow needs types.
Default would be number but it would be nice to accept CSS types, maybe Typed OM.
`Object.animator(obj).types = { transform: "1s" };`
Or maybe it should map property names to types.
`Object.animator(obj).types = { variable: "transform", foo: "width", bar: "px" };`
`Object.animator(obj).properties = { variable: "1s" };` or `Object.animator(obj).transitions = { variable: "1s" };` to define transitions.
`types` would need to also accept transition duration, in milliseconds (I would prefer seconds) or a string with a combination number and unit.
`properties` would map from property name to something from the Typed OM or Web-Animations.
An object's animator would be associated with it by weak map or something else.
The reverse relationship could associate the animator to its object.
An object in that map can't be added to the other.
You must not get an animator from `Object.animator(Object.animator(obj))` or `Object.animator(Object.animator(obj).specified)` or `Object.animator(Object.animator(obj).computed)`
The properties of an object itself, not its animator, would return `computed` values during rAF but `specified` values during events.
During setTimeout it would return `specifed` values.
There wouldn't be a way to disable this other than removing types or properties by passing null or empty object.


This doesn't aim to restrict behavior by demanding types be enforced at "compile" time.
Rather it is to add behavior only in certain cases.
Animated values would only be returned in by an object with an animator in rAF or (a copy containing animated values?) from `specified`.
What one does with those values is up to them.
If you wanted, `Object.animator(obj).types` could be enforced on `obj` in some capacity, and you could ignore that you can add animations or trigger transitions on it.



### A type of discrete animation for arrays of objects, and a way to pass a sort function, written in Javascript, where equal values establish identity not equality.
Relative animation of discrete types in a set or unsorted array, which would use a sort function provided to the animation.
Or `Object.animator(obj).sort = { variable: function, foo: function, bar: function};` but only used on discrete types like `Set` and `Array`.
Only if property type is array or set?
`Object.animator(obj).types = { variable: "transform", foo: Number, piyo: "px", fuge: Array };`
Discrete transition of an array ["a", "b", "c", "d"] would occur when setting a new Array, not mutating, the objects property to ["a", "b", "d"],
if relative and transition duration and a sort function are provided, ["a", "b", "c", "d"] minus ["a", "b", "d"] 
results in ["c"] which would get added back to ["a", "b", "d"] and produce ["a", "b", "d", "c"] before sorting and  ["a", "b", "c", "d"] after, for the duration of the transition.
This value would only be available on the object during rAF or on a copy when returned from `computed` or `presentation` or `animated`

This would be useful for animating unmounting in DOM diffing libraries like React.
It might also be possible to animate the contents of NodeList, the ultimate dream.
Since the DOM is "live" a means would be needed to access a non-animated version for construction of relative animations,
which might needlessly complicate the API.
Like javascript objects, a way to sort would be needed, perhaps attributes on elements.





## But wait there's more

### Advanced CSS Transitions & Animations Proposal

The quite old and perhaps forgotten Advanced CSS Transitions & Animations Proposal
[https://www.w3.org/Graphics/fx/wiki/Advanced_CSS_Transitions_%26_Animations_Proposal]
as written was not aware of or condusive to relative animation's use of discrete underlying values.
In its section "Relative Keyframes" (which has a completely different and unrelated use of the word "relative") it describes potentially new keyframe rules.
These rules are `from()`, `to()`, and `prev()`.
Their `from()` is the current calculated animated value, and their `prev()` refers to an accumulator similar to that used in the javascript `reduce()` function, but instead for keyframe values.
More condusive to this proposal would be the following.
Their `from()` should be renamed `current()` or something similar.
Their `prev()` should be renamed `acc()` or `accumulated()` or something similar.
A different `prev()` or `previous()` should be the discrete non-animated value that was set prior to the new value whose setting triggered the new transition.
And maybe `destination()` could replace `to()` if need be.
This would be beneficial for both types of transitions enabled by the proposed `transition-interrupt` values `regular` and `special`.
This is not meant to be included in this proposal.

## Conclusion

### Anything else?
There may be additional changes needed to benefit this pattern which have not been addressed or considered.
Naming and feature bikeshedding or other comments and questions are encouraged and highly welcome.

### Thanks!
Let's usher in a new era of user interface animation for the web by implementing at least some of this proposal.




