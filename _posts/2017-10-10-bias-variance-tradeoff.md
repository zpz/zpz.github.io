---
layout: post
title: On the Bias-Variance Tradeoff
tags: [machine-learning]
---

Yes, we have all seen the nice decomposition formula

```
  mean_squared_error = squared_bias + variance + irreducible_error
```

However, I've been puzzled by explanations of the **tradeoff** based on this formula,
because it's never convincingly clear that the ``mean_squared_error`` and ``irreducible_error``
terms are **fixed** as we explore different models!

Today I think I got the idea, which is, do not try to quantitatively explain
the tradeoff with this formula. Instead, keep it a little vague and just move on
to talking about understandable examples.

I can think of a couple of contexts for such examples.

1. Re-sampling approaches. Like *random forest*, reduce variance while not changing (or increasing) bias.
2. Model complexity or number of parameters. Like *linear regression*,
   increasing model complexity will reduce variance while increase bias.

There is another, bigger, context that is sometimes unstated, which is,
the data volume or some form of "material investment on this modeling effort" is fixed.
For otherwise, variance and bias can certainly both be improved.

With this bigger context explicit, I can imagine another type of examples, which go like this:
if we take steps to improve on one (say variance), we'll probably need to draw resources away from the other (i.e. bias).

For example, we conduct a telephone survey on voting intents, and the total number of phone calls we can make
is fixed. If we want to reduce sampling bias, we can expand the regions to cover,
resulting in decreased sample size in each region. This will reduce bias at the cost of increased variance.



