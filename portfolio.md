---
layout: page
title: Technical Mini-Portfolio
permalink: /portfolio/
---

Here I list a few resources that showcase some of my technical skills,
which may not be visible without these pointers.

# Coding

I'll point to my occasional
<a href="{{ site.baseurl }}/blog">blog posts</a>.
There are a few longer pieces there.
To the right audience, each is an interesting story.
For example:


- <a href="{{ site.baseurl }}/speeding-up-python-with-cython-appetizer/">Speeding up Python with Cython: an Appetizer</a>
shows how I took two lines of regular Python code
and pushed the limit of performance in multiple iterations,
eventually achieving a 147x speed-up.
It's a story of relentless focus on bottleneck with the help of a profiler.

- <a href="{{ site.baseurl }}/overloading-equality-operator-in-cpp-class-hierarchy/">Overloading `operator==` for a C++ Class Hierarchy</a>:
a small study of a C++ language detail,
having to do with inheritance, runtime polymorphism (virtual functions), and design of class interface.

- <a href="{{ site.baseurl }}/python-cpp-pybind11-stringview/">Python, C++, Pybind11, and string_view</a>:
a deep dive into an elusive issue
caused by the interplay between Python's garbage collection, object lifetime,
peculiarities of string objects, and Python/C++ interoperation.

- <a href="{{ site.baseurl }}/poor-mans-CD-system-using-Docker/">A Poor Man's Continuous Deployment System Using Docker</a>:
using Docker and shell scripts to achieve a naive CD workflow---execution environments
in sync between development and deployment machines, automatically updated at code push,
with logging and some configuration handled transparently.

- <a href="{{ site.baseurl }}/talking-to-spark-from-python-via-livy/">Talking to Spark from Python via Livy</a>:
treating Spark as a RESTful service, so to decouple Spark and non-Spark environments,
while being able to cross the boundary with ease.

- <a href="{{ site.baseurl }}/athena-ctas/">"Insert Overwrite Into Table" with Amazon Athena</a>:
a data engineering utility that enhances the usability and power of Amazon Athena.

I do not have any large software project in the open.
I'll point to a couple comparatively more interesting items in my Github account:

- <a href="https://github.com/zpz/utilities.cc/blob/master/include/zpz/avro.h">avro.h</a>:
a C++ utility for navigating in an Apache Avro file,
making heavy use of an advanced C++ language feature called "variadic template".

- <a href="https://github.com/zpz/datamill">datamill</a>:
a watered-down Python package for unified feature engineering, with Cython extensions for speed.

- <a href="https://github.com/anchored-inversion/client.R">anchored-inversion/client.R</a>:
an example client package for programmatically talking to a web service
(i.e. calling RESTful API) to run a model.
The web service is developed by me to make the modeling method I invented
(see the publication below) usable by others---the model implementation is on the server side.
The web service site has some UI and dynamic plot rendering.
This whole thing is functional but not polished.


# Machine learning

- <a href="{{ site.baseurl }}/gradient-boosting-tree-for-binary-classification/">Understanding Gradient Boosting Tree for Binary Classification</a>: this is an expository piece that might be of interest.

- <a href="https://arxiv.org/abs/1409.2221">Adaptive Anchored Inversion for Gaussian random fields using nonlinear data</a>.
This is my final academic publication:

  It is a computation-intensive statistical method tackling a problem
that is a longtime fundamental challenge in several disciplines.
I conceived the approach and developed it over four years' of intense work,
experiencing three or four overhauls to the methodology,
each time lifting it to a new level.
(Academics do not use the term "invent", but this is just that.)
I developed several software packages along the way to implement the method.
Results of example applications are very compelling.
This is a piece of work I'm proud of.

  Some strategies in this work are very familiar and important to Machine Learning, including:

  - Bayesian statistics
  - iterative algorithm
  - kernel density estimation (I have a separate publication on this as a by-product of the main-line of research)
  - dimension reduction
  - automatic adaptation
  - loss function
