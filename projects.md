---
layout: projects
title: Projects
short_title: projects
permalink: /projects/
---

# Projects

## Adaptive Anchored Inversion

<a href="https://arxiv.org/abs/1409.2221">Adaptive Anchored Inversion for Gaussian random fields using nonlinear data</a>
is my final (and best) academic publication.

It is a computation-intensive statistical method tackling a problem
that is a longtime fundamental challenge in several disciplines.
I conceived the approach and developed it over four years' of intense work,
undertaking three or four overhauls to the methodology,
each time lifting it to a new level.
(Academics do not use the term "invent", but this is just that.)
I developed several software packages along the way to implement the method.
Results of example applications are very compelling.

Some strategies in this work are very familiar and important to Machine Learning, including:

- Bayesian statistics
- iterative algorithm
- kernel density estimation (I have a separate publication on this as a by-product of the main-line of research)
- dimension reduction
- automatic adaptation
- loss function

This is a piece of research work that I am proud of.
I believe it continues to be a significant contribution to the field of its subject matter.
I try to make the methodology useful, or at least exposed, to others via a web service.
The idea is that the modeling code runs behind a web service,
whereas the user interacts with it through web UI or API calls.
The web site has some management and demonstration capabilities, including dynamic plots.

Example client code resides in the
<a href="https://github.com/anchored-inversion">Github organization "anchored-inversion"</a>.

This whole thing is functional on a proof-of-concept level but not polished.
Progress is slow due to lack of time.


## Open-source Coding Projects

I do not have any large software project in the open.
Below are a few comparatively more interesting items in my <a href="https://github.com/zpz">Github account</a>.

- <a href="https://github.com/zpz/utilities.cc/blob/master/include/zpz/avro.h">avro.h</a>:
a C++ utility for navigating in an Apache Avro file,
making heavy use of an advanced C++ language feature called "variadic template".

- <a href="https://github.com/zpz/datamill">datamill</a>:
a watered-down Python package for unified feature engineering, with Cython extensions for speed.

