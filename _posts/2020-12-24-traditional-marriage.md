---
layout: post
title: The Traditional Marriage Algorithm in Python
date: 2020-12-24
categories: blog
tag: Python, Mathematics, Economics
mathjax: true
---
_If you'd like to try the code out for yourself, the scripts can be found [here](https://github.com/ethan-cheong/randomWalks/tree/main/traditionalMarriage)_

Whilst (tragically) revisiting assignments for my introductory analysis course over winter break, I chanced upon this problem involving the traditional marriage algorithm. I had glanced over this question during the term-time, but recently I've become interested in the similarities between discrete math and object-oriented programming, and thought it would be worth taking a second look. If you'd like to give it a try, the full problem set can be found [here](/assets/traditionalmarriage/03ex.pdf) (all credits to Professor Peter Allen).

The problem is as follows: we have _n_ men and _n_ women, and need to pair them off. Each man wants to marry a woman, and vice versa. Furthermore, each man and woman have a personal ranking of the member of the other sex they'd like to marry.

We have to pair off all the men and women. However, if there exists a man and woman who both prefer each other to their assigned partner, then they will cheat. We say an arrangement is _stable_ when there are no such cheating pairs.

The traditional marriage algorithm is as follows:
1. Each night, every man visits his favourite woman amongst those who haven't rejected him
2. Each woman looks at the men visiting her, and rejects all but her most preferred among that collection.
3. Repeat until the night where every woman is visited by exactly one man, where they get married.

The fact that this problem reduces in size after each iteration (as well as the fact that it was in an induction problem set) hints that the solution to the problems above involve induction. And something I noticed during the term was the relationship between induction and recursion - both involve a base case, and both involve Professor Allen's somewhat disliked phrase "and so on..." I thought it an interesting link between something I'd learned in my Math degree and a coding concept that I'd picked up in an MOOC, and thought it'd be worth exploring in greater depth.

### Applying the algorithm ###
Let's try applying the traditional marriage algorithm. Our plan of attack is as follows:
1. Find a way to represent a man, woman and their preferences.
2. Implement the traditional marriage algorithm
3. Visualization of the algorithm
4. Check that arrangements are stable
5. Prove that the arrangements given by the algorithm are always stable.
6. Try out other situations - e.g. if men or women lie about their preferences.

First, let's make representations for a person. We'll give each person an identity from 0 to n-1 (with n being the number of pairs, and following python's counting convention). We represent each person's preferences as a list and randomize them, then convert these to a deque. Compared to lists, deques have better performance for popping entries from the front and back.
{% highlight python %}
import random
import collections # For improved time complexity

def createPreferences(identity, n):
    pref_list = [i for i in range(n)]
    random.shuffle(pref_list) # randomize the order of people's preferences
    pref_deque = collections.deque(pref_list)
    return pref_deque

class Person:
    def __init__(self, identity, n):
        # Representation of a person
        self.identity = identity
        self.preferences = createPreferences(identity, n)

    def getIdentity(self):
        return self.identity

    def getPreferences(self):
        return self.preferences

    def setPreferences(self, input):
        # Takes a list as input, allowing us to set custom preferences
        self.preferences = collections.deque(input)
{% endhighlight %}
Next, we can make classes for men and women that inherit from `Person`. Men will have the `getRejected` function, and an additional list `remaining_preferences` to track the women that haven't rejected them. Although the list of their preferences is shortened as nights pass, we need their original preferences so that we can check for cheating at the end.
{% highlight python %}
class Man(Person):
    def __init__(self, identity, n):
        # Representation of a man
        super().__init__(identity, n)
        self.remaining_preferences = copy.copy(self.preferences)

    def getRemainingPreferences(self):
        return self.remaining_preferences

    def getRejected(self):
        self.remaining_preferences.popleft() # time complexity O(1)
{% endhighlight %}
Women need several additional characteristics: they have a `man_list`, which shows the men that visit them each night. We need to be able to add men to this list, as well as clear the list after each night. They also need the ability to choose their most preferred men, as well as generate a list of men that they reject.
{% highlight python %}
class Woman(Person):
    def __init__(self, identity, n):
        # Representation of a woman
        super().__init__(identity, n)
        self.man_list = []

    def getManList(self):
        return self.man_list

    def addToList(self, man):
        self.man_list.append(man)

    def clearManList(self):
        # We need to reset the list of men visiting after every iteration
        self.man_list = []

    def chooseMan(self):
        # Given a list of men, pick the one whose identity is the leftmost entry
        # from your list of preferences.
        for preferred_man in self.preferences:
            if preferred_man in self.man_list:
                return preferred_man

    def getRejectedMen(self, chosen_man):
        # Get the men that the woman didn't choose each night so they can be rejected
        rejected_men = self.man_list[:]
        rejected_men.remove(chosen_man)
        return rejected_men
{% endhighlight %}
`initializePeople` creates 2 arrays with n men and women each. I chose to keep this in a separate function so we could tweak the arrays we give our main function later if necessary.
{% highlight python %}
def initializePeople(n):
    # Initialize arrays of men and women
    men = [Man(i, n) for i in range(n)]
    women = [Woman(i, n) for i in range(n)]
    return (men,women)
{% endhighlight %}
`nightIteration` does the bulk of the work - it implements the sequence of events that happens each night. The implementation is VERY inefficient due to the number of nested for loops - although I've decided to leave them in, since they capture the logic that's used in the algorithm itself, which will be useful when visualizing the algorithm in action. There's definitely a way to optimize this with matrix algebra - feel free to fork the repo if you're interested!
{% highlight python %}
def nightIteration(men_array, women_array):
    # Function for each night that passes
    for woman in women_array:
        woman.clearManList()

    for man in men_array:
        print("Man " + str(man.getIdentity()) + " has the preferences "
        + ','.join([str(i) for i in man.getRemainingPreferences()]) + ".")
        favourite_woman = man.getRemainingPreferences()[0]
        print("Man " + str(man.getIdentity()) + " is visiting woman " + str(favourite_woman) + "!")
        # add the men visiting them to their man_list
        women_array[favourite_woman].addToList(man.getIdentity())

    for woman in women_array:
        if woman.getManList():
            print("Woman " + str(woman.getIdentity())
            + " is being visited by man/men "
            + ','.join([str(i) for i in woman.getManList()]) + "!")
        preferred_man = woman.chooseMan()
        if preferred_man is not None:
            print("Woman " + str(woman.getIdentity()) + " chose man "
            + str(preferred_man) + "!")
            rejected_men = woman.getRejectedMen(preferred_man)
            for rejected_man in rejected_men:
                men_array[rejected_man].getRejected()
{% endhighlight %}
Finally, we have the `traditionalMarriage` function, which takes in arrays of men and women, and repeatedly calls `nightIteration` until each woman has exactly one man visiting them.
{% highlight python %}
def traditionalMarriage(men, women):
    if len(men) != len(women):
        print("We need the same number of men and women!")
    nights = 0
    # Iterate until each woman has exactly one man visiting them.
    while True:
        if all(len(woman.getManList()) == 1 for woman in women):
            print("Matching took " + str(nights) + " nights!")
            break
        else:
            nightIteration(men, women)
            nights += 1
            print(str(nights) + " nights have passed!")
{% endhighlight %}
When we run the algorithm with 5 pairs (`traditionalMarriage(*initializePeople(5))`), it looks something like this:
![image1](/assets/traditionalMarriage/image1.png)
Of course, it would be nicer if we had a visualization of the algorithm in action.
{% highlight python %}

{% endhighlight %}

{% highlight python %}

{% endhighlight %}
