---
layout: post
title: Machine Learning and Econometrics - a rewarding relationship?
date: 2021-01-28
categories: blog
tag: Machine Learning, Economics
permalink: /ml-and-metrics/
---
Disclaimer: these are my thoughts on the subject based on my 2+ years of learning machine learning and my experiences talking to seniors (who've actually studied econometrics). No doubt my views on this will change next year, and everything I say about econometrics now should probably be fact-checked rigorously before it ends up anywhere else.

I'd like to argue on the merits of an understanding of both machine learning and econometrics, in spite of their wildly varying methodologies.

I'm quite sure the importance of machine learning and big data have been espoused across every applied field imaginable, whether in academia or in industry. Economics is no exception. With techniques like natural language processing and web scraping, economists can easily scrape platforms like twitter to easily get data on and quantify "consumer sentiment" - something that's been historically hard to quantify. For more examples on how machine learning may be applied to economic research, check out Dr Stephen Hansen's work [here]([https://sekhansen.github.io/pdf_files/qje_2018.pdf]).

So we can get nice big datasets by borrowing techniques from machine learning - this is nothing new, and applies to pretty much every applied area of study out there. What makes economics different is that it already has time-tested methods in place for modelling behaviour - i.e. econometrics. I was talking to my senior recently about this to understand what second-year econometrics at LSE involved (I've got a slight obsession with planning ahead), and the conversation set me thinking about some interesting contradictions.

## Conflicting Approaches to Linear Regression ##
In econometrics, linear models reign supreme, and a significant amount of time in Y2 Econometrics is spent exploring the _Normal Equation_ - a solution to optimising linear regression. (The first section on the wikipedia page for econometrics is on Linear Regression). But if you're familiar with linear regression in a machine learning context, you'll know that the Normal Equation is very very very inefficient - it runs in something like O(n^2.4) to O(n^3) time. (What this means is that doubling the dataset size could increase the time taken to train the model by a factor of 8). This comes from the difficulty of inverting a matrix, which is needed to solve the Normal Equation.

This inefficiency is what motivates _Gradient Descent_ - an iterative algorithm to solve optimization problems, used everywhere in machine learning. I've talked about this in my [post on love song classification]([https://ethan-cheong.github.io/love-song-classification-1/]), and it's a very elegant solution to optimizing a wide range of problems. It empirically has a much better time complexity than a normal equation - so what can we conclude from this? Are economists being inefficient because they refuse to learn about big O time complexity?

Well, economics is quite old (at least in comparison to statistical learning) - Normal Equations are probably sufficient for datasets economists look at (used to look at?). Contrast this with machine learning, where we might have millions of observations in a dataset - inverting the matrix of features alone might take hours, motivating the use of gradient descent. This ties into this idea of _intent_ - the tools and methods economists/data scientists used aren't better or worse than each other; in fact, they're perfectly suited to what they need. We'll look at another example of this.

## Why Linearity at all? ##
Our discussion of linear regression begs the question - from a machine learning perspective, why would we ever want to use linear regression over the smorgasboard of other algorithms in our toolbox? We could throw our training observations into [XGBoost]([https://ethan-cheong.github.io/love-song-classification-1/]) or some Neural Network and get something like 0.05 MSE easily. If we really want to restrict ourselves to linearity we could use a generalized linear model (with some regularization thrown in to prevent overfitting). Why would we want to use old-fashioned linear regression?

As before, the answer is _intent_ - machine learning is primarily concerned with _prediction_ (feel free to disagree with me on this). Just take a look at [kaggle.com]([https://www.kaggle.com/competitions]) - users compete in making predictions, their submissions judged based on test accuracy. Thousands of dollars have been won on the tiniest of margins (I'm not exaggerating - at the time of this post, Jane Street has a Market Prediction Competition going on, with the first prize a cool $100,000). Using a regression in one of these competitions would probably get you something like 60% accuracy (trust me, I've tried). Better than random guessing, sure, but not really what a bank would want out of an algorithmic trading strategy. The algorithms used to win these competitions are nasty and convoluted, but they get the job done.

Contrast this with econometrics, where _interpretability_ is much more important - so important, in fact, that we aren't just content with finding a linear relationship between factors; we have to do one better and try to unconfound them (since correlation != causation). And if we're talking about interpretability, regression is the undisputed champion. If you disagree with me on this, look at the diagram below and tell me with confidence you have a decent idea how the model works (this is BERT, used to predict sentiment from text):

![BERT Architecture](/assets/mlmetrics/bert.png)

Compare this to a linear regression that might try to answer the same problem:

![Linreg](/assets/mlmetrics/plot.png)

This model doesn't fit the data superbly, but it helps us to better understand the relationship between the number of exclamation marks in a text message and whether the author was angry or not. Will this help us make predictions? Maybe - but it definitely won't be as accurate as BERT. Nonetheless, it sheds light on a linear relationship between factors, something that BERT just can't do (Whether there's any causation between them is something else entirely).

# So what?
What does this all mean? Well, even in a data science context, _interpretability is important_. If a model performs well, why not just trust it and ignore why it made a decision, regardless of how complicated it made things? [This paper]([https://arxiv.org/abs/1702.08608]) goes into it in more depth, but to summarize:
* A single metric (like Mean Squared Error / Classification Accuracy) is an incomplete description of real-world task. Learning why something happened might give you reasons why a model failed/succeeded, and let you improve it further
* Debugging a model (say if it fails miserably on a new prediction set) is much easier when you know how it works
* You may need to justify how your model works to others (maybe a boss), and understanding how a model works doesn't mean you'll be able to explain it simply to someone without the expertise (they may just get bored).

What this means is that beyond learning about linear models, there's a non-insignificant reason to learn about econometrics, even as an ML practitioner (thankfully validating my university course choice).

What if you're an economist? Well I can't speak to the feasibility of using modern machine learning methods for economic modelling (although methods that marry interpretability and accuracy like Boosted Trees do exist). I'll have more thoughts on this once year 2 starts.
