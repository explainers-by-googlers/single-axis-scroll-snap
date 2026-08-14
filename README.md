# Explainer: Scroll Snap for Single-Axis Scroll Containers

This proposal outlines the design and specification extensions required to enable **CSS Scroll Snap** to support **[single-axis scroll containers](https://github.com/explainers-by-googlers/single-axis-scroll-containers)** (`overflow: auto clip` / `overflow: clip auto`).

## Introduction

CSS Overflow 4 introduced **[single-axis scroll containers](https://github.com/explainers-by-googlers/single-axis-scroll-containers)**, allowing elements to establish a scroll container in a single dimension (e.g., `overflow-x: auto; overflow-y: clip;`). While traditional 2D scroll containers (and `overflow: hidden`) establish scrolling contexts across both axes, a single-axis scroll container only establishes a scroll container along its scrollable axis and clips along the other.

CSS Scroll Snap was originally designed around 2D scroll containers, where any scroll container captures all descendant scroll snap areas (`scroll-snap-align`) in both dimensions. With the introduction of single-axis scroll containers, web developers can now compose independent scrollers along orthogonal axes.

This proposal extends CSS Scroll Snap with **per-axis scroll snap area propagation**: a single-axis scroll container consumes snap requirements for its scrollable axis, while snap requirements along the non-scrollable (clipped) axis propagate to ancestor scroll containers.

---

## Goals

- Extend CSS Scroll Snap to support single-axis scroll containers (`overflow: auto clip` / `overflow: clip auto`).
- Enable snap areas to be consumed by the single-axis container in its scrollable axis while propagating along the clipped axis to ancestor scroll containers.
- Align with how `position: sticky` tracks ancestor scroll containers independently per axis in [single-axis scroll containers](https://github.com/explainers-by-googlers/single-axis-scroll-containers).
- Support resolution of logical snap alignments across varying writing modes between inner and outer containers.
- Maintain existing behavior for standard 2D scroll containers and containers with `overflow: hidden`.

---

## Non-goals

- Changing the behavior of standard 2D scroll containers or `overflow: hidden` (which remain 2D scroll containers capable of programmatic scrolling in both axes).
- Introducing new CSS properties or syntax for `scroll-snap-align` or `scroll-snap-type`.

---

## Background and Motivation

The introduction of [single-axis scroll containers](https://github.com/explainers-by-googlers/single-axis-scroll-containers) updated the behavior of sticky positioning (`position: sticky`), allowing sticky elements (such as table headers or sidebars) to track different ancestor scroll containers independently for each axis. For example, a sticky header can stick to a vertical ancestor scroller while freely sliding within an inner horizontal single-axis container.

We want to apply a similar per-axis model to **CSS Scroll Snap**. When an element with `scroll-snap-align` resides inside a single-axis scroll container, the container should only consume snap points for its active scrollable axis, allowing snap requirements for the clipped axis to propagate to outer ancestor scroll containers.

---

## Proposal

### Consuming and Propagating Snap Axes

- **Axis Consumption**: When an element specifies `scroll-snap-align`, its nearest ancestor scroll container consumes only the snap alignment for the axes along which it can actually scroll.
- **Axis Propagation**: For any axis where the container does not scroll (i.e. clipped via `overflow: clip`), the snap alignment along that axis remains unconsumed and propagates up the ancestor tree to the next scroll container capable of scrolling that axis.
- **Scroll Snap Type**: If `scroll-snap-type` on a single-axis container specifies snapping along an unscrollable axis (such as `scroll-snap-type: both` on an `overflow-x: auto; overflow-y: clip;` element), the non-scrollable axis is ignored on that container, allowing descendant snap areas for that axis to propagate.

### Resolving Axes with Writing Mode

Because `scroll-snap-align` specifies logical dimensions (`inline` and `block`), whereas scroll containers operate on physical dimensions ($X$ and $Y$):

1. **Initial Resolution**: The logical alignments of a snap area are resolved into physical axes using the [writing mode](https://www.w3.org/TR/css-writing-modes-4/#writing-mode) and text direction of the nearest ancestor scroll container.
2. **Physical Axis Hand-off**: Any physical snap axis not consumed by the nearest container propagates up the tree as a physical snap requirement. An ancestor scroll container that scrolls along that physical axis captures and applies the snap alignment directly in its physical coordinate space, regardless of differences in writing mode or logical orientation across the scroller hierarchy.

---

## Use Case: Multi-Carousel Feed ([Demo](https://jsfiddle.net/lcdavid94/m4zvnxLo/))

A pervasive pattern in media, streaming, and e-commerce websites is a vertical feed containing multiple horizontal carousels (e.g., "Trending Now", "Continue Watching", "Recommended").

Authors want:
1. **Horizontal pagination within each carousel**: Scrolling left/right snaps horizontally to individual item cards (`scroll-snap-type: x mandatory` on each carousel).
2. **Vertical section snapping on the page**: Scrolling up/down snaps vertically so each carousel section locks flush into the viewport (`scroll-snap-type: y mandatory` on the outer feed).

With per-axis scroll snap area propagation:
- Each horizontal carousel consumes the horizontal snap alignment (`start`) to paginate through its items.
- The vertical snap alignment (`start`) continues propagating up to the outer feed, enabling the outer page scroller to snap vertically to the top of each carousel section.

---

## Compatibility

The behavior of existing web pages may change if they currently define `scroll-snap-align` along both axes (e.g., `scroll-snap-align: start`) on descendant elements inside a single-axis scroll container (e.g., `overflow-x: auto; overflow-y: clip;`), and have an ancestor scroll container that scrolls along the clipped axis (`overflow-y: auto`).

Previously, the descendant's vertical snap alignment was ignored entirely because the single-axis container captured and dropped it. Under this proposal, that snap area will now propagate to the ancestor vertical scroll container and affect its snap positions.

Although such configurations are generally not quite correct to begin with (authors usually did not intend to declare snap alignments on an axis that is unscrollable without expecting it to participate in scrolling), authors who want to prevent an element from snapping in an ancestor scroller can explicitly set snap alignment to `none` along that axis (e.g., `scroll-snap-align: start none`).
