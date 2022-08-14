---
layout: post
title: Understanding Word Embedding by word2vec
use_math: true
excerpt_separator: <!--excerpt-->
tags: [nlp]
---

I read a few sources explaining "word embedding", especially as carried out by the `word2vec` algorithm.
I felt there is still some intuitive clarity to be desired on a high level.
Here I'll attempt to describe it in my own words.
<!--excerpt-->
The objective is to have a really *easy-to-remember* explanation on "what it is",
without diving into "how it is done" (which is kind of the meat of it).

The concept is simple: *word embedding* means representing each word in a vocabulary by a dense numerical vector of a specified length.

The emphasis on "dense" is mainly to contrast with the straightforward "one-hot encoding",
which is *sparse* and is nothing more than a categorical label.

Word embedding has three oft-mentioned benefits:

1. It is prerequisite in order for texts to be treated in Machine Learning models, because most if not all models can only take numbers.

2. The geometric relationships (e.g. distance, translation) between the numerical vectors "preserve" some semantic and syntactic relationships between the words. The
$\boldsymbol{v}\_{\text{king}} - \boldsymbol{v}\_{\text{man}} + 
\boldsymbol{v}\_{\text{woman}} \approx \boldsymbol{v}\_{\text{queen}}$
example is the poster child.

3. Dimension reduction or compression, especially in comparison with one-hot encoding.

Suppose we have a vocabulary $V$ with $n$ words.
We will represent each word by a numerical vector of length $k$.
These vectors need to be learned by a model.
The model will be trained to minimize a loss function that is defined based on
some meatric of the "closeness" between words in $V$.
The closeness metric is empirically extracted from a large corpus
that covers the vocabulary $V$.

Let's get specific.
The `word2vec` algorithm uses local co-occurrence of words to define closeness.
Specifically, in the example sentence below,

<center>
<span style="background-color: LawnGreen; padding:2mm">Cat and</span>
<span style="background-color: yellow; padding:2mm">dog</span>
<span style="background-color: LawnGreen; padding:2mm">both run</span>
fast</center>

we treat "dog" as the "focus" word and the two words before it ("cat", "and") and the two words after it ("both", "run") as "context" words. (Of course, the size of the context window is up to our decision.) Similarly, we can treat each of the other words as the focus word and identify the context words in the corresponding window.
This way, a *closeness* relation is established between the *focus* word and its *context* words.

Suppose the example sentence above is our entire corpus, then we have a vocabulary of size 6. Let's assign indices $1,2,...,6$ to the words in the order they appear in that sentence.
Each word will get two representations, used depending on the role:
one representation is used when the word is considered a "focus word",
and the other is used when the word is considered a "context word".
We'll denote the two representations by $\boldsymbol{v}$ (focus) and $\boldsymbol{u}$ (context), respectively,
both of length $k$.

Now treat the colored window as a single data sample.
Define two sets of words (or word indices):
$\boldsymbol{F}$, of size $n_f$, is the set of focus word(s);
$\boldsymbol{C}$, of size $n_c$, is the set of context words.
Although there is only a single focus word in this example (and in the typical literature),
we will pretend $\boldsymbol{F}$ can have multiple elements, just like $\boldsymbol{C}$.
The following derivation will show that the single-focus-word situation is merely a special case
that does not require any special treatment.
For our data sample,
$\boldsymbol{F} = \\{ 3 \\}$, and
$\boldsymbol{C} = \\{ 1, 2, 4, 5 \\}$.

There are two ways to use this data sample:

1. Given the focus words, how can we predict the context words?
In `word2vec`, this is called the "Skip-Gram" formulation.

2. Given the context words, how can we predict the focus words?
In `word2vec`, this is called the "Continuous Bag-of-Words", or CBOW, formulation.

Now that we allow $\boldsymbol{F}$ to have multiple elements,
the two fomulations are the same except for some swap of notations.
Let's take Skip-Gram.
In this formulation, given the set of focus words,
the target that we want the model to get to is the correct probability distribution of the context words over the vocabulary, that is,

$$
\begin{bmatrix}
1/4 \\
1/4 \\
0 \\
1/4 \\
1/4 \\
0
\end{bmatrix}
$$

More generally, we can write the target as

$$\begin{equation}\label{y}
\boldsymbol{y} = 
\begin{bmatrix}
y_1 \\
\vdots \\
y_j \\
\vdots \\
y_n
\end{bmatrix}
,

\quad
\text{where }
y_j =
\begin{cases}
&1/n_c,\; &\text{if $j \in \boldsymbol{C}$} \\
&0,\;   &\text{otherwise}
\end{cases}
\end{equation}
$$

We take the mean vector (element-wise mean) of the focus words,

$$\begin{equation}\label{mean-input}
\boldsymbol{v} = \frac{1}{n_f}\sum_{i\in \boldsymbol{F}} \boldsymbol{v}_i
\end{equation}
$$

Then, we use the inner product of $\boldsymbol{v}$
with each potential context word in the vocabulary as the basis of some "closeness" measure,
and take softmax to arrive at a predicted distribution, i.e. vector of probabilities:

$$\begin{equation}\label{yhat}
\hat{\boldsymbol{y}} = 
\begin{bmatrix}
\hat{y}_1 \\
\vdots \\
\hat{y}_j \\
\vdots \\
\hat{y}_n
\end{bmatrix}
,
\quad
\text{where }
\hat{y}_j = 
\frac{\langle \boldsymbol{v}, \boldsymbol{u}_j \rangle}
     {\sum_{j'=1}^n \langle \boldsymbol{v}, \boldsymbol{u}_{j'} \rangle}
\end{equation}
$$

A reasonable loss function for estimating the distribution $\boldsymbol{y}$
by $\hat{\boldsymbol{y}}$ is the cross-entropy:

$$\begin{equation}\label{skip-gram-loss}
J = -\sum_{j=1}^n y_j \log \hat{y}_j
  = -\sum_{j\in \boldsymbol{C}} \frac{1}{n_c}
    \log \hat{y}_j
\end{equation}
$$

Let's recap:
for vocabulary $V$, we aim to estimate two sets of word vectors:
$\\{ \boldsymbol{v}\_i \\}$ for the words as focus words, and
$\\{ \boldsymbol{u}\_i \\}$ for the words as context words.
Given a single data sample with focus words $\boldsymbol{F}$ and context words $\boldsymbol{C}$,
the model predicts the distribution of $\boldsymbol{C}$ over the vocabulary.
The response is written as $\boldsymbol{y}$ as in $(\ref{y})$, 
which is directly determined by $\boldsymbol{C}$.
However, the input $\boldsymbol{x}$ is not 
$\\{ \boldsymbol{v}\_i;\; i\in \boldsymbol{F}\\}$,
because these vectors are unknown.
Instead, we'll use the one-hot encodings of the words in $\boldsymbol{F}$ as the model input,
and find a mechanism to translate this input to 
$\\{ \boldsymbol{v}\_i;\; i\in \boldsymbol{F} \\}$, and then to $(\ref{mean-input})$.
The model makes a prediction $\hat{\boldsymbol{y}}$ as in $(\ref{yhat})$.
Using the loss function $(\ref{skip-gram-loss})$,
which involves the $\boldsymbol{v}$ vectors of the focus words and the $\boldsymbol{u}$ vectors
of the entire vocabulary,
we can estimate the $\boldsymbol{v}$ and $\boldsymbol{u}$ vectors.
(With a lot more data samples, the loss $J$ will involve both the $\boldsymbol{v}$ vectors
and the $\boldsymbol{u}$ vectors  of the entire vocabulary.)

This is the high-level idea of the word vectors.
The rest is how to estimate them.

`word2vec` designs
a neural network with an input layer, a single hidden layer, and an output layer.
The hidden layer has $k$ neurons.
The output layer (effectively) has $n$ neurons.
In the Skip-Gram architecture,
the input layer has $n_f$ channels each of length $n$.
The weights between the input and hidden layers are the $\boldsymbol{v}$ vectors (transposed as needed),
whereas
the weights between the hidden and output layers are the $\boldsymbol{u}$ vectors (transposed as needed).
After training this network, the $\boldsymbol{v}$ vectors ("input weights") and $\boldsymbol{u}$ vectors ("output weights") are both estimated. One can take either of them as the word embeddings to be used in other applications.

The idea apparently allows more flexibilities than the Skip-Gram and CBoW formulations.
For example, we could predict what word immediately follows a given word.
The empirically constructed $\boldsymbol{y}$ for this information
not only contains multiple non-zero elements, 
but can have different positive values indicating varying probabilities.
This can be handled easily by the cross-entropy loss function.
However, the word embeddings learned by this model tend to be used in applications that
do not have direct connections with the "closeness measure" in the training data
(e.g. whether one word immediately follows another).
For this reason, it requires empirical evaluation to determine
whether a particular training data captures the kind of inter-word relationships that are important for a target application. 



## References

[word2vec Parameter Learning Explained](https://arxiv.org/abs/1411.2738) by Xin Rong, 2016.
