---
layout: post
title: The Traditional Marriage Algorithm in Python
date: 2020-12-24
categories: blog
tag: Python, Mathematics, Economics
mathjax: true
---

Whilst (tragically) revisiting assignments for my introductory analysis course over winter break, I chanced upon this problem involving the traditional marriage algorithm. I had glanced over this question during the term-time, but recently I've become interested in the similarities between discrete math and object-oriented programming, and thought it would be worth taking a second look. If you'd like to give it a try, the full problem set can be found [here](/assets/traditionalmarriage/03ex.pdf) (all credits to Professor Peter Allen).

The problem is as follows: we have _n_ men and _n_ women, and need to pair them off. Each man wants to marry a woman, and vice versa. Furthermore, each man and woman have a personal ranking of the member of the other sex they'd like to marry.

We have to pair off all the men and women. However, if there exists a man and woman who both prefer each other to their assigned partner, then they will cheat. We say an arrangement is _stable_ when there are no such cheating pairs.

The traditional marriage algorithm is as follows:
1. Each night, every man visits his favourite woman amongst those who haven't rejected him
2. Each woman looks at the men visiting her, and rejects all but her most preferred among that collection.
3. Repeat until the night where every woman is visited by exactly one man, where they get married.

The fact that this problem reduces in size after each iteration (as well as the fact that it was in an induction problem set) hints that the solution to the problems above involve induction. And something I noticed during the term was the relationship between induction and recursion - both involve a base case, and both involve Professor Allen's somewhat disliked phrase "and so on..." I thought it an interesting link between something I'd learned in my Math degree and a coding concept that I'd picked up in an MOOC, and thought it'd be worth exploring in greater depth.

### The Relationship between Induction and Recursion ###
Consider the following problem: Prove that \\( \forall n \in \mathbb{N}, \sum_{i=1}^n i = 0.5n(n+1)\\) We can solve this by induction:
* Let $$P(n)$$ represent the statement above.
* The base case $$P(1)$$ is true.
* Assume for our induction hypothesis that $$P(k)$$ holds. Thus, $$ \sum_{i=1}^k i = 0.5k(k+1) $$
* Next, we'll try to prove that $$P(k+1)$$ holds.

$$\begin{eqnarray}
\sum_{i=1}^{k+1} i
&=& k+1 + \sum_{i=1}^k i \\
&=& k+1 + 0.5k(k+1) \\
&=& 0.5k^2 + 1.5k + 1.5 \\
&=& 0.5(k+1)(k+2)
\end{eqnarray}$$

Which is $$P(k+1)$$.
* Hence, by the principle of induction, $$P(n)$$ applies for all natural numbers.

Now, consider the following problem: given _n_, we want to find the sum of all natural numbers up to n.
{% highlight python %}
def sumToN(n):
    if n <= 1:
        return n
    else:
        return n + sumToN(n - 1)
{% endhighlight %}
In general, for recursive functions, the argument has the form of an inductive proof based on an induction principle.

Let's apply this to the traditional marriage algorithm. Our plan of attack is as follows:
1. Find a way to represent a man, woman and their preferences.
2. Implement the traditional marriage algorithm (ideally in a visual manner)
3. Prove that the arrangements given by the algorithm are always stable.
4. Consider what happens if men lie and women lie about their preferences.

First, let's make representations for a person. We'll give each person an identity from 0 to n-1 (with n being the number of pairs, and following python's counting convention). We represent each person's preferences as a list and then randomize them.
{% highlight python %}
import random

def createPreferences(n, identity):
    pref_list = [i for i in range(n)]
    random.shuffle(pref_list) # randomize the order of people's preferences
    return pref_list

class Person:
    def __init__(self, identity, n):
        # Representation of a person
        self.identity = identity
        self.preferences = createPreferences(n, identity)

    def getIdentity(self):
        return self.identity

    def getPreferences(self):
        return self.preferences
{% endhighlight %}
Next, we can make classes for men and women that inherit from `Person`. Men will have the `getRejected` function, and an additional list to track the women that haven't rejected them. Although the list of their preferences is shortened as nights pass, we need their original preferences so that we can check for cheating.

We'll take the list of preferences to be in increasing ranking from front to back. This isn't just arbitrary - it's faster to pop elements from the back of the list than from the front.
{% highlight python %}
class Man(Person):
    def __init__(self, identity, n):
        # Representation of a man
        super().__init__(identity, n)
        self.remaining_preferences = self.preferences

    def getRemainingPreferences(self):
        return self.remaining_preferences

    def getRejected(self):
        self.remaining_preferences.pop()
{% endhighlight %}
Women will have the function `chooseMan`, which takes a list of men that visit them and returns the man highest on their list of preferences (or furthest to the right of the list)
{% highlight python %}
class Woman(Person):
    def __init__(self, identity, n):
        # Representation of a woman
        super().__init__(identity, n)

    def chooseMan(self, man_list):
        # Given a list of men, pick the one whos identity is the rightmost entry
        # on your list of identities.
{% endhighlight %}


{% highlight python %}

{% endhighlight %}


{% highlight python %}

{% endhighlight %}
