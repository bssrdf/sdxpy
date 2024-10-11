---
template: slides
title: "An HTML Validator"
---

## The Problem

-   We generate HTML pages to report experiments

-   Want to be sure they have the right structure
    so that people can get information out of them reliably

-   Learning how to do this prepares us for checking code

---

## HTML as Text

-   HTML documents contain [%g tag "tags" %] and text

-   An [%g tag_opening "opening tag" %] like `<p>` starts an element

-   A [%g tag_closing "closing tag" %] like `</p>` ends the element

-   If the element is empty,
    we can use a [%g tag_self_closing "self-closing tag" %] like `<br/>`

-   Opening and self-closing tags can have [%g attribute "attributes" %]

    -   Written as `key="value"` (with some variations)

-   Tags must be properly nested:
    `<a><b></a></b>` is illegal

---

## HTML as a Tree

-   HTML elements form a [%g tree "tree" %] of [%g node "nodes" %] and text

-   The object that represent these make up the [%g dom "Document Object Model" %] (DOM)

[% figure
   slug="check-dom-tree"
   img="dom_tree.svg"
   alt="DOM tree"
   caption="Representing HTML elements as a DOM tree."
%]

---

## From Text to DOM

-   Real-world HTML is often messy

-   Use [Beautiful Soup][beautiful_soup] to parse it

-   Nodes are `NavigableString` (for text) or `Tag` (for element)

-   `Tag` nodes have properties `name` and `children`

[%inc parse.py mark=main %]

---

## From Text to DOM

[%inc parse.py mark=text %]
[%inc parse.out %]

---

## Recursion

[%inc parse.py mark=display %]

-   Text nodes don't have children

-   `for child in node` loops over children of element nodes

---

## Attributes

-   A dictionary `node.attrs`

-   Can be single-valued or multi-valued

[%inc attrs.py mark=display %]

---

## Attributes

[%inc attrs.py mark=text %]
[%inc attrs.out %]

---

## Build a Catalog

-   What kinds of children do elements have?

    -   `<tr>` (table row) should only appear inside `<table>` or `<tbody>`

-   Recurse through DOM tree

[%inc contains.py mark=recurse %]

---

## Build a Catalog

[%inc page.html %]

---

## Build a Catalog

[%inc contains.out %]

---

## The Visitor Pattern

-   A [%g visitor_pattern "visitor" %] is a class
    that knows how to get to each element of a data structure

-   Derive a class of our own that does something for those elements

-   When we recurse, allow separate handlers for entry and exit

    -   Useful for things like pretty-printers

---

## The Visitor Pattern

[%inc visitor.py mark=visitor %]

-   `pass` rather than `NotImplementedError`
    because many uses won't need all these methods

---

## Catalog Reimplemented

[%inc catalog.py mark=visitor %]

-   Only a few lines shorter than the original

-   But the more complicated the data structure is,
    the more helpful the Visitor pattern becomes

---

## Catalog Reimplemented

[%inc catalog.py mark=main %]

---

## Visitor in Action

[% figure
   slug="check-visitor"
   img="visitor.svg"
   alt="Visitor pattern order of operations"
   caption="Visitor checking each node in depth-first order."
%]

---

## Find Style Violations

-   Compare each parent-child combination against a [%g manifest "manifest" %]

[%inc manifest.yml %]

---

## Find Style Violations

[%inc check.py mark=check %]

---

## Running the Checker

[%inc check.py mark=main %]

---

## Results

[%inc check.out %]

-   Because content is supposed to be inside a `section` tag,
    not directly in `body`

-   And we're not supposed to *emphasize* words in lists

---

<!--# class="summary" -->

## Summary

[% figure
   slug="check-concept-map"
   img="concept_map.svg"
   alt="Concept map for checking HTML"
   caption="Concept map."
%]
