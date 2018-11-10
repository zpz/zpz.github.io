---
layout: post
title: Understanding Gradient Boosting Tree for Binary Classification
use_math: true
---

I did some reading and thinking about Gradient Boosting Machine (GBM),
especially used for binary classification, and cleared up some confusion in my mind. In this post I'll try to explain some of this newly-gained understanding in a way that is as intuitive, clear, and understandable as I can make it.

I'll assume the *weak learner* is a tree, which is by far the most common practice.
In this situation, the algorithm is also called *Boosting Tree*.

## The analogy between Gradient Boosting and Gradient Descent

### Gradient descent

Let's fit a simple linear regression by gradient descent.
The data points are $(x_1, y_1), (x_2, y_2),..., (x_n, y_n)$.
The model is $Y = a + bX$. The unknown parameters to be solved for are $a$ and $b$.

Suppose we have iterated $m$ steps, and the values of $a$ and $b$ are now
$a_m$ and $b_m$. The task is to update them to $a_{m+1}$ and $b_{m+1}$, respectively.

To do that, we need to define a **loss function**, and then revise $a_m$ and $b_m$ so that the loss function will decrease. The loss function is the standard "mean squared error":

$$
    L(a_m, b_m) = \frac{1}{2n}\sum_i^n (y_i - \hat{y}_i)^2
                = \frac{1}{2n}\sum_i^n (y_i - a_m - b_m x_i)^2
$$

Take partial derivatives,

$$
\begin{align*}
\frac{\partial L}{\partial a_m} &= -\frac{1}{n}\sum_i^n (y_i - a_m - b_m x_i) \\\\
\frac{\partial L}{\partial b_m} &= -\frac{1}{n}\sum_i^n (y_i - a_m - b_m x_i) x_i
\end{align*}
$$

Regardless the current value of the loss $L$, since we want to reduce it,
we go in the opposite direction of its derivatives, or gradient:

$$
\begin{align*}
a_{m+1} &= a_m - \eta \frac{\partial L}{\partial a_m} \\\\
b_{m+1} &= b_m - \eta \frac{\partial L}{\partial b_m}
\end{align*}
$$

where $\eta$ is a tuning parameter called *learning rate*.

### Gradient boosting

Now consider gradient boosting. I will use the same notation for the data points, but understand that $x$ may well be a vector. I'm also going to keep the nature of the problem pretty unspecified and generic. My goal is to find a model, say $\hat{y} = F(x)$, that maps $x$ to $y$ so that a certain loss function $L$ is minimized.

This is important and deserves repeat: to keep things general, **my goal is to minimize the loss function, which is not necessarily matching $\hat{y}$ to $y$.**

This is an iterative procedure.
Suppose at iteration $m-1$, my model is $F_{m-1}$, i.e.,

$$
\hat{y}_i = F_{m-1}(x_i),\qquad i=1,...,n
$$

The task is to *revise* this model so that the loss function is decreased.
However, I'm not going to revise $F_m$ directly in any way; instead, I'm going to **add** another model, a "delta model", to it, so that my model will become

$$
\hat{y}_i = F_{m}(x_i) \equiv F_{m-1}(x_i) + f_{m}(x_i),\qquad i=1,...,n
$$

Note how similar in principle this is to the case of revising the parameter estimate $a_m$ by adding a small delta to it. While it is pretty open-ended how we could revise $F_m$, keeping it intact and adding another "small" model to it can be more controllable.

Now the question is: what is $f_{m}(x_i)$, and how do I find it? It's quite OK to specify its general type up-front, say I'm going to use a shallow tree. Then the remaining question is how to train this tree, in other words, what's the data and loss function for it?

The change to the model causes a change to each predicted $\hat{y}\_i$
by the amount of $f_{m}(x_i)$. This changes the value of the loss function.
This is how $f_m$ is connected to the loss function.
This connection give us a handle to determine $f_m$.

Now I flip the question. Instead of having whatever $f_{m}$ and asking how it changes $\hat{y}_i$ and the loss, let me ask: in order to decrease the loss, how do I want each $\hat{y}_i$ to change? The answer is, look at the gradient!
Specifically, I want each $\hat{y}_i$ to change in the opposite direction of the derivative of $L$ with regard to $\hat{y}_i$:

$$
\hat{y}_i - \eta \frac{\partial L}{\partial \hat{y}_i}
$$

where $\eta$ is a tuning parameter. This suggests that I want to have

$$
f_{m}(x_i) = -\eta \frac{\partial L}{\partial\hat{y}_i},\qquad i=1,...,n
$$

In general this can not be achieved exactly for all $i$, but this says that the model
$f_{m}$ should be trained with the same predictors $x_i$, whereas the responses should be $-\eta\frac{\partial L}{\partial\hat{y}_i}$.
Intuitively, we want the model to generate these predictions, hence I train the model against these values.

To clarify the notation a little bit, we start with a simple model $F_0$;
with iterations the model becomes $F_1$, $F_2$,..., $F_m$.
The relation between two consecutive iterations is

$$
F_{m}(x_i) \equiv F_{m-1}(x_i) + f_{m}(x_i), \qquad m = 1,2,...
$$

where $f_{m}$ is trained by the dataset
$(x_i, -\eta\frac{\partial L}{\partial \hat{y}_i})$, $i=1,...,n$.

When training $f_{m}$, the loss function is totally unrelated to the "overall" loss function $L$; mean squared error might be a reasonable default choice.

Expanding the model increments, we see

$$
F_m(x_i) \equiv F_0(x_i) + f_1(x_i) + \dots + f_{m}(x_i),\qquad i=1,2,...
$$

### The analogy

When using gradient descent to estimate some variable,
in each iteration, we make a change to the variable's value.
The change is determined based on the gradient of the loss with respect to the variable.

When using gradient boosting to estimate some model,
in each iteration, we make a change to the model.
The change is realized by adding an "incremental", usually simple, model to the previous model.
This incremental model is determined by what $y$ values are used to train it, along with the original $x$.
In particular, the $y$ values are determined based on the gradient of the loss function
with respect to the corresponding prediction by the previous model.


### "Fit the residual"

If the loss is mean squared error, then

$$
-\eta\frac{\partial L}{\partial \hat{y}_i}
= -\frac{\eta}{2}\frac{\sum_j^n(y_j - \hat{y}_j)^2}{\partial \hat{y}_i}
= \eta(y_i - \hat{y}_i)
$$

I have omitted the factor $1/n$ to make the formula cleaner.

If $\eta$ is 1, then $f_m$ is fitted to the *residuals*
of the previous model, $F_{m-1}$.

When the loss $L$ is not mean squared error, the fitting does not need to be against the residuals.


## The loss function of a binary classifier

Consider a single data point with binary response $y \in (0,1)$.
Suppose we have a probability prediction $\hat{y} = \mathrm{Prob}(y = 1)$.
We need to define the loss for this single data point.

First, let's consider log-likelihood. If $y = 1$, the log-likelihood is $\log\hat{y}$.
If $y = 0$, the log-likelihood is $\log(1 - \hat{y})$.
If we want to express both cases in one unified formula, the following works:

$$
y \log\hat{y} + (1-y)\log(1 - \hat{y})
$$

The loss should be negative (log-)likelihood, so we can use

$$
L = -y \log\hat{y} - (1-y)\log(1 - \hat{y})
$$

Next, consider two probability distributions $p$ and $q$ over the same underlying set of events. Suppose $p$ is the true distribution, and $q$ is used to approximate $p$.
Roughly speaking, cross-entropy $H(p,q)$ measures the "distance" in this approximation.
When $p$ and $q$ are discrete, the cross-entropy is defined by

$$
H(p,q) = -\sum_x p(x) \log q(x)
$$

This is the negative expected log-likelihood when events happen according to $p$,
but we calculate the likelihood using distribtion $q$.
$H(p,q)$ decreases as the quality of the approximation improves,
and is minimized when $p$ and $q$ are identical.
Therefore, $H$ can be used as a measure of loss.

For a single data point in a binary classification problem,
when $y = 1$, we have $H = -\log \hat{y}$;
when $y = 0$, we have $H = -\log(1 - \hat{y})$.
Combining the two cases, we have

$$
H = - y \log \hat{y} - (1-y) \log(1 - \hat{y})
$$

The **cross-entropy loss** is also known as the **log loss** or **deviance**.
Obviously, this coincides with the metric $L$ derived from a log-likelihood perspective.


## The loss function in a binary classification boosting tree

In a binary classifier, one approach is to build a linear model

$$
\hat{y} = \sum_j \beta_j x_j,\qquad j=0,...,k
$$

and take the logit of it as a prediction of probability:

$$
\hat{p} = \frac{1}{1 + e^{-\hat{y}}}
$$

Here, $j$ is the feature index, not the observation index.
Also note that $x_0$ is always 1, corresponding to the bias parameter.
In fact, the $\hat{y}$ is not a prediction of the binary response $y$;
these two are not comparable at all. So this notation is a little sloppy, but it will be convenient below.

Now let's consider using GBM for binary classification.
Although we want to provide probability predictions,
we can not let the models $F_0, f_1, f_2,...$ output $\hat{p}$,
because the boosting tree is an additive model, yet $\hat{p}$ is not additive.
How can we overcome this?

The solution is that we make $F_0, f_1, f_2,...$ **regression trees**, which are additive.
We perform the logit transformation only when we are about to provide the probability prediction, but that transformation is not really part of the boosting tree model.

Let's take any particular incremental model, $f_m$, and write

$$
\hat{y}_i = f_m(x_i),\qquad i=1,2,...,n
$$

$f_m$ is not a linear regression; it's a regression tree.
The prediction $\hat{y}_i$ is not comparable to the binary observation $y_i$.
Luckily, as shown above, this is not a problem.
We only need to clarify the loss function and work out the gradient with respect to $\hat{y}_i$.

A good loss function for classification problems is cross-entropy. We still use it here.
The issue is the probability $\hat{p}_i$ is "virtual". We need to do away with it and manipulate things with respect to $\hat{y}_i$.

$$
L_i = -y_i \log \hat{p}_i - (1 - y_i)\log(1 - \hat{p}_i)
    = -y_i\log \frac{\hat{p}_i}{1 - \hat{p}_i} - \log(1 - \hat{p}_i)
$$

substituting $\hat{p}_i = \frac{1}{1 + e^{-\hat{y}_i}}$,

$$
\begin{align*}
\log(1 - \hat{p}_i) &= \log \frac{1}{1 + e^{\hat{y}_i}} = -\log(1 + e^{\hat{y}_i}) \\\\
\log\frac{\hat{p}_i}{1 - \hat{p}_i} &= \log e^{\hat{y}_i} = \hat{y}_i 
\end{align*}
$$

hence

$$
L_i = \log(1 + e^{\hat{y}_i}) - y_i \hat{y}_i
$$

Notice that $\log\frac{\hat{p}_i}{1 - \hat{p}_i}$ is the log odds,
hence the loss function is now expressed by the binary response and the log odds.

This loss function is what is used in
`sklearn.ensemble.gradient_boosting.BinomialDeviance.__call__`.

References:

[How to explain gradient boosting](https://explained.ai/gradient-boosting/index.html), by Terence Parr and Jeremy Howard.

StackExchange answers by Matthew Drury,
[here](https://stats.stackexchange.com/questions/157870/scikit-binomial-deviance-loss-function)
and
[here](https://stats.stackexchange.com/questions/204154/classification-with-gradient-boosting-how-to-keep-the-prediction-in-0-1)
