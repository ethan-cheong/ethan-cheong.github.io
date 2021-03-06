---
layout: post
title: The Traditional Marriage Algorithm in Python
date: 2020-12-24
categories: blog
tag: Mathematics, Economics, Python
mathjax: true
permalink: /traditional-marriage/
---
Whilst revisiting assignments for my introductory analysis course over winter break, I chanced upon this problem involving the traditional marriage algorithm. I had glanced over this question during the term-time, but recently I've become interested in object-oriented programming, and thought it would be worth taking a second look.

The code for this post can be found [here](https://github.com/ethan-cheong/randomWalks/tree/main/traditionalMarriage), and if you'd like to give it a try, the full problem set can be found [here](\assets\traditionalmarriage\03ex.pdf) (all credits to Professor Peter Allen).

The problem is as follows: we have _n_ men and _n_ women, and need to pair them off. Each man wants to marry a woman, and vice versa. Furthermore, each man and woman have a personal ranking of the member of the other sex they'd like to marry. This problem can represent a variety of real-world situations - one notable example is assigning graduating medical students to their first hospital appointments.

We have to pair off all the men and women. However, if there exists a man and woman who both prefer each other to their assigned partner, then they will cheat. We say an arrangement is _stable_ when there are no such cheating pairs.

The traditional marriage algorithm is as follows:
1. Each night, every man visits his favourite woman amongst those who haven't rejected him
2. Each woman looks at the men visiting her, and rejects all but her most preferred among that collection.
3. Repeat until the night where every woman is visited by exactly one man, where they get married.

The fact that this problem reduces in size after each iteration (as well as the fact that it was in an induction problem set) hints that the solution to the problems above involve induction - we can use induction to prove that the pairs produced by this algorithm are always stable.

In my implementation, I emphasize intuitive code over performance - you may well see examples of terrible practices, like several nested for loops - as I'm more focused on making the logic easily comprehensible. This could be a useful starting point for someone looking to understand the logical flow before implementing a more efficient version.

### Implementing the Algorithm ###
Our plan of attack is as follows:
1. Find a way to represent a man, woman and their preferences.
2. Implement the traditional marriage algorithm
3. Visualization of the algorithm
4. Check that arrangements are stable

First, let's make representations for a person. We'll give each person an `identity` from 0 to n-1, following python's counting convention (with n being the number of pairs). We represent each person's preferences as a list and randomize them, then convert these to a deque. Compared to lists, deques have better performance for popping entries from the front and back.
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
{% endhighlight %}
Next, we can make classes for men and women that inherit from `Person`. Men will have the `getRejected` function, and an additional list `remaining_preferences` to track the women that haven't rejected them. Although the list of their preferences is shortened as nights pass, we will need their original preferences after the algorithm terminates so we can check for stability. We can also change their preferences with `setRemainingPreferences` - letting us represent men with non-random preferences.
{% highlight python %}
class Man(Person):
    def __init__(self, identity, n):
        # Representation of a man
        super().__init__(identity, n)
        self.remaining_preferences = copy.copy(self.preferences)

    def getRemainingPreferences(self):
        return self.remaining_preferences

    def setRemainingPreferences(self, input):
        self.remaining_preferences = collections.deque(input)

    def getRejected(self):
        self.remaining_preferences.popleft()
{% endhighlight %}
Women need several additional characteristics: they have a `man_list`, which shows the men that visit them each night. We need to be able to add men to this list, as well as clear the list after each night. They also need the ability to choose their most preferred men, as well as generate a list of men that they reject. Finally, we should also add the ability to adjust their preferences.
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

    def setPreferences(self, input):
        self.preferences = collections.deque(input)
{% endhighlight %}
`initializePeople` creates 2 arrays with n men and women each. I chose to keep this in a separate function so we could tweak the arrays we give our main function later if necessary.
{% highlight python %}
def initializePeople(n):
    # Initialize arrays of men and women
    men = [Man(i, n) for i in range(n)]
    women = [Woman(i, n) for i in range(n)]
    return (men,women)
{% endhighlight %}
`nightIteration` does the bulk of the work - it implements the sequence of events that happens each night. The implementation is VERY inefficient due to the number of nested for loops - although I've decided to leave them in, since they capture the logic that's used in the algorithm itself, which will be useful when visualizing the algorithm in action. There should be a way to optimize this with matrix algebra - feel free to fork the repo if you'd like to give it a try.
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
{:refdef: style="text-align: center;"}
![image1](/assets/traditionalmarriage/image1.png){: style="float: center"}
{: refdef}
Of course, it would be nice if we had a visualization of the algorithm in action.
### Visualizing the Algorithm ###
We can use `matplotlib` to make animations showing how the algorithm works. Firstly, we need to update our classes for men and women to contain modifiable row and column positions, so that they can be represented and manipulated in 2D space.
{% highlight python %}
class Man(Person):
    def __init__(self, identity, n):
        super().__init__(identity, n)
        self.remaining_preferences = copy.copy(self.preferences)
        # set starting coordinates for visualization later
        self.row_position = 2 * n - 1        # Puts the men in the second row from the bottom
        self.col_position = 2 * identity + 1 # Leaves one column between each man

    def getPosition(self):
        return (self.row_position, self.col_position)

    def updatePosition(self, new_row_position, new_col_position):
        self.row_position = new_row_position
        self.col_position = new_col_position

    def resetPosition(self):
        self.row_position = 2 * self.n - 1
        self.col_position = 2 * self.identity + 1
{% endhighlight %}
Since women won't have to move (the men will be going to them), they only need a `getPosition` function so the men know their position.
{% highlight python %}
class Woman(Person):
    def __init__(self, identity, n):
        # Representation of a woman
        super().__init__(identity, n)
        self.man_list = []
        self.row_position = 1
        self.col_position = 2 * identity + 1

    def getPosition(self):
        return (self.row_position, self.col_position)
{% endhighlight %}
Now, we need some way to represent the space the men and women are in - I chose to use numpy arrays because they're easy to initialize and manipulate.

The function below returns a numpy array with women represented by '0.7's, men with '0.3's and empty spaces with zeroes. The reason for this seemingly arbitrary choice will be apparent soon.
{% highlight python %}
import numpy as np

def updateWorld(men_array):
    # Function to plot the world based on current positions of men
    n = len(men_array)
    world = np.zeros((2*n+1, 2*n+1))
    # Fill in females
    for i in range(1,2*n+1,2):
            world[1, i]=0.7
    # Fill in males
    for man in men_array:
        current_row_position = man.getPosition()[0]
        current_col_position = man.getPosition()[1]
        world[current_row_position, current_col_position] = 0.3
    return world
{% endhighlight %}
Let's see what it looks like when we start a world with 5 pairs of people:
{% highlight python %}
men_array, women_array = initializePeople(5)
world = updateWorld(men_array)
print(world)
{% endhighlight %}


![image2](/assets/traditionalmarriage/image2.png)


We see that we get a nice array which we can turn into a visualization. `matplotlib` has a convenient function `imshow` which outputs a grid of colours corresponding to each entry in an array. The colour mapped depends on the numerical value in each array entry; here's what it looks like when we apply it to our world:
{% highlight python %}
import matplotlib.pyplot as plt

plt.imshow(world, cmap="inferno", vmin=0, vmax=1)
{% endhighlight %}

![image3](/assets/traditionalmarriage/image3.png)

This is a good starting point! Each orange square represents a man and each purple square a woman. Next, we need a way to make the men moving towards the women they're courting; `moveMen` does that by checking the position of each man with respect to their desired partner, and then moving them by one square.
{% highlight python %}
def moveMen(world, men_array, women_array):
    for man in men_array:
        current_row_position = man.getPosition()[0]
        current_col_position = man.getPosition()[1]
        favourite_woman = man.getRemainingPreferences()[0]
        goal_row_position = women_array[favourite_woman].getPosition()[0]
        goal_col_position = women_array[favourite_woman].getPosition()[1]

        if current_col_position == goal_col_position and current_row_position == goal_row_position:
            # Men will not move if they are in the same position as their desired partner.
            pass
        elif current_col_position == goal_col_position:
             # Men will move vertically by one square if they are in the same column as their desired partner.
            man.updatePosition(current_row_position - 1, current_col_position)
        elif current_col_position > goal_col_position:
            # Men will move horizontally if they are in the wrong column. Here they move to the left.
            man.updatePosition(current_row_position, current_col_position - 1)
        elif current_col_position < goal_col_position:
            # Here they move to the right.
            man.updatePosition(current_row_position, current_col_position + 1)
{% endhighlight %}
We also have to change our `updateWorld` function to account for when men and women stand on the same square, or when two men stand on the same square:
{% highlight python %}
def updateWorld(men_array):
    n = len(men_array)
    world = np.zeros((2*n+1, 2*n+1))
    # Fill in females
    for i in range(1,2*n+1,2):
            world[1, i]=0.7
    # Fill in males
    for man in men_array:
        current_row_position = man.getPosition()[0]
        current_col_position = man.getPosition()[1]
        if world[current_row_position, current_col_position] > 0.7:
            # Change the colour if there are men who reach the female tile.
            world[current_row_position, current_col_position] += 0.08
        elif world[current_row_position, current_col_position] == 0:
            # Change the colour if a man steps on an empty tile.
            world[current_row_position, current_col_position] = 0.3
        else:
            # Change the colour if two men step on the same tile.
            world[current_row_position, current_col_position] += 0.1
    return world
{% endhighlight %}
![gif1](/assets/traditionalmarriage/gif1.gif){: style="float: right"}

The functions above allow us to move men one step at a time and plot out each step. We can then combine these with our main algorithm functions to get our full animation (the code can be found in `animations.py` on my github if you'd like to play around with it).

The example to the right pairs up 20 men with 20 women; we can see that for this particular combination of men and women, the algorithm took 15 iterations to get our pairs. Remember that the number of iterations changes depending on the (unseen) personal preferences of each man and woman!

![gif2](/assets/traditionalmarriage/gif2.gif){: style="float: right"}

Next, here's what happens when you have 5 men, all with the same preferences in women. We see that it takes 5 nights, and you can see each of the remaining men (besides the one that was selected by the woman) choosing the next woman on their list to propose to.

This next one is interesting - I've changed the code so that each man and woman has a different colour, so we can see more clearly which man visits which woman, and track how each individual changes their choice of partner as the nights go on.

![gif4](/assets/traditionalmarriage/gif4.gif){: style="float: left"}

Notice how for the first four nights, the leftmost woman (yellow) is only visited by the teal man. However, by night 5, only the green man is visiting her, and by the final night, she ends up marrying the dark blue man!

It's evident what happens from the animation - after being rejected by the orange woman on night 3, the green man visits the yellow woman together with the teal man on night 4, and since he's higher on her list of preferences, the yellow woman now rejects the teal man. The algorithm accounts for these changes in preferences by only terminating when all women have found a single match.

Finally, just for fun, here's the algorithm at work on 50 pairs of men and women.
{:refdef: style="text-align: center;"}
![gif3](/assets/traditionalmarriage/gif3.gif){: style="float: centre"}
{: refdef}

### Checking for Stability ###
Now that we have our algorithm in action, we need to check that it gives us _stable_ pairs.

First of all, we can update our `Person` class so that they can be assigned a partner.
{% highlight python %}
class Person:
    def __init__(self, identity, n):
        self.identity = identity
        self.n = n
        self.preferences = createPreferences(identity, n)
        self.partner_identity = None  # Initialize with no partner ID

    def setPartnerIdentity(self, input):
        self.partner_identity = input

    def getPartnerIdentity(self):
        return self.partner_identity
{% endhighlight %}
Next, when the traditional marriage algorithm terminates, we need to assign each woman and man their partners:
{% highlight python %}
def traditionalMarriage(men, women):
    nights = 0
    while True:
        if all(len(woman.getManList()) == 1 for woman in women):
            print("Matching took " + str(nights) + " nights!")
            # Assign partners
            for woman in women:
                woman.setPartnerIdentity(woman.getManList()[0])
            for man in men:
                man.setPartnerIdentity(man.getRemainingPreferences()[0])
            break
        # ...
{% endhighlight %}
The function below takes in arrays of men and women, and returns `True` if there are no cheating pairs.
{% highlight python %}
def checkStability(men, women):
    for man in men:
        for woman in women:
            if man.getPreferences().index(man.getPartnerIdentity()) >      \
               man.getPreferences().index(woman.getIdentity()) and         \
               woman.getPreferences().index(woman.getPartnerIdentity()) >  \
               woman.getPreferences().index(man.getIdentity()):
                  return False
    return True
{% endhighlight %}
Running the code below:
{% highlight python %}
bool_list = []
for i in range(1000):
    men, women = initializePeople(100)
    traditionalMarriage(men, women)
    bool_list.append(checkStability(men, women))

sum(bool_list) # Remember that when performing arithmetic on boolean values, True is 1 and False is 0
{% endhighlight %}
We get an output of `1000`. Thus, the algorithm gave stable arrangements 1000/1000 times when we had 100 pairs of men and women.

I hope this post gave you a nice little taster to how we can use python to model real-life situations using object-oriented programming - it's certainly given me some ideas for future projects.
