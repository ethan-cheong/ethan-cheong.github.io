---
layout: post
title: Love Song Classification with R
date: 2020-12-22
categories: blog
tag: Projects
---
_You can find the datasets and code for this project here: [github.com/ethan-cheong/loveSongs](http://github.com/ethan-cheong/loveSongs)_

_This is a work in progress!_
## Motivation ##

Since finishing my project with the projects division, I've been looking into other applications of Natural Language Processing (NLP). Specifically, I've been exploring how NLP can be used to distinguish love songs from non-love songs, and have been trying to train a classifier that would be able to tell if a song is a love song from the lyrics.

At first glance, this seemed like quite an easy problem - after all, there should be certain 'trigger' words that indicate if a song is a love song. Some obvious words that come to mind include "love", "tears" and "cry". The thing is, we can quite easily think of examples of non-love songs that contain these words - for example, Starships by Nicki Minaj, which contains the phrase "I love to dance". On the other hand, some love songs like Iris by the Goo Goo Dolls don't contain obvious phrases like these. Especially tricky to classify are songs where the entire song is a _metaphor_, and doesn't explicitly mention love or relationships at all.

Clearly, the solution isn't as simple as just looking for trigger words - we'll try a few traditional classification algorithms as well as more cutting-edge NLP techniques. First of all, I came across the somewhat tricky problem of defining a love song - Wikipedia calls it "a song about romantic love, falling in love, heartbreak after a breakup, and the feelings that these experiences bring." Just to be clear, this means songs like Ed Sheeran's "Shape of You" are **not** love songs.

This distinction that songs about sleeping with someone are not considered love songs make things slightly difficult for our classifier, and I set the somewhat arbitrary goal that the classifier should be able to label tricky examples like "Shape of You" correctly.

I then came up with a rough plan for the process:
1. Collecting data
2. Cleaning and Visualization
3. Feature Engineering
4. Modelling and Prediction

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
I repeated this process for artists, and then wrote the scraped songs, artists and dates to a csv.
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
Next, I had to get lyrics for each song that I'd scraped. I ran into problems with the `geniusr` library, so I decided to use the `lyricsgenius` library from python for this next step.
{% highlight python %}
{% raw %}
import lyricsgenius
import pandas as pd
import re
import datetime

songData = pd.read_csv('.\\datasets\\songs_unlabelled.csv')

print(songData)

allSongData = pd.DataFrame()

api = lyricsgenius.Genius(# insert your genius api key here ,
    sleep_time = 0.01, verbose = False)

startTime = datetime.datetime.now()

for i in range(0, len(songData)):
  passed = str(datetime.datetime.now() - startTime)
  print(passed + " has passed.")
  rollingPct = int((i/len(songData))*100)
  print(str(rollingPct) + "% complete.")

  songTitle = songData.iloc[i]['song']
  songTitle = re.sub(" and ", " & ", songTitle)
  artistName = songData.iloc[i]['artist']
  artistName = re.sub(" and | \\+ | Featuring ", " & ", artistName) # to account for the difference between how billboard and spotify identify collaborations

  try:
    song = api.search_song(songTitle, artist = artistName)
    print("scraping for " + songTitle)
    songAlbum = song.album
    songLyrics = re.sub("\n", " ", song.lyrics) # remove newline breaks, we won't need them.
    songYear = song.year

  except AttributeError:  # if song not found
    print(songTitle + " not found!")
    songAlbum = "null"
    songLyrics = "null"
    songYear = "null"

  except TypeError:  # if connection timed out
    print("Connection timed out! Writing scraped songs to csv")
    break

	row = {
  	"Chart date": songData.iloc[i]['date'],
  	"Song Title": songData.iloc[i]['song'],
  	"Artist": songData.iloc[i]['artist'],
  	"Album": songAlbum,
  	"Lyrics": songLyrics,
  	"Release Date": songYear,
  }
  allSongData = allSongData.append(row, ignore_index=True)

allSongData.to_csv(".\\datasets\\songs_lyrics_unlabelled.csv", index = False)
{% endraw %}
{% endhighlight %}
After 10 hours of scraping, I got a dataset of ~5000 songs that looked something like this:

![image1](/assets/lovesongs/image1.png)

Notice that there are some null entries - these are songs that the scraper couldn't find lyrics or metadata for.

I now faced another problem - to train a supervised learning algorithm, I needed a _labelled_ dataset. I spent about 25 hours manually labelling each song, giving it a label of 1 for love song, and 0 for otherwise. Whilst doing this, I also took out any songs that weren't in english (mostly korean/latin songs) or songs that didn't have lyrics. I also removed any remixes or covers of songs that were already in the dataset.

## Cleaning and Visualization ##
### Cleaning with the tidyverse ###
Now that I had my labelled dataset, I had to clean it - especially the lyrical data, which I was eventually going to feed into my models. I started by importing the labelled dataset and relevant libraries.
{% highlight R %}
{% raw %}
library(tidyverse)
library(lubridate)
library(tidytext)
library(lexicon)

song.data <- read_csv("./datasets/songs_lyrics_labelled.csv",
                      col_names = c(
                        "album",
                        "chart.date",
                        "love.song",
                        "lyrics",
                        "release.date",
                        "song.title",
                        "artist"
                      ),
                      col_types = cols(
                        album = col_character(),
                        chart.date = col_date("%d/%m/%Y"),
                        love.song = col_factor(),
                        lyrics = col_character(),
                        release.date = col_date("%d/%m/%Y"),
                        song.title = col_character(),
                        artist = col_character()
                      ),
                      skip = 1)
{% endraw %}
{% endhighlight %}
First, I removed any brackets and square brackets and fixed the encoding issues with quotation marks.
{% highlight R %}
{% raw %}
song.data.clean <- song.data %>%
  mutate(lyrics = str_remove_all(lyrics, "\\[.+?\\]")) %>%
  mutate(lyrics = str_remove_all(lyrics, "\\(.+?\\)")) %>%
  mutate(lyrics = str_replace_all(lyrics, "â€™", "'")) %>%
  mutate(lyrics = str_to_lower(lyrics))
{% endraw %}
{% endhighlight %}
The code below expands any contractions (you'll have to manually inspect your dataset and find out what needs to be expanded) and removes punctuation.
{% highlight R %}
{% raw %}
FixContractions <- function(string) {
  string <- str_replace_all(string, "won't", "will not")
  string <- str_replace_all(string, "can't", "cannot")
  string <- str_replace_all(string, "n't", " not")
  string <- str_replace_all(string, "'ll", " will")
  string <- str_replace_all(string, "'re", " are")
  string <- str_replace_all(string, "'ve", " have")
  string <- str_replace_all(string, "'m", " am")
  string <- str_replace_all(string, "'d", " would")
  string <- str_replace_all(string, "n'", "ng")
  string <- str_replace_all(string, "'cause", "because")
  string <- str_replace_all(string, "it's", "it is")
  string <- str_replace_all(string, "'s", "")
  string <- str_replace_all(string, "'less", "unless")
  return(string)
}

song.data.clean <- song.data.clean %>%
  mutate(lyrics = lyrics %>%
           FixContractions() %>%
           str_replace_all("[^a-zA-Z0-9\\-' ]", " ") %>%
           str_replace_all("([a-zA-Z])\\1\\1",""))
{% endraw %}
{% endhighlight %}
Finally, I converted chart.dates to lubridate date formats for easier visualizations later, and wrote the dataset to a csv to use in subsequent scripts.
{% highlight R %}
{% raw %}
song.data.clean <- song.data.clean %>%
  mutate(chart.date = as_date(chart.date),
         release.date = as_date(release.date))

write_csv(song.data.clean, "datasets/song_data_clean.csv", col_names = TRUE)
{% endraw %}
{% endhighlight %}
### Exploratory Data Analysis ###

{% highlight R %}
{% raw %}

{% endraw %}
{% endhighlight %}
