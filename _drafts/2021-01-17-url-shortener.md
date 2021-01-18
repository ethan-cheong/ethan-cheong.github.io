---
layout: post
title: Building a URL Shortener for LSESU Data Science Society
date: 2021-01-17
categories: blog
tag: Python
permalink: /url-shortener/
---

Chris (the President of LSESU's Data Science Society) posed me this problem recently: the society's link shortener had broke (it was previously hosted on another member's server), and we needed to make a new one that our marketing department could use (who may not necessarily have coding background).

This is probably the first project that I've done not out of interest, but instead out of necessity (there's no econs/math/machine learning involved!). So in a way, it's more of a pure software engineering type problem. Nonetheless, I thought it was a good opportunity to learn Django and get more experience with Amazon Web Services (AWS).

Here were the requirements we settled on:
1. The link shortener should take a URL such as facebook.com/dsatlse/g_7890dsa_i and generate a link like dsatlse/facebook. The link should be short enough to be easily posted on platforms like instagram.
2. The link it generates should be customizable
3. Any clicks on the link should be automatically redirected to our original URL.
4. Should be fast with minimal downtime (this happened right before one of the sessions last term)
5. If a user tries to input a URL that already has a link / tries to input a link that's already being used by a URL, it should give an error. It should also ask if they want to remove a URL from our database, and
6. There should be some kind of user interface that lets us see what URLs we've already entered, an entry box (with password authentication) and a delete box (also with password authentication).
7. The link shortener itself should be easily accessible. (No long URL/IP address!)

We may also be interested in tracking how many times a link is clicked. It's also a read-heavy service - we only need to make several thousand links at most, but a link will likely be clicked on several hundred times.

So we (the president and I) decided on the following structure:

1. A front end built with Django and hosted on an AWS EC2 instance (like my UK crime-map web app!). It should show us our database of short and long URLs. Also, there should be a field to add a new short and long URL to a PostgreSQL database, as well as a field to remove an entry from the database. It will also handle authentication and checking if a shorturl is already in use.
2. When the add/delete buttons are used, an AWS lambda function is called, adding/removing an entry from our database (hosted on AWS RDS). At the same time...
3. An AWS empty object with 'website redirect' metadata and the long url are created in/deleted from an S3 bucket by a separate lambda function. This will automatically handle the redirects.
4. Finally, we'll wrap everything up in cloudfront/electric beanstalk.

At the moment it doesn't seem like we can count the number of redirects with this method though.

### Making the web app with django
I chose Django for the front end because it has good support with AWS.

###
