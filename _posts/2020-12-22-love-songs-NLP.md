---
layout: post
title: Love Song Classification with R
date: 2020-12-22
categories: blog
tag: Projects
---
_You can find the datasets and code for this project here: [github.com/ethan-cheong/loveSongs](http://github.com/ethan-cheong/loveSongs)_

## Motivation ##

Since finishing my project with the projects division, I've been looking into other applications of Natural Language Processing (NLP). Specifically, I've been exploring how NLP can be used to distinguish love songs from non-love songs, and have been trying to train a classifier that would be able to tell if a song is a love song from the lyrics.

At first glance, this seemed like quite an easy problem - after all, there should be certain 'trigger' words that indicate if a song is a love song. Some obvious words that come to mind include "love", "tears" and "cry". The thing is, we can quite easily think of examples of non-love songs that contain these words - for example, Starships by Nicki Minaj, which contains the phrase "I love to dance". Other love songs, like Iris by the Goo Goo Dolls, don't contain explicit phrases like these, but are love songs nonetheless.

Clearly, the solution isn't as simple as just looking for trigger words - we'll try a few classification algorithms which we'll go over later. Firstly, I came across the somewhat tricky problem of defining a love song - Wikipedia calls it "a song about romantic love, falling in love, heartbreak after a breakup, and the feelings that these experiences bring." Just to be clear, this means songs like Ed Sheeran's "Shape of You" are **not** love songs.

This distinction that songs about sleeping with someone are not considered love songs make things slightly difficult for our classifier, and I set the somewhat arbitrary goal that the classifier should be able to label tricky examples like "Shape of You" correctly. 

I then came up with a rough plan for the process:
1. Collecting data
2. Cleaning and Visualization
3. Feature Engineering
4. Modelling

## Collecting Data ##
To start with, I needed a substantial list of pop songs - and because love songs transcend genres, I couldn't use user-made playlists from spotify. (User-made "love" playlists would probably tend towards a particular genre anyway.)

### Web Scraping with rvest ###
Using the `rvest` package, I first tried scraping the billboard top 100 songs and artists for the first 3 months of 2010.
{% highlight r %}
{% raw %}
library(rvest)
library(tidyverse)

# functions to scrape songs and artists
ScrapeSongs <- function(date){
	url <- str_c("https://www.billboard.com/charts/hot-100/", date)
	songs <- read_html(url) %>%
		html_nodes(".chart-element__information__song") %>% # we can inspect element in our web browser to find the html corresponding to the text we need
		html_text()
}
ScrapeArtists <- function(date){
	url <- str_c("https://www.billboard.com/charts/hot-100/", date)
	songs <- read_html(url) %>%
		html_nodes(".chart-element__information__artist") %>% 
		html_text()
}

# Vector of dates we need
date.list <- seq(as.Date("2010-01-01", as.Date("2010-04-01", by="weeks")

output.songs <- vector("list", length = length("date.list"))
for (i in seq_along(date.list)) {
	output.songs[[i]] <- ScrapeSongs(date.list[i])
}

output.artists <- vector("list", length = length("date.list"))
for (i in seq_along(date.list)) {
	output.artists[[i]] <- ScrapeArtists(date.list[i])
}

song.tibble <- tibble(
	song=unlist(output.songs),
	artist=unlist(output.artists)
)
{% endraw %}
{% endhighlight %}
With a bit of trial and error, I found that `rvest` could only scrape about 15 pages at a time before throwing a HTTP error. The loop below accounts for this by freezing execution of R commands after scraping 15 pages, and scrapes the pages of every week from 2010 to 2020.
{% highlight R %}
{% raw %}
# Vector of dates from 2010 to 2020
date.list.10 <- seq(as.Date("2010-01-01"), as.Date("2020-11-23", by="weeks")

output.songs.10 <- vector("list", length = length(date.list.10))
count <- 0 # Count how many pages we have scraped consecutively
for (i in seq_along(date.list.10)){
	if (count <= 14){
		output.songs.10[[i]] <- ScrapeSongs(date.list.10[i])
		count <- count + 1
	} else {
		Sys.sleep(60) # Suspends execution of R commands for a minute
		output.songs.10[[i]] <- ScrapeSongs(date.list.10[i])
		count <- 0 # Reset count after 15 iterations
	}
}
{% endraw %}
{% endhighlight %}
I repeated this process for artists, and then made a tibble with the scraped songs, artists and dates.
{% highlight R %}
{% raw %}
song.tibble <- tibble(
	song = unlist(output.songs.10), # collapse nested list of songs
	artist = unlist(output.artists.10),
	date = rep(date.list.10, each = 100) # Assign dates of the scraped pages to each song
) %>% distinct(across(c(song,artist)), .keep_all = TRUE) # keep only first instance of each song-artist pair

# We have 5174 unique songs. Check for any duplicates
song.tibble %>%
	count(song, artist) %>%
	arrange(desc(n))

# Write data to csv for next step
write_csv(song.tibble, "./datasets/songs_unlabelled.csv", col_names = TRUE)
{% endraw %}
{% endhighlight %}
### Lyric Scraping ###
Next, I had to get lyrics for each song that I'd scraped. I ran into problems with the `geniusr` library, so I decided to use python and `lyricsgenius` for this next step.
{% highlight python %}
{% raw %}
import lyricsgenius
import pandas as pd
import re
import datetime
{% endraw %}
{% endhighlight %}
