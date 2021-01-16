---
layout: post
title: Interactively Mapping Crime in the UK
date: 2021-01-16
categories: blog
tag: Python
permalink: /crime-map-uk/
---

First: Demonstration of the app in action!

I got this idea from here: https://shiny.rstudio.com/gallery/crime-watch.html
Felt like it could use alot more customization options.

So, here's a walkthrough of what our app can do:


I'll go through a high-level of overview of my process. As someone who doesn't have alot of experience with software/web development, this was admittedly quite a struggle, but I learned a lot from working at the whole thing from start to finish, and am glad I followed through with it.

Next:
Process behind developing the app:

I decided to use python for this project for the following reasons
* When coding in R, I sometimes feel “Wow, wouldn’t it be great if i could use a for loop here and not crash my computer for several minutes instead of turning this into a tibble and applying a loop”
* I wanted to build more things in python, and could use the practice with the syntax
* I’m more comfortable with R, which will make this more challenging and interesting
* The main reason I prefer R to python for data science projects is the ease of use of the tidyverse packages, which lets you quickly string together transformations with the pipe operator “%>%”.
* Possibility of combining this with other modern machine learning techniques, of which python admittedly has a better ecosystem for (it took me quite a while to realize this and even longer to admit it)

Collecting data - it's a BIG dataset (1.8 million entries? or something

Process behind building the app.
* Get datasets - find a way to append all the files into one big csv
* Start out with skeleton of all the html elements
* Code in all the interactivity between them that we're planning - this will take up the bulk of the app.
* Compiled aggregate csv for aggregate visualizations, so we dont have to call all these steps in python in our app - grouped counts can be expensive!
* Add in visualizations in the visualization pane
* Add in css stylings
* Refactor

### Step 1: Individual Visualization ###


### Step 2: Visualization by area ###
I decided to use LSOA and MSOA output areas (because LSOA was available in the police data!)
LSOAs correspond to areas that contain an average of 1500 people, and MSOAs are larger named areas that contain 6 of these LSOAs (LSOAs nicely nestle inside MSOAs)
You can find more info on these terms here: https://ocsi.uk/2019/03/18/lsoas-leps-and-lookups-a-beginners-guide-to-statistical-geographies/
These boundaries were designed by the office of national statistics during the census, and were made to compare trends in people across areas. The most important characteristic: They contain roughly the same number of families! Which means the number of crimes here can be used as a proxy for crime rate (no need to adjust for each region's population size), and gives us an indicator to how safe a region is.

We have our big dataset - made a script with R (I couldn't resist) to group it by year, month, LSOA, and crime type. Then linked each LSOA to its corresponding MSOA with a left join.

### CSS Stylings ###

### Deploy!###
So cool! we can host the app on a virtual machine on aws.
