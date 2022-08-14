---
layout: post
title: "Some Resources for Learning Python"
excerpt_separator: <!--excerpt-->
tags: [python]
---

Here I'm not trying to make a comprehensive list.
I just want to suggest a *manageable* list of really good resources.
<!--excerpt-->

Neither can I be comprehensive. 
What follows is just what I know are good or look good to me **for learners**.


## Prerequisites

A novice (or nonprofessional) programmer's code often suffers two problems:

1. There is no deliberate distinction between *library* and one-off *scripts*. Most work is (long) scripts.

2. Functions are long. A function grows on and on as you feel the need to do more and more.

The first problem is lack of *modularity* and *reusability*. The second is lack of *decomposition*.
While both aspects require a lot of experience to master,
I feel the latter is even harder to start for a new programmer.
The two short reads below reinforce this basic idea of decomposition.

- [Decomposition & Style](https://cs.stanford.edu/people/nick/compdocs/Decomposition_and_Style.pdf) (pdf)
  by Nick Parlante.

- [Improve your code with atomic functions](https://www.codementor.io/seantullis/improve-your-code-with-atomic-functions-r6dt43fy7)
  by Sean Tullis.

If you have learned something by now, then **stop right here, turn to your code and work on these problems. There is no point in learning more language features**.

I also suggest we always remember,

- [Don't live with broken windows](https://www.artima.com/intv/fixit2.html).
  Improve, fix, refactor as you go.
  Don't think an improvement is not worth doing because it looks "small", or it "works" for now.
  You won't be making the same small improvements again and again.
  The quality of your code tomorrow is on the level of your code today.
  If you improve your code today, the code you write tomorrow will be on the level of your code today---only *the improved version*!---on first writing.


## Books and Book-form Websites

- [Clean Code: A Handbook of Agile Software Craftsmanship](https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882/ref=sr_1_2?keywords=clean+code&qid=1550970855&s=books&sr=1-2)
  by Rober Martin.
  Just going through the TOC of the first few chapters was enough to convince me that this classic is a must-read.
  Since getting a hard copy, I have read it... 0 times (sorry!)

- [Fluent Python](https://www.amazon.com/Fluent-Python-Concise-Effective-Programming/dp/1491946008/ref=pd_bxgy_14_img_2/139-2313814-4715406?_encoding=UTF8&pd_rd_i=1491946008&pd_rd_r=b326f27a-37d0-11e9-bebf-15ead0be056d&pd_rd_w=ziIQ5&pd_rd_wg=1W2O5&pf_rd_p=6725dbd6-9917-451d-beba-16af7874e407&pf_rd_r=YRYNS3KGYG3AG1QJCD1E&psc=1&refRID=YRYNS3KGYG3AG1QJCD1E)
  by Luciano Ramalho. This book can be read like an essay---it's not a reference book.

- [Python 3 Module of the Week](https://pymotw.com/3/)
  by Doug Hellmann. *Great* online resource.

- [Powerful Python](https://www.amazon.com/d/0692878971)
  by Aaron Maxwell

- [Python 201: Intermediate Python](https://www.blog.pythonlibrary.org/buy-the-book/python-201-intermediate-python/)
  by Michael Driscoll.

  I think learning intermediate-level material is most useful---it is within everyone's capability, is immensely and immediately empowering, yet stops before the return-on-investment diminishes.

- [Python Tricks: A Buffet of Awesome Python Features](https://www.amazon.com/Python-Tricks-Buffet-Awesome-Features/dp/1775093301/ref=sr_1_1?keywords=python+tricks&qid=1550970116&s=gateway&sr=8-1)
  by Dan Bader

- [Effective Python](https://www.amazon.com/Effective-Python-Specific-Software-Development/dp/0134034287/ref=pd_bxgy_14_img_3/139-2313814-4715406?_encoding=UTF8&pd_rd_i=0134034287&pd_rd_r=cc0e5595-37cf-11e9-9f14-0518718f1dc0&pd_rd_w=akYTF&pd_rd_wg=A9hOO&pf_rd_p=6725dbd6-9917-451d-beba-16af7874e407&pf_rd_r=GHB0PYNPWGEZ2HTKJRAN&psc=1&refRID=GHB0PYNPWGEZ2HTKJRAN)
  by Brett Slatkin

- [Python Course](https://www.python-course.eu)
  by Bernd Klein. I once read its section on multiple inheritance and got to understand Python's Method Resolution Order (MRO) in that context. This made me believe that the site goes to decent depth.

- [Python Patterns](https://python-patterns.guide)
  by Brandon Rhodes.



## Guidelines

- [The Hitchhiker's Guide to Python](https://docs.python-guide.org)

- [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html).

- [PEP 8 -- Style Guide for Python Code](https://www.python.org/dev/peps/pep-0008/)



## Talks

- Many talks by [Raymond Hettinger](https://www.youtube.com/results?search_query=raymond+hettinger+python),
  such as [this one](https://www.youtube.com/watch?v=OSGv2VnC0go).

- Many talks by [Dan Bader](https://dbader.org/python-videos/). I have not watched his talks. But given his expertise and Python training career, and the topics of talks, I believe it's quality learning material.

- Many talks by [David Beazley](https://www.youtube.com/results?search_query=david+beazley+python)
  (*advanced* but entertaining), such as [this one](https://www.youtube.com/watch?v=lyDLAutA88s).

- Many talks by [Ned Batchelder](https://www.youtube.com/results?search_query=ned+batchelder+python),
  such as [this one](https://www.youtube.com/watch?v=EnSu9hHGq5o)
  and [this one](https://www.youtube.com/watch?v=FxSsnHeWQBY).

- Some talks by [Brandon Rhodes](https://www.youtube.com/results?search_query=brandon+rhodes+python),
  such as [this one](https://www.youtube.com/watch?v=DJtef410XaM).

- Some talks by [Bob Martin](https://www.youtube.com/results?search_query=uncle+bob+martin+programming),
  such as [this one](https://www.youtube.com/watch?v=TMuno5RZNeE).

- Some talks by [Venkat Subramaniam](https://www.youtube.com/results?search_query=Venkat+Subramaniam),
  such as [this one](https://www.youtube.com/watch?v=llGgO74uXMI).
  One great quote from his talk I remember is something like 

  > Do not confuse "simple" with "familiar".

  One common scenario is that when I suggest a simpler (usually more elegant, general, and extensible) approach to some code block, the other person resists (or silently chooses to reject) on the ground that the suggested approach is needlessly "hard" or "un-natural" or "complicated". In fact it's none of them; it's *unfamiliar* to her.

  "Recursion" comes to mind.

- and many more...



## Blogs and Assortments

- [Real Python](https://realpython.com)

- [PyCoder's Weekly](https://pycoders.com).
  Entire archive of past issues are [here](https://pycoders.com/issues).
  It's a good idea to subscribe by email.
  This one newsletter helps you stay up-to-date with Python and keep learning.

- [Full Stack Python](https://www.fullstackpython.com/table-of-contents.html).
  A lot of links. It's a curated index of resources with organization and descriptions.
  More useful after you have learned the basics.

- [Trey Hunner](https://treyhunner.com/blog/archives/)

- [Jeff Knupp](https://jeffknupp.com/blog/archives/)

- [Dan Bader](https://dbader.org/blog/)

- In a previous blog I suggested [Some High-level Tips for Python Projects](http://zpz.github.io/blog/python-project-tips/).


## Specific Topics

- [Write better Python functions](https://jeffknupp.com/blog/2018/10/11/write-better-python-functions/)

- [Google style Python docstrings](https://sphinxcontrib-napoleon.readthedocs.io/en/latest/example_google.html).
  This is the style I have adopted loosely.
  But, it is *by far* more important to have useful *doc*,
  regardless of the format.

  I also suggest using *type annotations* in place of some documentation.
  [Brett Cannon apparently agrees](https://snarky.ca/my-experience-with-type-hints-and-mypy/).

- [A Guide to Python's Magic Methods](https://rszalski.github.io/magicmethods/) by Rafe Kettler.

- [Python Design Patterns](https://github.com/faif/python-patterns)

- [String formatting](https://pyformat.info/)

- [*args, **kwargs, and more](https://treyhunner.com/2018/10/asterisks-in-python-what-they-are-and-how-to-use-them/)

- [Functions and decorators](https://stackoverflow.com/a/1594484)

- [script vs module, and relative imports](https://stackoverflow.com/a/14132912)

- [Iterators and generators](https://stackoverflow.com/a/231855)

- [classmethod vs staticmethod](https://stackoverflow.com/a/12179752)

- and many more...

<br>

Finally, I think this reminder is a good one to heed

- [Don't learn a programming language, solve a problem instead](https://medium.com/datadriveninvestor/dont-learn-a-programming-language-solve-a-problem-instead-654f6bbfb573).
