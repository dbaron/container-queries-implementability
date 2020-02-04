
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

## Proposal

```css
@container <selector> (<container-media-query>)? {
  // ... rules ...
}
```

* always establishes a new cascading context
  * so it's not just for container queries, but also depth-based cascading overrides that have more power than specificity
* if appropriate, tests media queries
  * what if the container doesn't have the necessary containment?  probably always false?  Implying the containment would be too weird, I think.
* container-media-query is a subset of media queries, allowing only the [Viewport/Page Dimensions Media Features](https://drafts.csswg.org/mediaqueries-4/#mf-dimensions) (including `min-` and `max-` variants)
