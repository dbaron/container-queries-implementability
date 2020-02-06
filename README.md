
# Thoughts on an implementable path forward for Container Queries

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

TODO: flesh this definition out more and go through the feedback on #4741

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

* if appropriate, tests media queries
  * what if the container doesn't have the necessary containment?  probably always false?  Implying the containment would be too weird, I think.
* container-media-query is a subset of media queries, allowing only the [Viewport/Page Dimensions Media Features](https://drafts.csswg.org/mediaqueries-4/#mf-dimensions) (including `min-` and `max-` variants)

Issues:
* Is the parsing disambiguated sufficiently?  In other words, are we OK assuming that when we hit an open parenthesis (that's not part of an existing selector function), it's the beginning of a media query and we're no longer parsing selectors.
