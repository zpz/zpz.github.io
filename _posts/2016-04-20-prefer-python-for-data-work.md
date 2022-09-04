---
layout: post
title: Why I Prefer Python to R for Data Work
excerpt_separator: <!--excerpt-->
tags: [R, data-science, Python]
---

I first encountered S-PLUS (a commercial distribution of S---R's parent) in 2000,
and used it for statistics coursework for a few years.
From 2004 through 2011, I used R daily and intensively for implementing my research
on statistical methodologies.
<!--excerpt-->
At its core, R is a decent functional (as opposed to object-oriented) language.
I have made use of some more "serious" sides of it including

- building packages
- some functional-style programming
- formal documentation generation by `roxygen2`
- interop with `C` and `Fortran` (I implemented small number-crunching components in `C` and `Fortran` as part of the packages and called them from `R`)
- profiling and speed tuning
- parallelism (light use of R's parallel packages)
- advanced graphics (customizing `lattice` graphics)

R has been created, developed, maintained, and used by statisticians.
It is a (or the) language for statistics.
This decade has seen growing use of the language outside of the statistician community proper.
In the increased use, the use case remains statistics---or more precisely,
the statistics functions in core R and the several thousand R packages developed by statisticians.
The increased use is not, in my opinion,
about the language itself or a vision that the language will grow beyond the statistics domain.

I have stopped active use of R and switched to Python for "data and modeling" work.
Here I will mainly talk about the weakness of R,
and only briefly about the strengths of Python because the latter is widely known.

## What R Has That Python Does Not

Two things:

### Specialized or new statistical methods

R has reached total monopoly in the statistics community.
It is largely safe to say that, nowadays, every computational study published in statistical journals
is backed by R implementation.

That's the 'cutting-edge' part. But the method being 'new' is not a requirement.
The variety of statistical methods implemented in R is likely unmatched by any other language, or all other languages combined.

### More authoritative implementation of classical and mature statistical methods

For classical and mature statistics like regression, hypothesis testing, and so on,
the R implementation can be looked upon as the authority,
because these routines have been created and scrutinized by numerous statistics professors.
I wouldn't be surprised to learn that R's implementation of a standard method
is more complete, and even more *correct* in corners, than the counterpart in another language.

Some may want to cite R graphics as a strength.
I do not think Python is weaker in this area.

## R's Weak Points

The two strengths listed above are important to researchers of statistics.
Whether (or how much) the strength in a particular case matters to industrial use
is another issue.

Now some random observations on R's weak sides.

### R's language design and implementation has some glaring issues

I will not brag about this. Every language has lovers and haters.

There is a rare examination of the R language by academics:
[Evaluating the Design of the R Language](http://r.cs.purdue.edu/pub/ecoop12.pdf)
by CS professor Jan Vitek and his students from Purdue.
I will quote the main points from this study's "Conclusions" section:

> As a language, R is like French;
> it has an elegant core, but every rule comes with a set of ad-hoc exceptions that directly contradict it.
>
> Many of its features are geared towards speeding up interactive data analysis....
> But, it is also clear that these very features hamper the development of larger code bases.
> For robust code, one would like to have less ambiguity and would probably be willing to pay for that by more verbose specifications, perhaps going as far as full-fledged type declarations.
> So, R is not the ideal language for developing robust packages.
>   
> One of the most glaring shortcomings of R is its lack of concurrency support.
>   
> The object-oriented side of the language feels like an afterthought.
> The combination of mutable objects without references or cyclic structures is odd and cumbersome.
> The simplest object system provided by R is mostly used to provide printing methods for different data types.
> The more powerful object system is struggling to gain acceptance.
>   
> The current implementation of R is massively inefficient.

### R is driven by amateur programmers instead of professional computer scientists

Currently, the [Board of the R Foundation](https://www.r-project.org/foundation/members.html) has 32 members. I looked up every member's profile and found:

- 21 are professors of statistics (3+ are retired)
- 1 is professor in social science (but in fact a statistician)
- 1 is professor in science
- 3 are in academic research institutions (heavy in statistics)
- 2 are in research arms in industry (heavy in statistics)
- 2 are statistical consultants
- 1, Dirk Eddelbuettel, seems to be a software freelancer (PhD in econometrics)
- 1, Hadley Wickham, a little hard to characterize but needs no introduction (PhD in statistics)

The first concern of most R users is to get the research done and paper published.
The second concern may be to make the software usable and used by others.
This is the accepted order of priorities for their profession.

In comparison to the numerous heavy-weight long-stays of Python packages,
how many R packages

- are on Github?
- have a **team** of contributors?
- have extensive tests?
- have good documentation?
- have a website?
- have been in consistently active development for more than 4 years?
- are the generally recognized go-to choice, in companies, for their problem domain?
- have a "large" code base?

A few years ago my research used an R package, which is the result of substantial research on a very specialized statistical problem.
I wanted to extract part of the package and adapt it in some way.
I made a couple attempts at understanding the code but soon gave up,
because I could not penetrate the `C` code that requires a `C++` compiler (may I call it spaghetti?).
To this date I still don't know of an alternative to that package.
BTW the author is a professor, and a member of the Board.

### R is hard to learn

Yes, you got started following one of the many tutorials.
But I'm talking about something else, a perspective I haven't seen mentioned.

I don't think it would be controversial to say a nice language should have a small core, and an adequate (or extensive) standard library.
With reasonable effort, you learn what the core has.
With more effort, you learn a higher level of topics in the core.
You pick up standard libraries individually as needed, and get thorough about the ones you use a lot.

R is the opposite. It has a large core (called "base") and unusual standard library (R does not use this terminology).
The standard libraries (which come with R releases bundled) consist of

- compiler
- datasets    (example datasets; not data processing utilities)
- graphics
- grDevices   (graphics)
- grid        (graphics)
- methods     (an unconventional OOP system struggling to be noticed by regular users)
- parallel
- splines
- stats
- stats4
- tcltk

We see that these libraries are not like the standard library of, say Python.
They do not form a smallish and orthogonal set of building blocks
for software development in R.
Most of them are quite specialized: you need them or not at all,
depending on your problem domain.
The logic in their choice and organization is not obvious to me.
(I think functions from these libraries are available automatically without namespace import, but I'm not sure.)

Now on to the `base` package.
The documentation on R's official website contains an excellent "Introduction to R",
and then a printout of individual functions' help in a 3500-page (!) "Reference Index".
There is nothing in between, nothing like the official documentation of, say, Python.
The "base" package in the "Reference Index" has about 400--500 entries,
listed alphabetically (!).
Most entries in the TOC are function names, whereas many are not, but rather are "topics"
(i.e. small group of functions around a common topic).
Many entries document multiple functions.
The content suggests that many functions are not listed in TOC.
How useful is this document?

In the `base` package we see functions like

- `.bincode`
- `as.Date`
- `AsIs`
- `assign`
- `c`
- `Cstack_info`
- `curlGetHeaders`
- `dontCheck`
- `.Defunct`
- `default.stringAsFactors`
- `.deparseOpts`
- `do.call`
- `gl`
- `iconv`
- `isSymmetric`
- `isS4`
- `asS4`
- `is.integer`
- `as.integer`
- `kappa`
- `l10n_info`
- `.Last.value`  (this is a variable, not a function)
- `La_version`
- `.NotYetImplemented`
- `path.expand`
- `normalizePath`
- `rle`
- `on.exit`
- `.onLoad`
- `.Last.lib`
- `pos.to.env`
- `readRenviron`
- `sQuote`
- `stopifnot`
- `Sys.glob`
- `system.file`
- `trimws`
- `t`
- `Vectorize`
- `xtfrm`
- `and_So.on` (I made up this one)

There is no naming convention. (Admittedly, naming is the hardest thing in computer science.) Apparently, R has grown organically.
The early developers threw convenience utilities and whatever they needed into the big pile.
The core (`base`) does not contain a carefully debated set of functionalities
on a similar level of abstraction.
The functionalities range from very fundamental and commonly-used to very advanced or corner-case.
For example, R `base` contains what appears to be a few dozen string-manipulating functions, including full-power regex functions.
Why should a regex function appear in the language's core? (unless it's Perl.)

Ten years ago, I [posted to the R mailing list](https://stat.ethz.ch/pipermail/r-devel/2006-September/042785.html) calling for some modularization of the R `base`.
I got no response. I deserved it. I did not think how impossible it is to disentangle such a giant pile of code.

So this is what I mean by "R is hard to learn": while you can get started writing scripts like with any other language,
it is *very* hard to go to the next level, let alone *master* it---because you don't know where to poke.

If we want to learn R by books,
we find that most such books have 1--2 chapters introducing R,
then move on to using R on their scientific subject matter.
There are very few books that teach R the language in depth, **systematically**. (Hadley Wickham is trying to do some.)

The poor organization of R the language core is reflected in the R software ecosystem as well as the source code of individual packages.
I may say that the mindset of most regular users of R is writing scripts rather than developing software.
For the typical purpose of these users,
it is not very important to care about
code organization,
documentation,
scalability,
maintainability,
robustness,
system integration,
and perhaps even speed (which is not a big concern if scalability is not).


### R is not a general-purpose language

R is a DSL (domain-specific language)---it is created and used by statisticians to do statistics. It is not intended for or suited to general purpose software development.

In picking a language for substantial software development, I agree with [John D. Cook](http://www.johndcook.com/blog/2015/07/16/scientific-computing-in-python/),

> I’d rather do mathematics in a general programming language than do general programming in a mathematical language.
>

Even with my decent knowledge in R, if I continue using it,
I would necessarily continue investing more time learning
the language, the ecosystem, and particular packages.
With the same amount of investment made on a **general-purpose, main-stream** language,
like Python, I would advance my skill much better as an IT professional,
even if certain tasks would have been easier to handle in R.

### R tends to be slow

Both R and Python are slow.
Both have very fast packages (that do the actual work in `C` or `C++` or `Fortran`).
But R tends to be slower.
One reason is its pass-by-value semantics.
To be safe, a lot of copying is made that is not always necessary.
Until a very recent version (3.1?),
if you need to manipulate a small element in an R `list` (similar to a Python `dict`), the entire `list` will be copied, however big it is. To be sure, this is not a symptom of a functional language. For example, Clojure does not have this problem as far as I know.

One summer a few years ago,
Neal Radford, a CS professor and statistician,
dug into some dark corners of the R implementation (which is in `C`)
and produced a series of patches that speed up some common operations.
His patches were not welcomed (at least not in a timely fashion) by the Core Team
for integration into the official code base.
He ended up maintaining a fork, which he calls ["Pretty Quick R" (pqr)](https://github.com/radfordneal/pqR). He has been the sole contributor to this project on Github. (Granted, not many people are capable of contributing to such language implementation projects.)

There are other efforts on improving R's speed. I mention this project only to show some not so forward-looking attitudes of the R maintainers.


## What if you have to use some R?

What if there is a specialized R package or function that is absolutely, exactly the right thing to use?

Check out [rpy2](https://rpy2.bitbucket.io). Use it to call R from Python in a way such that your visible use of R is very localized, very isolated, very encapsulated, very controlled, very minimized. Do not let R spill into peripheral data processing, I/O,  source code structuring, and other functionalities that do not have to use R.
This is not saying that R is very bad at the peripheral tasks.
This is saying that **you want to minimize language diversity in your project**.

Even in this case, I would definitely have careful team review and supervision.
Introducing one more **major** language into a system is always a serious matter.
I have not encountered a case where this is necessary.

(This note was written in an attempt to dissuade my team from introducing R into the system.)

