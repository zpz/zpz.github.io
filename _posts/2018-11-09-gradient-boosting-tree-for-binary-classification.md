---
layout: post
title: Understanding Gradient Boosting Tree for Binary Classification
use_math: true
---

I did some reading and thinking about Gradient Boosting Machine (GBM),
especially used for binary classification, and cleared up some confusion in my mind. In this post I'll try to explain some of this newly-gained understanding in a way that is as intuitive, clear, and understandable as I can make it.

I'll assume the *weak learner* is a tree, which is by far the most common practice.
In this situation, the algorithm is also called *Boosting Tree*.

# The high-level idea of Gradient Boosting

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
\frac{\partial L}{\partial a_m} &= -\frac{1}{n}\sum_i^n (y_i - a_m - b_m x_i) \\
\frac{\partial L}{\partial b_m} &= -\frac{1}{n}\sum_i^n (y_i - a_m - b_m x_i) x_i
\end{align*}
$$

Regardless the current value of the loss $L$, since we want to reduce it,
we go in the opposite direction of its derivatives, or gradient:

$$
\begin{align*}
a_{m+1} &= a_m - \eta \frac{\partial L}{\partial a_m} \\
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

### The analogy between Gradient Boosting and Gradient Descent

When using gradient descent to estimate some variable,
in each iteration, we make a change to the variable's value.
The change is determined based on the gradient of the loss with respect to the variable.

When using gradient boosting to estimate some model,
in each iteration, we make a change to the model.
The change is realized by adding an "incremental", usually simple, model to the previous model.
This incremental model is determined by what $y$ values are used to train it, along with the original $x$.
Specifically, the $y$ values are determined based on the gradient of the loss function
with respect to the corresponding prediction by the previous model.

Let me try it again.
Gradient boosting uses gradient descent to iterate over the prediction for each data point, towards a minimal loss function. In each iteration, the desired change to a prediction is determined by the gradient of the loss function with respect to that prediction as of the previous iteration.
The desired changes to the predictions are realized by adding an incremental model to the model of the previous iteration---this incremental model is trained against the desired changes as $y$, along with the original $x$, so that its predictions will produce, more or less, the desired changes.


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


# The loss function of a binary classifier

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


# The loss function in a binary classification boosting tree

In a binary classifier, one approach is to build an additive (e.g. linear) model
$
\hat{y} = f(x)
$,
and take the logit of it as a prediction of probability:

$$\begin{equation}\label{logit}
\hat{p} = \frac{1}{1 + e^{-\hat{y}}}
\end{equation}
$$

In a sense, the $\hat{y}$ here is not trying to predict the binary response $y$;
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
\begin{align}
\log(1 - \hat{p}_i) &= \log \frac{1}{1 + e^{\hat{y}_i}} = -\log(1 + e^{\hat{y}_i}) \\
\log\frac{\hat{p}_i}{1 - \hat{p}_i} &= \log e^{\hat{y}_i} = \hat{y}_i \label{log-odds}
\end{align}
$$

hence

$$
L_i = \log(1 + e^{\hat{y}_i}) - y_i \hat{y}_i
$$

Notice that $\log\frac{\hat{p}_i}{1 - \hat{p}_i}$ is the log odds (aka logit of $p$),
hence the loss function is now expressed by the binary response and the log odds.

Take derivative,

$$
\begin{equation}\label{boost-gradient}
\frac{\partial L_i}{\partial \hat{y}_i}
= \frac{e^{\hat{y}_i}}{1 + e^{\hat{y}_i}} - y_i
= \hat{p}_i - y_i
\end{equation}
$$


# Implementation details of gradient boosting classifier in scikit-learn

Let's really understand how this algorithm works as implmented in `scikit-learn`.
There are many details; we'll focus on a high level around the loss function.
The main module is [`sklearn.ensemble.gradient_boosting`](https://github.com/scikit-learn/scikit-learn/blob/master/sklearn/ensemble/gradient_boosting.py).
The module's doc summarizes beautifully:

```python
"""Gradient Boosted Regression Trees

This module contains methods for fitting gradient boosted regression trees for
both classification and regression.

The module structure is the following:

- The ``BaseGradientBoosting`` base class implements a common ``fit`` method
  for all the estimators in the module. Regression and classification
  only differ in the concrete ``LossFunction`` used.

- ``GradientBoostingClassifier`` implements gradient boosting for
  classification problems.

- ``GradientBoostingRegressor`` implements gradient boosting for
  regression problems.
"""
```

I'm going to anchor at the class `GradientBoostingClassifier` and understand its initializer (`__init__`) as well as the methods `fit`, `predict_proba`, `predict`.
Although the code can handle multi-class classification, I will only track down its behavior for binary classification.

Unless otherwise noted, all classes investigated below reside in the module
`sklearn.ensemble.gradient_boosting`.

### `__init__`

The doc states very clearly,

```python
class GradientBoostingClassifier(BaseGradientBoosting, ClassifierMixin):
    """Gradient Boosting for classification.

    GB builds an additive model in a
    forward stage-wise fashion; it allows for the optimization of
    arbitrary differentiable loss functions. In each stage ``n_classes_``
    regression trees are fit on the negative gradient of the
    binomial or multinomial deviance loss function. Binary classification
    is a special case where only a single regression tree is induced.
    """
```

The following parameters are particularly interesting here:

```python
    """
    Parameters
    ----------
    loss : {'deviance', 'exponential'}, optional (default='deviance')
        loss function to be optimized. 'deviance' refers to
        deviance (= logistic regression) for classification
        with probabilistic outputs. For loss 'exponential' gradient
        boosting recovers the AdaBoost algorithm.

    learning_rate : float, optional (default=0.1)
        learning rate shrinks the contribution of each tree by `learning_rate`.
        There is a trade-off between learning_rate and n_estimators.

    n_estimators : int (default=100)
        The number of boosting stages to perform. Gradient boosting
        is fairly robust to over-fitting so a large number usually
        results in better performance.

    subsample : float, optional (default=1.0)
        The fraction of samples to be used for fitting the individual base
        learners. If smaller than 1.0 this results in Stochastic Gradient
        Boosting. `subsample` interacts with the parameter `n_estimators`.
        Choosing `subsample < 1.0` leads to a reduction of variance
        and an increase in bias.

    criterion : string, optional (default="friedman_mse")
        The function to measure the quality of a split. Supported criteria
        are "friedman_mse" for the mean squared error with improvement
        score by Friedman, "mse" for mean squared error, and "mae" for
        the mean absolute error. The default value of "friedman_mse" is
        generally the best as it can provide a better approximation in
        some cases.

    max_depth : integer, optional (default=3)
        maximum depth of the individual regression estimators. The maximum
        depth limits the number of nodes in the tree. Tune this parameter
        for best performance; the best value depends on the interaction
        of the input variables.

    max_features : int, float, string or None, optional (default=None)
        The number of features to consider when looking for the best split:

        - If int, then consider `max_features` features at each split.
        - If float, then `max_features` is a fraction and
          `int(max_features * n_features)` features are considered at each
          split.
        - If "auto", then `max_features=sqrt(n_features)`.
        - If "sqrt", then `max_features=sqrt(n_features)`.
        - If "log2", then `max_features=log2(n_features)`.
        - If None, then `max_features=n_features`.

        Choosing `max_features < n_features` leads to a reduction of variance
        and an increase in bias.
    """
```

The default loss function is the "deviance". For binary classification,
this is `BinomialDeviance` (set in `BaseGradientBoosting._check_params`).
The exact details of the loss function is in `BinomialDeviance.__call__`.

The parameter `subsample` allows Bagging-like behavior but is disabled by default.

The parameter `max_features` allows Random-Forest-like behavior but is disabled by default.

### `fit`

`GradientBoostingClassifier.fit` directly uses the parent
`BaseGradientBoosting.fit`.

The most important steps (skipping all sanity checks and most non-default or non-binary-class scenarios) are as follows:

1. Set `self.classes_ = [0, 1]` and `self.n_classes_ = 2`.

2. Set `self.loss_ = BinomialDeviance(2)`.
Unpon this setting, `self.loss_.K` is 1, the number of regression trees to be induced
(see `LossFunction`).

3. Set `self.init_ = self.loss_.init_estimator()`, which effectively does
`self.init_ = LogOddsEstimator()`.

4. Fit initial model: `self.init_.fit(X, y)`.
The effect of this operation is
`self.init_.prior = np.log(pos / neg)`,
where `pos` and `neg` are counts of 1's and 0's in `y`, respectively.

5. Make initial predictions: `y_pred = self.init_.predict(X)`.
The effect of this assignment is that
`y_pred` is a column vector of the length of `y`, filled with the value of `self.init_.prior`.

6. Start iterations (see `BaseGradientBoosting._fit_stages`).
The workhorse in `_fit_stages` is `_fit_stage`, which is called once in each iteration,
taking `X`, `y`, `y_pred`, and returning new `y_pred`, to be used in the next iteration.
`X` and `y` are the original data, and they do not change in the iterations.


### `_fit_stage`

For binary classification, `_fit_stage` fits a `sklearn.tree.DecisionTreeRegressor`;
most work is delegated to the latter class.
The fitted tree is appended to the list of "estimators".

The main steps are as follows.

1. Compute `residual = self.loss_.negative_gradient(y, y_pred)`.
This `residual` turns out to be `y` minus the logit of `y_pred`, hence essentially
`residual = y - 1./(1. + np.exp(-y_pred))`.

2. Fit a `DecisionTreeRegressor` with `X` as $x$ and `residual` as $y$.
An interesting parameter passed to `DecisionTreeRegressor` is
`criterion=self.criterion`, which is `'friedman_mse'` by default.
This criterion uses mean squared error with Friedman's improvement score
to assess potential splits (see `sklearn.tree._criterion.FriedmanMSE`).

3. Update tree leaves: `self.loss_.update_terminal_regions`. This does two things:

   1. Determine each leaf's prediction value as this:
      `np.sum(residual) / np.sum((y - residual) * (1 - y + residual))`.
      The summation is over data samples that fall in the particular leaf.
      Because `residual` is $y - \hat{p}$, this leaf value is
      $\frac{\sum_i (y_i - \hat{p}_i)}{\sum_i \hat{p}_i (1 - \hat{p}_i)}$,
      where the data point $x_i$ falls in the leaf in question.

   2. Increment `y_pred` of observation $i$ by `self.learning_rate` times the leaf value
      in which this observation falls. The updated vector `y_pred` is the return value
      of `_fit_stage`.

Let's understand the two keys here, namely `y_pred` and `residual`.

The initial value of `y_pred` is `np.log(pos / neg)`, which is the global log odds.
In $(\ref{logit})$, if we take $\hat{p}$ to be 
$\frac{\mathrm{pos}}{\mathrm{pos} + \mathrm{neg}}$,
then by $(\ref{log-odds})$,
$\hat{y} = \log\frac{\hat{p}}{1 - \hat{p}} = \log\frac{\mathrm{pos}}{\mathrm{neg}}$.
So indeed, the initial `y_pred` is the additive part corresponding to the global fraction of positive responses as the probability. This is a reasonable starting point.

The tree regressor is fitted against `residual`.
As long as `y_pred` continues to play the role of the additive part in a logistic regression (which is the case at the beginning of the iteration),
`residual` is $y - \hat{p}$. This is exactly the negative gradient according to
$(\ref{boost-gradient})$.




### `predict_proba`

### `predict`


References:

[How to explain gradient boosting](https://explained.ai/gradient-boosting/index.html), by Terence Parr and Jeremy Howard.

StackExchange answers by Matthew Drury,
[here](https://stats.stackexchange.com/questions/157870/scikit-binomial-deviance-loss-function)
and
[here](https://stats.stackexchange.com/questions/204154/classification-with-gradient-boosting-how-to-keep-the-prediction-in-0-1)
