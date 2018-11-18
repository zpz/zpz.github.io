---
layout: post
title: Understanding Word Embedding by word2vec
use_math: true
excerpt_separator: <!--excerpt-->
tags: [deep-learning, nlp]
---

I read a few sources explaining "word embedding", especially the `word2vec` algorithm.
I felt there is still some clarity to be desired on a high level.
Here I'll attempt to describe it in my own words.
<!--excerpt-->
The objective is to have a really *easy-to-remember* statement of "what it is".
I will not dive into "how it is done" (which is kind of the meat of it).

The concept is simple: "word embedding" means representing each word in a vocabulary by a dense numerical vector of a specified length.

The emphasis on "dense" is mainly to contrast with the straightforward "one-hot encoding",
which is *sparse* and is nothing more than a categorical label.

Word embedding has three oft-mentioned benefits:

1. It is prerequisite for treating the texts in Machine Learning models, because most if not all models can only take numbers.

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

we treat "dog" as the "focus" word and the two words before it ("cat", "and") and the two words after it ("both", "run") as "context" words. (Of course, the size of the context window is up to our decision.) Similarly, we can treat each of the other words as the focus word and identify the context words in the specific window.
This way, a *closeness* relation is defined between the *focus* word and its *context* words.

Suppose the example sentence above is our entire corpus, then we have a vocabulary of size 6. Let's assign indices $1,2,...,6$ to the words in the order they appear in that sentence.
Each word will get two representations, as a focus word and as a context word, respectively. I've made up these notations:

$$\begin{align*}
V_{\text{focus}} &= \{ \boldsymbol{v}_i;\; i=1,\ldots,n \} \\
V_{\text{context}} &= \{ \boldsymbol{u}_i;\; i=1,\ldots,n \}
\end{align*}
$$

Now treat the colored window as a single data point.
There are two ways to use this data point:

1. Given focus word "dog", what are the context words? The answer is "cat", "and", "both", "run". In `word2vec`, this is called the "Skip-Gram" formulation.

2. Given context words "cat", "and" "both", "run", what is the focus word? The answer is of course "dog". In `word2vec`, this is called the "Continuous Bag-of-Words", or CBOW, formulation.

Let's take Skip-Gram first.
In this formulation, the "input" is the single focus word, "dog".
What should be the "output" of the model and the loss for this single data point?


Note: take this is a multi-lable problem? When the loss and activation need to be verified.
Recommendations seem to be `binary_crossentropy` loss and sigmoid activation; seems to be using soft-max.