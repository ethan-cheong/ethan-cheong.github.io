---
layout: post
title: Interactively Mapping Crime in the UK
date: 2021-01-16
categories: blog
tag: Python
permalink: /crime-map-uk/
---
I spent a good part of the first two weeks of 2021 building a dashboard that plots crime rates in the UK. It runs on Dash, a framework for making web apps in Python (built on flask), and is running on an Amazon Web Services EC2 instance.
You can try it out here: [http://13.212.193.155:8050/](http://13.212.193.155:8050/) (I haven't gotten around to registering a domain name for this, and will probably do so if I ever decide to register a domain for the blog.)

Originally, I had plans to do a "racism-detector" type app that would scrape twitter for racist tweets (maybe using some kind of model to identify if a tweet was racist) and then plot density of racist tweets across the UK. I unfortunately ran into issues with geo-tagging each tweet and had to scrap that, although that eventually became the motivation for this project. I was also partly inspired by [this](https://shiny.rstudio.com/gallery/crime-watch.html) example app in the documentation for Shiny - I thought it was interesting in concept (I think we have a strange obsession with crime, especially when we can actually see it happening near us) and worth expanding on.

Let's take a look at what our app can do. We first choose the visualization type, which determines whether we map yearly aggregate crime rate (by region) or monthly individual crimes. The app dynamically updates, changing the sort of options we can pick from depending on which visualization type we use.

### Aggregate Crimes
Say we choose aggregate crimes; we then have a slider to pick the year, a dropdown menu that lets us select what crimes we want to measure (there's also a convenient select-all option), and a boundary selector, which defaults to Local Authority District. A bit more information about these boundaries:
* Local Authority Districts (or just districts) are used for local governments. Each of London's boroughs is a district.
* Lower-layer Super Output Areas were regions marked out in the 2011 census, specifically designed for statistical analysis. They're specially drawn so that each LSOA consists of about 1500 people - which means with all else remaining constant, we expect each LSOA to have the same crime rate. LSOAs are in general much smaller than districts, and are useful if you want to compare crime density across neighbourhoods or streets.

You can also enter a UK postal code to zoom in on a particular address or region quickly; otherwise it shows the whole of England.

Our app then takes all these inputs and plots the natural logarithm of crime rate across each region (depending on which boundary level we've chosen) - the brighter the colour, the higher the crime rate. It updates whenever we make a change to our input settings. When we mouse over a region, we get a label with the following info:
* LAD/LSOA - the code for the region
* LADName/LSOAName - the actual name of the region
* Crimes per 1000 people / Number of crimes (actual) - Crime rate adjusted for population size for a district / the actual number of crimes in an LSOA
* Log crime rate / Number of crimes (log) - the natural log of the above statistic, used for plot colour consistency

If we select regions on the map using the lasso/box select (mouse over the top-right of the map to see these), we also get a plot of the crime rate/log crime rate for each region in the window in the bottom left. This bit is entirely dependent on what we plot in the map, which means it updates accordingly based on the date/crime types/region level we select.

### Individual Crimes
For individual crimes, we have all the other fields (except region level), and an additional option to brighten individual crime.

I made the postal code entry field with this mode in mind: entering a postal code lets us zoom into a particular street and see all the crime that's happened there over the selected month. If we leave the field blank, it defaults to a view of Central London (although you can still scroll around and zoom out to see the whole of the UK).

Individual crimes are shown as translucent dots, but if we want to see them more clearly we can check the Brighten Individual Crimes option. Crimes are color-coded based on type, so we can plot multiple types of crime at once.

Feel free to play around with the app - you can find some interesting things! It's worth noting that at the time of this post (January 2021) the crime data for December 2020 isn't available on the UK Police Department's website, so picking that date won't plot any map.

## Building the app
As someone who doesn't have much experience with software/web development, this was quite a struggle, but I learned a lot from working at the whole thing from start to finish.

Although I was originally planning on making a Shiny web app with R, I decided to use Python:
* I'd like to build more things in python, and could use the practice with the syntax - R is more of a specialist language for statisticians/economists
* The main reason I prefer R to Python for data science projects is the ease of use of the tidyverse packages, which lets you quickly string together functions with the pipe operator “%>%”. It'd be nice to be less reliant on the pipe, especially since Python is becoming the industry standard for data science
* I'd like the possibility to combine this with machine learning techniques in the future, for which Python has a better ecosystem for. (It took me quite a while to realize this and even longer to admit it)

I split up building the app into these steps, which took me about a week:
* Prepare datasets - the UK Police API gives us separate datasets for each month, we have to find a way to make this as compact as possible
* Build a skeleton of our app with html and dash core components
* Code in all the interactivity between them (for example, changing the visualization mode should change the options available)
* Implement visualizations for individual crimes
* Compile an aggregate dataset for aggregate visualizations, so we don't have to call all these steps in python in our app - grouped counts can be expensive!
* Implement visualizations for aggregate crimes
* Get the graph in the bottom left corner working
* CSS to make the app look better
* Deploy on an AWS EC2 instance (a virtual computer)

Admittedly the [code](https://github.com/ethan-cheong/CrimeMapUK) could use a lot of work - there's a lot of refactoring to be done, and there are definitely ways to reduce RAM usage on our virtual computer (we start by loading a 2 gigabyte dataset!) - but I was more interested in making a working prototype than optimizing its performance. One possibility I considered was putting the dataset into a database, and having the app do SQL calls so that data is only loaded into memory right before it's used to plot something.
