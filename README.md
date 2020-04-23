
# Thoughts on an implementable path forward for Container Queries

# WARNING: This is only really half-written at this point and not yet ready for wide review.

The idea of Container Queries has been discussed widely,
such as in [WICG/container-queries](https://github.com/WICG/container-queries/)
and in [w3c/csswg-drafts#3852](https://github.com/w3c/csswg-drafts/issues/3852).
CSS authors want a way to use a feature like media queries on a *part* of the page.
Many of the use cases underlying this desire relate to things that are component-like,
that is, pieces of a web page that might be used in different webpages.
Authors wish to construct a component whose appearance responds to its size,
but they want this component to be a part of the webpage that contains it.

## Dependence on containment

The main reason this is theoretically difficult
is that this requires that styles depend on the size of the component,
yet given how CSS works, the styles in a component *influence* its size.
Arbitrarily breaking this loop would both give weird results
and would interfere with browser optimizations related to incremental relayout.

In [w3c/csswg-drafts#1031](https://github.com/w3c/csswg-drafts/issues/1031)
I proposed a way around this, based on the ideas in the
[CSS Containment](https://drafts.csswg.org/css-contain/) specification.
In particular, I believe that container queries should be possible
if an element has both `layout` containment
and has `size` containment in the axis used for the container queries.
The problem here is that we don't have a definition of `size` containment
in a single axis.

I believe that it's plausible to revise the definition of size containment
to define a variant that applies in a single axis
(which would probably be named `width`, `height`, `block-size`, or `inline-size`)
by revising the definition
(perhaps after being amended per
[w3c/csswg-drafts#4741](https://github.com/w3c/csswg-drafts/issues/4741))
to say "when calculating the size of the containing block in the given axis".
(What this means, underneath, probably needs to be defined per layout system,
as does the existing definition.)

### Containment in a single axis

To explain what containment in a single axis means,
we need to define it for each display type.

**TODO**: flesh this definition out more and go through the feedback on #4741
... and figure out if this whole thing has a chance of actually working!

#### ... `inline-size` containment for `display:block`

The computation of
[intrinsic sizes](https://drafts.csswg.org/css-sizing-3/#intrinsic-size)
in the inline axis assumes that the block has no contents.

There is no effect on layout since
block layout doesn't generally produce output in the inline dimension.

(It's not clear what the effect on
intrinsic sizes in the block axis
should be,
but it will perhaps be clearer once
[it's clearer](https://github.com/w3c/csswg-drafts/issues/2890)
what intrinsic sizes in the block axis are.)

#### ... `block-size` containment for `display:block`

First, the computation of
[intrinsic sizes](https://drafts.csswg.org/css-sizing-3/#intrinsic-size)
in the block axis (whatever they mean in the first place)
assumes that the block has no contents.

Second, the final block-size computation during layout
also assumes that the block has no contents.

#### ... `inline-size` containment for `display:table`

The computation of both
[intrinsic sizes](https://drafts.csswg.org/css-sizing-3/#intrinsic-size)
in the inline axis
and the inline-size computed during layout
assume that the table has no contents.

#### ... `block-size` containment for `display:table`

First, the computation of
[intrinsic sizes](https://drafts.csswg.org/css-sizing-3/#intrinsic-size)
in the block axis (whatever they mean in the first place)
assumes that the table has no contents.

Second, the final block-size computation during layout
also assumes that the table has no contents.
It's not clear to me what the resulting row block-sizes are,
but they're presumably the same as whatever happens for `size` containment.

#### ... `inline-size` containment for `display:table-cell`

same as for `display: block`

#### ... `block-size` containment for `display:table-cell`

same as for `display: block`
(with the caveat that the final block-size computation during layout
is more complicated)

#### ... containment for `display:flex` in the [main axis](https://drafts.csswg.org/css-flexbox-1/#main-axis)

#### ... containment for `display:flex` in the [cross axis](https://drafts.csswg.org/css-flexbox-1/#cross-axis)

#### ... `inline-size` containment for `display:grid`

#### ... `block-size` containment for `display:grid`

#### ... `inline-size` containment for [multi-column](https://drafts.csswg.org/css-multicol/#multi-column-container)

#### ... `block-size` containment for [multi-column](https://drafts.csswg.org/css-multicol/#multi-column-container)

#### ... more cases that need to be handled?

## Scope-based CSS cascading

The proposal I'm making here addresses the need for container queries,
but it also addresses a second need that I'd like to explain a little further.
It adds support for a different type of CSS cascading priority.

CSS cascading has the existing concept of
[specificity](https://drafts.csswg.org/selectors-4/#specificity-rules).
This concept often doesn't work very well for authors of CSS.
I believe that one of the reasons for this is that the specificity
is based on selectors that match anywhere in the tree,
and doesn't consider the proximity of the element selected by a selector
to the target of the selector.

Cascading Level 4 has added the concept of
[scope](https://drafts.csswg.org/css-cascade-4/#cascade-scope)
to the cascade level.
I *think* this concept is currently only accessible through Web Components
(although I'm not sure it's accessible through them).
(It was previously accessible through scoped `<style>` elements,
before they were removed.)
This proposal makes it accessible through CSS syntax.
The idea of **scope** is that a CSS rule can be associated with
the subtree of a particular element in the tree,
and those rules override CSS rules associated with any ancestor subtree.
This feature allows authors to say that a particular set of CSS rules
applies to a particular subtree of the document,
and those rules override rules for subtrees higher in the document.

Given that the syntax I'm proposing for container queries lends itself
to adding this feature at the same time, I propose to do so.

## Proposal

To address the need for container queries,
I'm proposing a feature that both addresses container queries
and also adds syntax for scope-based cascading to CSS.

```css
@container <selector> (<container-media-query>)? {
  // ... rules ...
}
```

The `<selector>` given as an argument to this `@container` rule
matches the container(s) to which the rules inside apply.
The rules can match only that container or its descendants.
These rules also apply in the CSS cascade at the
[scope](https://drafts.csswg.org/css-cascade-4/#cascade-scope)
of that container, so that they override rules for containers
higher in the document tree.
Thus this proposal is not only for container queries,
but also adds CSS syntax for scope-based cascading.

**Issue**: Is the parsing disambiguated sufficiently?
In other words,
are we OK assuming that when we hit an open parenthesis
(that's not part of an existing selector function),
it's the beginning of a media query and we're no longer parsing selectors?

The optional `<container-media-query>` argument to this `@container` rule
allows the rules inside it to match only if
the container has the necessary containment rules, and
the media query matches the container.
Note that this wording means that *negated* media queries
still won't match if the container doesn't have the necessary containment rules;
this means that when the containment is not as required,
then the entire rule will effectively be ignored.
(**Issue**: I think this seems preferable to treating the media queries as false
and then applying negation to them,
although I'm not sure.)

The `<container-media-query>` production is intended to be
a subset of media queries,
allowing only the
[Viewport/Page Dimensions Media Features](https://drafts.csswg.org/mediaqueries-4/#mf-dimensions)
(including `min-` and `max-` variants).
However, these queries would have modified definitions so that
their tests apply to the dimension of the container
rather than to the dimensions of the viewport or page.

The containment necessary for such a rule to operate would depend
on which media queries are used:
* if no media queries are used, no containment is necessary
* if media queries testing only the block-axis size are used, then both `layout` and `block-size` containment are needed
* if media queries testing only the inline-axis size are used, then both `layout` and `inline-size` containment are needed
* if media queries testing both the block-axis size and inline-axis size are used, then both `layout` and `size` containment are needed.

If the containment needed for any of the queries is not present,
then the queries are considered not to match,
and the rules inside are never applied.
(**Issue**: Is this the right choice, or should queries be allowed separately across toplevel commas?)

The selectors on the rules inside the `@container` rule
should probably match only *descendants* of the container element.
It would be possible to match the container element itself
for some properties but not others,
since some properties would influence the size of the container
that the query is testing against.
It's probably most straightforward to just say that the rules apply
only to descendants and not to the element itself.

**Todo**: selectors that reference the element that `<selector>` matches.  Should [`:scope`](https://drafts.csswg.org/selectors-4/#the-scope-pseudo) work here?

**Todo**: what rules are allowed inside `@container`?

## Performance characteristics

I believe the most substantial performance overhead of this feature
in an implementation that is well-optimized for it
should be that the implementation needs to
switch more between styling and layout.
The need for this switching may, in turn,
reduce opportunities for parallelism in both styling and layout
(although the later pass(es) might provide clearer opportunities for parallelism as well).

In particular,
this is because processing these queries requires that layout be done on the ancestors of the container
before styling can be done on its descendants.

This is still a substantial improvement on the workarounds used today,
which perform an *extra* pass of styling and layout on the descendants of the container
before (in at least some cases) correcting the styles to the final styles
and then performing style and layout again.

In other words, relative to a page that doesn't use this feature,
things may be slower due to the increased separation of passes.
But relative to a page that works around the lack of this feature in today's implementations,
things should be faster because the number of passes is (in general) the same
but duplicated work is avoided.

