---
layout: post
title: Should we use Scala or Python?
excerpt_separator: <!--excerpt-->
tags: [Python, Scala, data-science]
---

Data team,

With the prospect of starting to use `Spark` seriously, people are saying “(it’s time to) learn `Scala`”.
I’m very worried about the data team become split between languages like our platform team is. 
<!--excerpt-->
I’d like to share some thoughts here on this matter, trying to be brief.

- Any one small project/team can not afford to have two primary languages.
  That situation is never needed, nor manageable. 

  The difficulty is not to make most team members *understand* two languages.
  That’s not the difficult part.
  Besides code review, a team needs to refactor the high-level code organization, 
  introduce some common infrastructure, promote some good practice, 
  sit together to learn the tool, and so on. 
  To do these things with two primary languages,
  either everyone's work load is doubled, or the team is split in two.

- Our task is to fight the problems, not the language (or the tool in general).

- In terms of language or tool, our priority is not adopting “more powerful” (fancier) languages. 
  Our priority is to improve (or start) code sharing/reuse/modularization/testing/review, 
  knowledge exchange, mutual training, and all the SE good practices.

- If we are not conscious about following good practices with one language, we will **not** with another.

- `Python` is one of the few (or two) mainstream, catch-all languages that can get anything done,
  including getting you a job.
  For “data work” oriented at analytics and modeling, `Python` is likely the #1 language choice.
  The `PyData` community is very active; the tools landscape is evolving fast.
  I’ve long had a couple languages on my “to-learn” list but I’m putting that on hold again and again because there’s still **a lot**
  I can learn about, or through, `Python`.
  I’ve not gone through most of the **excellent** documentations of `numpy`, `pandas`, `scikit-learn`,
  (and the standard library) among many other equally excellent but smaller ones;
  I’ve not solidly practiced many general SE topics, using `Python` as the media,
  such as concurrency/parallelism, CI, IDE use, and many more.

  There’re many big names in `Python` (just like in any other language).
  Do they not know `Python`’s shortcomings? Yes they do.
  Are they ignorant of other nice languages? No they are not.
  Mike Bayer, the author of `SQLAlchemy`, once wrote “`Python` is Very, Very slow” in a blog post section title
  (later dropped one “very”). Then why does he still use it?

  Because `Python` is a **good tool**, while there is no perfect tool.
  The point is to master a, not many, good tool, and use it to **produce good work**.

  If one spends time to master *many* tools, she becomes a *tool expert*, not a *subject expert*.
  Do you want to be the former, or the latter?
  And, frankly, have you mastered *one* tool yet?

- `Scala` proponents like to say that, for `Spark` programming,
  “`Python` support often lags behind `Scala`, and faces various restrictions compared to `Scala`”.
  I agree totally, and would pick `Scala` any day, **if there were no cost**.

  And I also can say with confidence, that for many (non-`Spark`) things, “`Scala` lags behind `Python` badly,
  and faces various restrictions compared to `Python`”.

- `Spark` will **not** be the center of our work, just like `Hive` is not.
  It’s a tool, maybe even a big one, but is **much** smaller than the primary programming language,
  in virtually every aspect of volume of work, capacity, scope, past and ongoing investment, and long-term value.

- Personally, `Scala` is not really on my “to-learn” list, because it has pretty firmly gained the reputation
  of being complicated and hard to maintain. Some say it’s the “`C++` on JVM” (or worse).
  I saw one comment that it’s become even more complicated than `C++`.
  This year, Odersky (creator of `Scala`)  announced that some features will be
  [removed from the language](https://www.lightbend.com/company/news/after-a-quiet-2015-martin-odersky-outlined-significant-plans-for-scala-at-scala-days-new-york).

  Everyone is smart enough to learn `Scala`, or `C++`, or even `Perl`, and feel fast and efficient.
  That’s not the problem. The problem is the team, is collaboration, is focus of work.

To wrap up, this is not about which one of `Scala` and `Python` is better,
or whether `Scala` is worth learning. It’s about **ROI**.
If you’re not already “pretty good” at `Python`, my opinion is to resist the distractions.
Learn the language. Learn a new library. Learn a new type of programming topic.
Use the tool you have invested in.

