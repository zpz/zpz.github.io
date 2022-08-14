---
layout: post
title: Understanding Gradient Boosting Tree for Binary Classification
use_math: true
excerpt_separator: <!--excerpt-->
tags: [machine-learning, boosting, classification, scikit-learn]
---

I did some reading and thinking about Gradient Boosting Machine (GBM),
especially for binary classification, and cleared up some confusion in my mind.
<!--excerpt-->
In this post I'll try to explain some of this newly-gained understanding in a way that is as intuitive, clear, and understandable as I can make it.

I'll assume the *base learner* is a tree, which is by far the most common practice.
In this situation, the algorithm is also called *Boosting Tree*.

## The high-level idea of Gradient Boosting

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
\frac{\partial L}{\partial b_m} &= -\frac{1}{n}\sum_i^n (y_i - a_m - b_m x_i)\, x_i
\end{align*}
$$

Since we want to reduce $L$,
we move $a_m$ and $b_m$ in the opposite direction of $L$'s derivatives wrt them,

$$
\begin{align*}
a_{m+1} &= a_m - \eta \frac{\partial L}{\partial a_m} \\
b_{m+1} &= b_m - \eta \frac{\partial L}{\partial b_m}
\end{align*}
$$

where $\eta$ is a tuning parameter called *learning rate*.

### Gradient boosting

Now consider gradient boosting. I will use the same notation for the data points, but understand that $x$ may well be a vector. I'm also going to keep the nature of the problem pretty unspecified and generic. My goal is to find a model, say $\hat{y} = F(x)$, that maps $x$ to $y$ so that a certain loss function $L$ is minimized.

This is important and deserves repeat: to keep things general,
**my goal is to minimize a loss function, which is not necessarily to match $\hat{y}$ to $y$.**

This is an iterative procedure.
Suppose at iteration $m$, my model is $F_{m}$, i.e.,

$$
\hat{y}_i = F_{m}(x_i),\qquad i=1,...,n
$$

The task is to *revise* this model so that the loss function is decreased.
However, I'm not going to revise $F_m$ directly in any way;
instead, I'm going to **add** another model, a "delta model" or "incremental model", to it,
so that the model will become

$$
\hat{y}_i = F_{m+1}(x_i) \equiv F_{m}(x_i) + f_{m+1}(x_i),\qquad i=1,...,n
$$

Note how similar in principle this is to the case of revising the parameter estimate $a_m$ by adding a small delta to it.
If we were to revise $F_m$ directly, it is hard to see how we should start.
In comparison, keeping $F_m$ intact and adding another "small" model to it can be much more controllable.
Because this is an iterative procedure, it feels quite OK to require that $f_{m+1}$
is of certain specific type with a relatively simple structure.
Then the question is how to train this model, in other words, what's the data and the loss function for it?

The addition of $f_{m+1}$ causes a change to each predicted $\hat{y}\_i$
by the amount of $f_{m+1}(x_i)$. This in turn changes the value of the loss function.
This is how $f_{m+1}$ is connected to the loss function,
and this connection is the key to finding $f_{m+1}$.

Now I flip the question.
Instead of asking how $f_{m+1}$ (however it is determined) changes $\hat{y}_i$ and the loss,
I ask: in order to decrease the loss, how do I want each $\hat{y}_i$ to change?

The answer is, of course, look at the gradient!
Specifically, I want each $\hat{y}_i$ to change in the opposite direction of the derivative of $L$ with regard to it:

$$
\hat{y}_i \longleftarrow \hat{y}_i - \eta \frac{\partial L}{\partial \hat{y}_i}
$$

where $\eta$ is a tuning parameter. This suggests that I want to have

$$
f_{m+1}(x_i) = -\eta \frac{\partial L}{\partial\hat{y}_i},\qquad i=1,...,n
$$

In general this can not be achieved exactly for all $i$, but this says that the model
$f_{m+1}$ should be trained against
$-\eta\frac{\partial L}{\partial\hat{y}_i}$ as $y$,
with the corresponding predictors $x_i$ as $x$.
Intuitively, I want the model to generate these predictions, hence I train the model against these values.

To clarify the notation a little bit, we start with a simple model $F_0$;
with iterations the model becomes $F_1$, $F_2$,..., $F_m$.
The relation between two consecutive iterations is

$$
F_{m+1}(x_i) \equiv F_{m}(x_i) + f_{m+1}(x_i), \qquad m = 0,1,...
$$

where $f_{m+1}$ is trained with the dataset
$
(x\_i, -\eta\frac{\partial L}{\partial \hat{y}\_i})
$,
$i=1,...,n$.
When training $f_{m+1}$, the loss function is totally unrelated to the "overall" loss function $L$;
compared to the boosting iterations, the training of $f_{m+1}$ is a "local" problem.

Expanding the model increments, we see

$$
F_m(x_i) \equiv F_0(x_i) + f_1(x_i) + \dots + f_{m}(x_i),\qquad i=1,2,...,n
$$

### The analogy between Gradient Boosting and Gradient Descent

When using gradient descent to estimate some variable,
in each iteration, we make a change to the variable's value.
The change is determined based on the gradient of the loss with respect to the variable.

When using gradient boosting to estimate some model,
in each iteration, we make a change to the model.
The change is realized by adding an "delta", usually simple, model to the previous model.
This delta model is determined by what $y$ values are used to train it, along with the original $x$.
Specifically, the $y$ values are determined based on the gradient of the loss function
with respect to the corresponding prediction by the previous model.

Let me try it again.
Gradient boosting uses gradient descent to iterate over the prediction for each data point, towards a minimal loss function. In each iteration, the desired change to a prediction is determined by the gradient of the loss function with respect to that prediction as of the previous iteration.
The desired changes to the predictions are realized by adding an "delta" model to the model of the previous iteration---this delta model is trained against the desired changes as $y$, along with the original $x$, so that its predictions will produce, more or less, the desired changes.


### "Fit the residual"

If the loss is mean squared error, then

$$
-\eta\frac{\partial L}{\partial \hat{y}_i}
= -\frac{\eta}{2}\frac{\partial \sum_j(y_j - \hat{y}_j)^2}{\partial \hat{y}_i}
= -\frac{\eta}{2}\frac{\partial (y_i - \hat{y_i})^2}{\partial \hat{y}_i}
= \eta(y_i - \hat{y}_i)
$$

(I have omitted the factor $1/n$ to make the formula cleaner.)
If $\eta$ is 1, then $f_{m+1}$ is fitted to the *residuals*,
$y\_i - \hat{y}\_i$,
of the previous model, $F_{m}$.
This is why we often see statements like "each subsequent model is trained against the residule of the previous model" in descriptions of the gradient boosting algorithm.

However, when the loss $L$ is not mean squared error, the training does not need to be against the residuals. The more general statement is that the training is against "the gradient of the loss function wrt the predictions of the previous model".


## The loss function for binary classification

Consider a single data point with binary response $y \in \\{0,1\\}$.
Suppose we have a probability prediction $\hat{p} = \operatorname{prob}(y = 1)$.
We need to define the loss for this single data point.
To fix notations,
we'll use $y$ for the *variable* and
$y^*$ for the actually observed value.


First, let's consider **log-likelihood**. If $y^* = 1$, the log-likelihood is $\log\hat{p}$.
If $y^* = 0$, the log-likelihood is $\log(1 - \hat{p})$.
If we want to express both cases in one unified formula, the following works:

$$
y^* \log\hat{p} + (1-y^*)\log(1 - \hat{p})
$$

The loss should be negative (log-)likelihood, so we can use

$$
\begin{equation}\label{neg-loglikelihood-loss}
L = -y^* \log\hat{p} - (1-y^*)\log(1 - \hat{p})
\end{equation}
$$

Next, let's consider **cross-entropy**.
Consider two probability distributions $Q$ and $P$ over the same underlying set of events. Suppose $Q$ is the true distribution, and $P$ is used to approximate $Q$.
Roughly speaking, cross-entropy $H(Q,P)$ measures the "distance" in this approximation.
When $Q$ and $P$ are discrete distributions taking event $y$ in some finite set,
the cross-entropy is defined by

$$
H(Q,P) = -\sum_y Q(y) \log P(y)
$$

This is the negative expected log-likelihood when events happen according to $Q$,
but we calculate the likelihood using distribtion $P$ (because we thought the true distribution is $P$).
Because $H(Q,P)$ is *negative* log-likelihood,
it decreases as the quality of the approximation improves,
and reaches a minimum when $Q$ and $P$ are identical.
This interpretation goes very well with the situation where we use $P$ to *estimate* an unkown distribution $Q$. And for that estimation, $H(Q,P)$ can readily play the role of a loss function.

For a single data point in a binary classification problem, the possible events are $y=0$ and $y=1$.
If we observe $y^*=1$, the distribution $Q$ for this single data point is

$$
Q(y) = \begin{cases}
0,\quad & y=0 \\
1,\quad & y=1
\end{cases}
$$

If we observe $y^*=0$, the distribution $Q$ is

$$
Q(y) = \begin{cases}
1,\quad & y=0 \\
0,\quad & y=1
\end{cases}
$$

Both cases can be nicely expressed by one formula:

$$
Q(y) = \begin{cases}
1-y^*,\quad & y=0 \\
y^*,\quad & y=1
\end{cases}
$$

On the other hand, our estimate is the distribution

$$
P(y) = \begin{cases}
1 - \hat{p},\quad & y=0 \\
\hat{p},\quad & y=1
\end{cases}
$$

We see the cross-entropy loss can be calculated as follows,

$$
\begin{equation}\label{cross-entropy-loss}
H = - y^* \log \hat{p} - (1-y^*) \log(1 - \hat{p})
\end{equation}
$$

Obviously, the cross-entropy and negative log-likelihood are the same quantity.
This loss function is also known as the **log loss** or the **deviance**.


## The loss function in a Gradient Boosting Tree for binary classification

For binary classification, a common approach is to build some model
$
\hat{y} = f(x)
$,
and take the logit of it as a prediction of probability:

$$\begin{equation}\label{logistic}
\hat{p} = \frac{1}{1 + e^{-\hat{y}}}
\end{equation}
$$

Here, $x$ represents the predictor(s), the response $y$ is binary-valued (either 0 or 1),
$\hat{y}$ takes any real value on $(-\infty, \infty)$,
and $\hat{p}$ is a real value between 0 and 1, interpreted as the probality of $y=1$.
It is reasonable to say that the $\hat{y}$ here is not trying to *predict* the binary response $y$,
because the two are comparable in neither the meaning nor the value.
The *prediction* is $\hat{p}$, of the probability of $y=1$ (and not of the value $y$ itself),
whereas $\hat{y}$ is just some intermediate modeling device.

It's easy to derive

$$\begin{equation}\label{logit}
\hat{y} = \log\frac{\hat{p}}{1 - \hat{p}}
\end{equation}
$$

This shows $\hat{y}$ is the log odds.
The function $(\ref{logistic})$, which converts $\hat{y}$ to $\hat{p}$, is called the `logistic`;
its inverse, i.e. $(\ref{logit})$, which converts $\hat{p}$ to $\hat{y}$,
is called the `logit`.

The most familiar example is logistic regression, in which $\hat{y} = f(x)$ is a linear model of $x$.
More generally, $f$ is an **additive** model.
By this we mean $f$ can be built up as the summation of multiple components,
for example a series of basis functions, or regression trees.
Because the range of $f$ is $(-\infty, \infty)$,
there is no hard constraint to the components or the summation.

Which leads to the construction of Gradient Boosting Trees for Classification.
In such algorithms, the *base learners* are **regression trees**, whose summation provides $\hat{y}$.
They can not be *classification trees*, each providing a prediction of probability $\hat{p}$,
because the $\hat{p}$'s can not be added up to get a useful quantity.

We perform the logit transform only when we are about to provide a probability prediction based on $\hat{y}$, but that transform is not really part of the boosting tree model.

Let's take the model after the $m$-th iteration, i.e. $F_m$, and write

$$
\hat{y}_i = F_m(x_i),\qquad i=1,2,...,n
$$

As stated above,
the prediction $\hat{y}_i$ is not a prediction of the binary response $y_i$,
and this is not a problem; it's just an intermediate modeling device.
We only need to clarify the loss function and work out its gradient wrt $\hat{y}_i$.

The cross-entropy loss is the standard pick:

$$
\begin{equation}\label{L_i}
L_i
= -y_i \log \hat{p}_i - (1 - y_i)\log(1 - \hat{p}_i)
\end{equation}
$$

We can still use the quantity $\hat{p}$, as a transform of $\hat{y}$,
but we need to think about the loss mainly as a function of $\hat{y}$ rather than of $\hat{p}$.
Let's try to reform the loss in a way that will help expressing it in terms of $\hat{y}$:

$$
L_i = -y_i\log \frac{\hat{p}_i}{1 - \hat{p}_i} - \log(1 - \hat{p}_i)
$$

Using $(\ref{logistic})$ and $(\ref{logit})$, we get

$$
\begin{equation}\label{L_i_yhat}
L_i = \log(1 + e^{\hat{y}_i}) - y_i \hat{y}_i
\end{equation}
$$

The loss function is now expressed by the binary response and the log odds.

Take derivative,

$$
\begin{equation}\label{boost-gradient}
\frac{\partial L}{\partial \hat{y}_i}
= \frac{\partial \sum_j L_j}{\partial \hat{y}_i}
= \frac{\partial L_i}{\partial \hat{y}_i}
= \frac{e^{\hat{y}_i}}{1 + e^{\hat{y}_i}} - y_i
= \hat{p}_i - y_i
\end{equation}
$$

Alternatively, we have

$$
\frac{\partial \hat{p}_i}{\partial \hat{y}_i}
= \frac{e^{-\hat{y}_i}}{(1 + e^{-\hat{y}_i})^2}
= \frac{1}{1 + e^{-\hat{y}_i}} \frac{e^{-\hat{y}_i}}{1 + e^{-\hat{y}_i}}
= \hat{p}_i (1 - \hat{p}_i)
$$

Now take derivative of $(\ref{L_i})$,

$$
\frac{\partial L}{\partial \hat{y}_i}
= \frac{\partial L_i}{\partial \hat{y}_i}
= \frac{\partial L_i}{\partial \hat{p}_i} \frac{\partial \hat{p}_i}{\partial \hat{y}_i}
= \Bigl(-\frac{y_i}{\hat{p}_i} + \frac{1 - y_i}{1 - \hat{p}_i}\Bigr) \hat{p}_i (1 - \hat{p}_i)
= \hat{p}_i - y_i
$$

Neat!

Since we're at it, let's go one step further and find out the second derivative:

$$
\begin{equation}\label{loss-hessian}
\frac{\partial^2 L}{\partial \hat{y}_i^2}
= \frac{\partial^2 L_i}{\partial \hat{y}_i^2}
= \frac{\partial (\hat{p}_i - y_i)}{\partial \hat{y}_i}
= \frac{\partial \hat{p}_i}{\partial \hat{y}_i}
= \hat{p}_i(1 - \hat{p}_i)
\end{equation}
$$

This will be useful in a moment.


## Gradient Boosting classification in scikit-learn

Let's understand how Gradient Boosting classification works in `scikit-learn`.
There are many details; I'll focus on the high level around the loss function and model iteration.
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

I'm going enter at the class `GradientBoostingClassifier` and understand its initializer (`__init__`) as well as the main methods for training and prediction.
Although the code can handle multi-class classification, I will only track down its behavior for binary classification.

Unless otherwise noted, all classes investigated below reside in the module
`sklearn.ensemble.gradient_boosting`.

### `__init__`

The doc gives a good summary:

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

```python
class BinomialDeviance(ClassificationLossFunction):

    def __call__(self, y, pred, sample_weight=None):
        """Compute the deviance (= 2 * negative log-likelihood).

        Parameters
        ----------
        y : array, shape (n_samples,)
            True labels

        pred : array, shape (n_samples,)
            Predicted labels

        sample_weight : array-like, shape (n_samples,), optional
            Sample weights.
        """
        # logaddexp(0, v) == log(1.0 + exp(v))
        pred = pred.ravel()
        if sample_weight is None:
            return -2.0 * np.mean((y * pred) - np.logaddexp(0.0, pred))
        else:
            return (-2.0 / sample_weight.sum() *
                    np.sum(sample_weight * ((y * pred) - np.logaddexp(0.0, pred))))

    def negative_gradient(self, y, pred, **kargs):
        """Compute the residual (= negative gradient).

        Parameters
        ----------
        y : array, shape (n_samples,)
            True labels

        pred : array, shape (n_samples,)
            Predicted labels
        """
        return y - expit(pred.ravel())
```

Here, `pred` is the $\hat{y}$ in the previous section.
The loss function computed by `__call__` agrees with formula $(\ref{L_i_yhat})$.
The gradient computed by `negative_gradient` agrees with formula $(\ref{boost-gradient})$.
(`expit` is the logistic function imported from `scipy.special`.)

The parameter `subsample` allows Bagging-like behavior but is disabled by default.

The parameter `max_features` allows Random-Forest-like behavior but is disabled by default.

### `fit`

`GradientBoostingClassifier.fit` directly uses the parent
`BaseGradientBoosting.fit`.

Skipping all sanity checks and most non-default or non-binary-class scenarios,
the most important steps are as follows:

1. Set `self.classes_ = [0, 1]` and `self.n_classes_ = 2`.

2. Set `self.loss_ = BinomialDeviance(2)`.
Unpon this setting, `self.loss_.K` is 1, the number of regression trees to be induced.

3. Set `self.init_ = self.loss_.init_estimator()`, which effectively does
`self.init_ = LogOddsEstimator()`.

4. Fit initial model: `self.init_.fit(X, y)`.
The effect of this operation is
`self.init_.prior = np.log(pos / neg)`,
where `pos` and `neg` are counts of 1's and 0's, respectively, in the response vector.
This value is the global log odds, $\log\frac{p}{1 - p}$.

5. Make initial predictions: `y_pred = self.init_.predict(X)`.
The effect of this assignment is that
`y_pred` is a column vector of the length of `y`, filled with the value of `self.init_.prior`.

   `self.init_` plays the role of $F_0$ in the expositions above.
   Its `predict` method returns the global log odds.

6. Start iterations (see `BaseGradientBoosting._fit_stages`).
The workhorse in `_fit_stages` is `_fit_stage`, which is called once in each iteration,
taking `X`, `y`, `y_pred`, and returning updated `y_pred`, to be used in the next iteration.
`X` and `y` are the original data, and they do not change in the iterations.

For binary classification, `_fit_stage` fits a `sklearn.tree.DecisionTreeRegressor`;
most work is delegated to the latter class.

Before diving into `_fit_stage`,
recall that the initial value of `y_pred` is the global log odds.
By $(\ref{logit})$ we know $\hat{y}$ is the log odds,
so, indeed, the initial `y_pred` is the additive part corresponding to
an estimate of the global probability (the fraction of positive responses).
This is a reasonable starting point.

The main steps in `_fit_stage` are as follows.

1. Compute `residual = self.loss_.negative_gradient(y, y_pred)`.
This `residual` is $y - \hat{p}$, i.e. negative of the gradient derived in $(\ref{boost-gradient})$.

2. Fit a `DecisionTreeRegressor` with `X` as $x$ and `residual` as $y$.
An interesting parameter passed to `DecisionTreeRegressor` is
`criterion=self.criterion`, which is `'friedman_mse'` by default.
This criterion uses mean squared error with Friedman's improvement score
to assess potential splits (see `sklearn.tree._criterion.FriedmanMSE`).


3. The tree built in the previous step has settled the node splits.
Now ignore the leaf values and re-determine them in `self.loss_.update_terminal_regions`.
This does two things:

   1. Determine each leaf's prediction value as this:
      `np.sum(residual) / np.sum((y - residual) * (1 - y + residual))`.
      The summation is over data samples that fall in the particular leaf.
      Because `residual` is $y - \hat{p}$, this leaf value is
      $\frac{\sum_i (y_i - \hat{p}_i)}{\sum_i \hat{p}_i (1 - \hat{p}_i)}$,
      where the data point $x_i$ falls in the leaf in question.

      This formula comes from Friedman's original paper.
      The numerator is the sum of residuals (or negative gradients);
      the denominator is the sum of the second derivatives (see fomular $(\ref{loss-hessian})$ above).

      This new tree is the $f_m$ in the previous expositions.
      By now it has been fully determined and appended to the ensemble of base learners.

   2. Increment `y_pred` of observation $i$ by `self.learning_rate` times the leaf value
      in which this observation falls.
      This is the "boosting" step. The learning rate shrinks the prediction of the new tree
      to prevent overfitting.
      The updated vector `y_pred` is the return value
      of `_fit_stage`.

### `predict_proba`

Given an $x$, `predict_proba` retrieves the log-odds value stored in `self.init_` (think $F_0$),
and collects the value of the leaf where this $x$ belongs in each of the trees (think $f_m$),
multiplied by `self.learning_rate`, and sum them up.
This is exactly the `y_pred` as it goes through the iterations in `fit`.
In short, this computes $\hat{y}$.

In principle, $x$ can be walked down the ensemble of trees in parallel.
(The scikit-learn implementation is sequential because the data structure does not benefit
from parallelism on this operation.)
In contrast, the training of the trees has to be done sequentially.

Finally, the logistic function converts $\hat{y}$ to $\hat{p}$, which is returned.


### References

- [Blog post by Terence Parr and Jeremy Howard](https://explained.ai/gradient-boosting/index.html)

- [Q & A on StackExchange Stats](https://stats.stackexchange.com/questions/157870/scikit-binomial-deviance-loss-function)

- [Q & A on StackExchange Stats](https://stats.stackexchange.com/questions/204154/classification-with-gradient-boosting-how-to-keep-the-prediction-in-0-1)

- [Q & A on StackExchange Stats](https://stats.stackexchange.com/questions/330849/how-do-newton-raphson-updates-work-in-gradient-boosting)


- [Q & A on StackExchange Math](https://math.stackexchange.com/questions/1153655/newtons-method-vs-gradient-descent-with-exact-line-search)


- [Friedman's original paper from 1999](https://projecteuclid.org/euclid.aos/1013203451)

- [Newton's method in optimization](https://en.wikipedia.org/wiki/Newton%27s_method_in_optimization)
