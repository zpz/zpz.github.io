---
layout: post
title: "Some Resources for Learning Python"
excerpt_separator: <!--excerpt-->
tags: [python]
---

I do not try to make a comprehensive list.
I want to suggest a manageable list of really good resources.
<!--excerpt-->

Neither can I be comprehensive. 
This is just what I know are good or look good to me **for learners**.


## Prerequisites

If you are not yet a strong programmer, your code very likely suffers two problems:

1. There is no deliberate distinction between *library* and one-off *scripts*. Most work is (long) scripts.

2. Functions are long. A function grows on and on as you feel the need to do more and more.

The first problem is lack of *modularity* and *reusability*. The second is lack of *decomposition*.
The former is relatively easy to understand, and can be enforced to some extent.
The latter is harder.
I may say that, for a novice programmer, modularity and reusability is hard to do well,
but decomposition is hard to even start doing.
The two short reads below reinforce this basic idea of decomposition.

- [Decomposition & Style](https://cs.stanford.edu/people/nick/compdocs/Decomposition_and_Style.pdf) (pdf)
  by Nick Parlante.

- [Improve your code with atomic functions](https://www.codementor.io/seantullis/improve-your-code-with-atomic-functions-r6dt43fy7)
  by Sean Tullis.

If you have learned something by now, then **stop right here, and go work on these problems. There is no point in learning more language features**.

I would suggest you to always remember,

- [Don't live with broken windows](https://www.artima.com/intv/fixit2.html).
  Improve, fix, refactor as you go.
  Don't think an improvement is not worth doing because it looks "small", or it "works" for now.
  You won't be making such some improvements again and again.
  The quality of your code tomorrow is on the level of your code today.
  If you improve your code today, the code you write tomorrow will be on the level of your code today---only *the improved version*!---on first writing.


## Books and Book-form Websites

- [Clean Code: A Handbook of Agile Software Craftsmanship](https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882/ref=sr_1_2?keywords=clean+code&qid=1550970855&s=books&sr=1-2)
  by Rober Martin.
  Going through the TOC of the first few chapters convinced me this classic is a must-read.
  Since getting a hard copy, I have yet to read it... (sorry!)

- [Fluent Python](https://www.amazon.com/Fluent-Python-Concise-Effective-Programming/dp/1491946008/ref=pd_bxgy_14_img_2/139-2313814-4715406?_encoding=UTF8&pd_rd_i=1491946008&pd_rd_r=b326f27a-37d0-11e9-bebf-15ead0be056d&pd_rd_w=ziIQ5&pd_rd_wg=1W2O5&pf_rd_p=6725dbd6-9917-451d-beba-16af7874e407&pf_rd_r=YRYNS3KGYG3AG1QJCD1E&psc=1&refRID=YRYNS3KGYG3AG1QJCD1E)
  by Luciano Ramalho. This book can be read like an essay---it's not a reference book.

- [Python 3 Module of the Week](https://pymotw.com/3/)
  by Doug Hellmann. *Great* online resource.

- [Powerful Python](https://www.amazon.com/d/0692878971)
  by Aaron Maxwell

- [Python 201: Intermediate Python](https://www.blog.pythonlibrary.org/buy-the-book/python-201-intermediate-python/)
  by Michael Driscoll

- [Python Tricks: A Buffet of Awesome Python Features](https://www.amazon.com/Python-Tricks-Buffet-Awesome-Features/dp/1775093301/ref=sr_1_1?keywords=python+tricks&qid=1550970116&s=gateway&sr=8-1)
  by Dan Bader

- [Effective Python](https://www.amazon.com/Effective-Python-Specific-Software-Development/dp/0134034287/ref=pd_bxgy_14_img_3/139-2313814-4715406?_encoding=UTF8&pd_rd_i=0134034287&pd_rd_r=cc0e5595-37cf-11e9-9f14-0518718f1dc0&pd_rd_w=akYTF&pd_rd_wg=A9hOO&pf_rd_p=6725dbd6-9917-451d-beba-16af7874e407&pf_rd_r=GHB0PYNPWGEZ2HTKJRAN&psc=1&refRID=GHB0PYNPWGEZ2HTKJRAN)
  Brett Slatkin

- [Python Course](https://www.python-course.eu)
  by Bernd Klein. I read its section on multiple inheritance and got to understand Python's Method Resolution Order (MRO) in that context.



## Guidelines

- [The Hitchhiker's Guide to Python](https://docs.python-guide.org)

- [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html).
  I only remember reading the [naming](https://google.github.io/styleguide/pyguide.html#316-naming) section, but I believe the entire guideline is good.

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
  One great quote from his talk I remember is something like "Do not confuse 'simple' with 'familiar'".

- and many more...



## Blogs and Assortments

- [Real Python](https://realpython.com)

- [PyCoder's Weekly](https://pycoders.com).
  Entire archive of past issues are [here](https://pycoders.com/issues).
  It's a good idea to subscribe by email or on Twitter.
  This one newsletter guarantees you stay up-to-date with Python and keep learning.

- [Full Stack Python](https://www.fullstackpython.com/table-of-contents.html).
  A lot of links. It's a curated index of resources with descriptions.

- [Trey Hunner](https://treyhunner.com/blog/archives/)

- [Jeff Knupp](https://jeffknupp.com/blog/archives/)

- [Dan Bader](https://dbader.org/blog/)


## Specific Topics

- [Google style Python docstrings](https://sphinxcontrib-napoleon.readthedocs.io/en/latest/example_google.html).
  This is the style I have adopted loosely.
  But, it is *by far* more important to have useful *doc*,
  regardless of the format.

- [A Guide to Python's Magic Methods](https://rszalski.github.io/magicmethods/) by Rafe Kettler.

- [Python Design Patterns](https://github.com/faif/python-patterns)

- [String formatting](https://pyformat.info/)

- [*args, **kwargs, and more](https://treyhunner.com/2018/10/asterisks-in-python-what-they-are-and-how-to-use-them/)

- [Functions and decorators](https://stackoverflow.com/a/1594484)

- [script vs module, and relative imports](https://stackoverflow.com/a/14132912)

- [Iterators and generators](https://stackoverflow.com/a/231855)

- [classmethod vs staticmethod](https://stackoverflow.com/a/12179752)

- and many more...


Finally, I think this reminder is a good one to heed

- [Don't learn a programming language, solve a problem instead](https://medium.com/datadriveninvestor/dont-learn-a-programming-language-solve-a-problem-instead-654f6bbfb573).