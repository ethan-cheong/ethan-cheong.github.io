---
layout: post
title: Love Song Classification with R (Part 1)
date: 2020-12-18
categories: blog
tag: Natural Language Processing, Machine Learning, R
usemathjax: true
permalink: /love-song-classification-1/
---
_In this post, I explore methods for natural language processing and machine learning, and using them to classify songs.
 You can find the datasets and code for this project [here](http://github.com/ethan-cheong/loveSongs)._

_This is a long post! I go over each step of the project, as well as the theory behind the algorithms that I've applied._

## Motivation ##

Since finishing my project with the projects division, I've been looking into other applications of Natural Language Processing (NLP). Specifically, I've been exploring how NLP can be used to distinguish love songs from non-love songs, and have been trying to train a classifier that would be able to tell if a song is a love song from the lyrics.

At first glance, this seemed like quite an easy problem - after all, there should be certain 'trigger' words that indicate if a song is a love song. Some obvious words that come to mind include "love", "tears" and "cry". The thing is, we can quite easily think of examples of non-love songs that contain these words - for example, Starships by Nicki Minaj, which contains the phrase "I love to dance". On the other hand, some love songs like Iris by the Goo Goo Dolls don't contain obvious phrases like these. Especially tricky to classify are songs where the entire song is a _metaphor_, and doesn't explicitly mention love or relationships at all.

Clearly, the solution isn't as simple as just looking for trigger words - we'll try a few classification algorithms. First of all, I came across the somewhat tricky problem of defining a love song - Wikipedia calls it "a song about romantic love, falling in love, heartbreak after a breakup, and the feelings that these experiences bring." Just to be clear, this means songs like Ed Sheeran's "Shape of You" are **not** love songs.

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
Next, I had to get lyrics for each song that I'd scraped. I ran into problems with the `geniusr` package, so I decided to use the `lyricsgenius` package from python for this next step.
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
Finally, I converted chart.dates to `lubridate` date formats for easier visualizations later, and wrote the dataset to a csv to use in subsequent scripts.
{% highlight R %}
{% raw %}
song.data.clean <- song.data.clean %>%
  mutate(chart.date = as_date(chart.date),
         release.date = as_date(release.date))

write_csv(song.data.clean, "datasets/song_data_clean.csv", col_names = TRUE)
{% endraw %}
{% endhighlight %}
### Exploratory Data Analysis ###
At this stage, I tried coming up with questions to get a better understanding of the dataset. Firstly, what proportion of songs that chart each year are love songs?
{% highlight R %}
{% raw %}
install.packages("RColorBrewer")

song.data.explore %>%
  count(year = year(chart.date), love.song) %>%
  mutate(love.song = if_else(love.song==0, "No", "Yes")) %>%
  ggplot(aes(year, n)) +
  geom_bar(stat = "identity", position = "fill", aes(fill = love.song)) +
  labs(x = "Year", y = "Proportion", fill = "Love Song", title = "Yearly proportion of top-100 songs by song category") +
  scale_x_continuous(breaks = c(2010:2020)) +
  scale_fill_brewer(palette = "PuRd") +
  theme_dark()
{% endraw %}
{% endhighlight %}

![graph1](/assets/lovesongs/graph1.png)

We see a slight decrease in proportion of love songs over time - although it's difficult to see if it's a significant trend.

Next, what months are best to release songs - that is, what month were top 100 songs most likely to be released in?
{% highlight R %}
{% raw %}
song.data.explore %>%
  filter(release.date >= date("01-01-2010")) %>%
  count(month = factor(month(release.date))) %>%
  ggplot(aes(month, n, fill=month)) +
  geom_bar(stat = "identity") +
  scale_x_discrete("Month", labels = month(c(1:12), label = TRUE)) +
  labs(y = "Count", title = "Number of top 100 hits per month across 10 years") +
  scale_fill_brewer(palette = "Set3") +
  theme_dark() +
  theme(legend.position = "none")
{% endraw %}
{% endhighlight %}

![graph2](/assets/lovesongs/graph2.png)

We see that October has the most number of top 100 hits - however, this doesn't necessarily mean October is the best month to release songs! It's also possible that prolific artists like to release their albums in October.

Next, I decided to perform visualizations with the words themselves. Using the `tidytext` package, we can convert the dataset to a 'tidy' format - that is, with each individual word occupying a row.
{% highlight R %}
{% raw %}
library(tidytext)  # package to convert words to 'tidy' format
library(lexicon)   # lexicons used for filtering words

song.data.tidy <- song.data.explore %>%
  unnest_tokens(word, lyrics) %>%
  filter(word %in% grady_augmented | word %in% profanity_zac_anger) %>% # Keep only words in the lexicons
  distinct()
{% endraw %}
{% endhighlight %}
First, I found the most popular words in each class of song.
{% highlight R %}
{% raw %}
popular.words <- song.data.tidy %>%
  group_by(love.song) %>%
  count(word, love.song, sort=TRUE) %>%
  mutate(word = reorder_within(word, n, love.song)) %>%
  filter(rank(desc(n)) <= 45) %>%  # We only take the top 45 words here
  mutate(love.song = if_else(love.song == 0, "Other Songs", "Love Songs"))

popular.words %>%
  ggplot(aes(word, n, fill = love.song)) +
  geom_col(show.legend = FALSE) +
  facet_wrap(~love.song, scales = "free") +
  coord_flip() +
  scale_x_reordered() +
  scale_y_continuous(expand = c(0,0)) +
  labs(y = "Count",
       title = "What were the most popular words in love songs and other songs?") +
  scale_fill_brewer(palette = "PuRd") +
  theme_dark()
{% endraw %}
{% endhighlight %}

![graph3](/assets/lovesongs/graph3.png)

There are two main problems with this - firstly, the presence of filler words such as "I", "the", "to", and "and", which are present across both classes of song and won't help us much with classification. Secondly, songs of particular genres (rap in particular) tend to contain many more words than others, resulting in them being overrepresented if we use a measure like the frequency of a word. To solve the second problem, we can instead take the density of a word - that is, the frequency of a word in a song divided by the total number of lyrics in that song - and average that across all songs.
{% highlight R %}
{% raw %}
word.density <- song.data.tidy %>%
  count(word, love.song) %>%
  group_by(love.song) %>%
  filter(rank(desc(n)) <= 100) %>%  # take the 100 most frequent words
  left_join(song.data.clean) %>%
  mutate(total.words = str_count(lyrics, boundary("word"))) %>%
  mutate(density = str_count(lyrics, word)/total.words) %>%
  select(love.song, word, density) %>%
  group_by(word, love.song) %>%
  filter(n()>2) %>%
  summarise(mean.density = mean(density))

word.density %>%
  arrange(desc(mean.density)) %>%
  group_by(love.song) %>%
  filter(rank(desc(mean.density)) <= 45) %>%
  mutate(word = reorder_within(word, mean.density, love.song)) %>%
  mutate(love.song = if_else(love.song == 0, "Other Songs", "Love Songs")) %>%
  ggplot(aes(word, mean.density, fill = love.song)) +
  geom_col(show.legend = FALSE) +
  facet_wrap(~love.song, scales = "free") +
  coord_flip() +
  scale_x_reordered() +
  scale_y_continuous(expand = c(0,0)) +
  labs(y = "Average density of each word across songs",
       title = "Which words had the highest average density in songs across song types?") +
  scale_fill_brewer(palette = "PuBu") +
  theme_dark()
{% endraw %}
{% endhighlight %}

![graph4](/assets/lovesongs/graph4.png)

Using these two visualizations, we can solve the first problem by coming up with a list of stop words - words that we should subsequently filter from our dataset before using it to train our models. This will help reduce the dimensionality of our transformed dataset and subsequently the cost of training each model.
{% highlight R %}
{% raw %}
stop.words <- c(
  'a','b','c','d','e','f','g','h','i','j','k','l','m','n','o','p','q','r','s',
  't','u','v','w','x','y','z','in','the','it','you','on','no','me','at','to','is',
  'am','and','go','or','do','be','not','my','as','we','all','so','ai','that',
  'up','oh','now','like','your','one','of','out','yeah','for','got','can','if',
  'get','are','em','but','know','here','will','every','ever','always',
  'same','done','with','just','have','this','when','because'
)
{% endraw %}
{% endhighlight %}
After removing these stop words, I plotted average density across songs for each word one more time:

{% highlight R %}
{% raw %}
song.data.tidy.2 <- song.data.tidy %>%
	filter(!word %in% stop.words)

word.density.2 <- song.data.tidy.2 %>%
	count(word, love.song) %>%
	group_by(love.song) %>%
	filter(rank(desc(n)) <= 100) %>%
	left_join(song.data.clean) %>%
	mutate(total.words = str_count(lyrics, boundary("word"))) %>%
	mutate(density = str_count(lyrics, word)/total.words) %>%
	select(love.song, word, density) %>%
	group_by(word, love.song) %>%
	filter(n()>2) %>%
	summarise(mean.density = mean(density))

word.density.2 %>%
	arrange(desc(mean.density)) %>%
	group_by(love.song) %>%
	filter(rank(desc(mean.density)) <= 45) %>%
	mutate(word = reorder_within(word, mean.density, love.song)) %>%
	mutate(love.song = if_else(love.song == 0, "Other Songs", "Love Songs")) %>%
	ggplot(aes(word, mean.density, fill = love.song)) +
	geom_col(show.legend = FALSE) +
	facet_wrap(~love.song, scales = "free") +
	coord_flip() +
	scale_x_reordered() +
	scale_y_continuous(expand = c(0,0)) +
	labs(y = "Average density of each word across songs",
	     title = "Which words had the highest average density in songs across song types?") +
	scale_fill_brewer(palette = "RdPu") +
	theme_dark()
{% endraw %}
{% endhighlight %}

![graph5](/assets/lovesongs/graph5.png)

We can now see clear differences between the two song types! Even just considering the words with the highest average density in each category, love songs seem to use more first and second person pronouns ("us", "our"), whilst non-love songs contain a mix of first, second and third person pronouns ("he", "us", "her", "his"), with third person pronouns dominating.

We also see the phrase "love" turn up with higher relative density in love songs, which is to be expected. Words in our top 45 list that are unique to love songs include "heart", "would" (perhaps the singer is making promises?) and "fall", whilst those unique to other songs seem to be mostly swear words. Of course, the presence of these words alone isn't enough to determine whether a song is a love song, which is where the next few steps come in.
## Feature Engineering ##
Our data currently has a long string of lyrics for each song, and we have to convert it to a form that we can feed into our algorithms. Here, we explore several more traditional methods for coming up with these features, namely TF-IDF and n-grams.
### TF-IDF ###
Our goal is to generate a matrix where each row represents a song and each column represents a possible word in the song. We say _possible_ word because we need some measure of whether a song does not contain a word, in which case the entry for that word column will be 0. The number of rows should then be equal to the total number of words, without repeats, of every single song we have in our dataset - we call this huge list of words our **corpus**.

Now that we have our **document-term matrix** (or song-word matrix in this case), assuming a song contains a word, what should we put in that entry? We could simply put a 1 to indicate the song has the word, but as we've stated earlier, the presence of a word isn't enough to determine whether a song is a love song - we should have some measure for the _significance_ for the word in the context of that song.

Term Frequency - Inverse Document Frequency (TF-IDF) gives us a measure of this significance by taking how often a word appears and dividing it by the number of songs the word appears in. The intuition is that a term that appears many times in one document but few times in other documents will have a high significance to that specific song.

First of all, let's make sure our dataset contains only words in our lexicons.
{% highlight R %}
{% raw %}
# Function to remove words that aren't in the lexicons from lyrics
FilterLexicon <- function(string) {
  string.list <- str_split(string, boundary("word"))
  for (i in seq_along(string.list)){
    string.list[[i]] <- if_else(string.list[[i]] %in% grady_augmented | string.list[[i]] %in% profanity_zac_anger, string.list[[i]], "")
  }
  return(str_c(unlist(string.list), collapse = " "))
}
# Apply to song.data.clean
lyrics <- song.data.clean$lyrics
song.data.clean.2 <- song.data.clean %>%
  mutate(lyrics = map_chr(lyrics, FilterLexicon))
{% endraw %}
{% endhighlight %}
Now, we can use the `tm` package to get our features.
{% highlight R %}
{% raw %}
library(tm)

# Create the corpus
corpus <- VCorpus(VectorSource(song.data.clean.2$lyrics))
# document-term matrix, applying TF-IDF, stemming, and using stop words defined earlier
tfidf.dtm <- DocumentTermMatrix(corpus, control = list(weighting= weightTfIdf,
                                                       stopwords = stop.words,
                                                       removeNumbers = TRUE,
                                                       removePunctuation = TRUE,
                                                       stemming=TRUE))
# Get a data.frame for modelling
tfidf.df <- data.frame(as.matrix(tfidf.dtm),
                       stringsAsFactors = FALSE)
# Add labels
tfidf.df$song.label <- song.data.clean.2$love.song
{% endraw %}
{% endhighlight %}
Using `view(tfidf.df)`, we can see that we have a `data.frame` with 4453 documents and 12795 terms which looks something like this:

![image2](/assets/lovesongs/image2.png)

Notice how _sparse_ this dataframe is - many of the entries are 0, which correspond to songs that don't contain the word encoded by that particular column.

We can also get a similar dataframe `tf.idf` that uses term frequency instead of TF-IDF by setting the `weighting` parameter of `DocumentTermMatrix()` to `weightTf`.

![image3](/assets/lovesongs/image3.png)

Notice that the entries of this dataframe are integers, which indicate how many times a particular term appears in a song.
### N-grams ###
Some words take on a different meaning when they're paired together with other words - a simple example would be something like "lost" and "get lost", which clearly have different connotations. Since combinations of words have the potential to carry meaning that might be lost when we split words up, we can instead split up our words into combinations of n different words - called an n-gram. We can then follow the same procedure as before, but instead of each term occupying a column we have an n-gram.

We implement bigrams below:
{% highlight R %}
{% raw %}
# Function to tokenize bigrams
BigramTokenizer <- function(x){
  unlist(lapply(ngrams(words(x), 2), paste, collapse = " "), use.names = FALSE)
}

# Make bigrams document term matrix
bigrams.dtm <- DocumentTermMatrix(corpus, control = list(tokenize = BigramTokenizer))
{% endraw %}
{% endhighlight %}
Inspecting `bigrams.dtm`, we find that we have 328316 unique bigrams! To reduce the dimensionality of our matrix so that our models can fit more easily, we can take out bigrams using `removeSparseTerms()`
{% highlight R %}
{% raw %}
# Take out bigrams that are only present in 1% of the songs.
bigrams.dtm <- removeSparseTerms(bigrams.dtm, 0.99)
{% endraw %}
{% endhighlight %}
Now, we have 2808 unique bigrams, which is much better.
{% highlight R %}
{% raw %}
# Get df
bigrams.df <- data.frame(as.matrix(bigrams.dtm),
                         stringsAsFactors = FALSE)
# Add labels
bigrams.df$song.label <- song.data.clean$love.song
{% endraw %}
{% endhighlight %}
In a future post, we'll try more modern methods of creating word embeddings using frameworks like ELMo - for now, let's try modelling using the features we have.

## Modelling ##
We'll use `mlr3` to implement our machine learning algorithms - this is the newest version of the `mlr` package, a very popular tool for running these algorithms in R. I particularly like how this package integrates the whole modelling process into a pipeline, letting you quickly try multiple algorithms at once.

As an example, let's try implementing a random forest on our `tfidf.df`:
{% highlight R %}
{% raw %}
#Prepare classification task
tfidf.task <- TaskClassif$new(id = "tfidf",
                              backend = tfidf.df,
                              target = 'song.label')

# set a random forest learner
learner.rf <- lrn("classif.ranger")

# train-test split of 80:20
tfidf.train <- sample(tfidf.task$nrow, 0.8*tfidf.task$nrow)
tfidf.test <- setdiff(seq_len(tfidf.task$nrow), tfidf.train)

# train the model
learner.rf$train(tfidf.task, row_ids = tfidf.train)
prediction <- learner.rf$predict(tfidf.task, row_ids = tfidf.test)

# Get test accuracy
measure = msr("classif.acc")
prediction$score(measure)
{% endraw %}
{% endhighlight %}
Running this block of code once gave me a test accuracy of 78.5%.

We can also use `benchmark()` to compare multiple algorithms at the same time. We'll try k-nearest neighbours, a naive bayes classifier, random forests, support vector machines and XGBoost.
{% highlight R %}
{% raw %}
# vector of algorithms to try -
learners <- c("classif.kknn", "classif.naive_bayes", "classif.ranger", "classif.svm", "classif.xgboost")
learners <- lapply(learners, lrn, predict_sets=c("train", "test"))

#use 3-fold cross validation for resampling
resamplings <- rsmp("cv", folds=3)
design <- benchmark_grid(tfidf.task, learners, resamplings)

# Evaluate benchmark
bmr <- benchmark(design)

# Choose performance measure
measures <- list(
  msr("classif.ce", id = "ce_train", predict_sets = "train"),
  msr("classif.ce", id = "ce_test")
)

bmr$aggregate(measures)
{% endraw %}
{% endhighlight %}

![image4](/assets/lovesongs/image4.png)

We can see that Random Forests give us the lowest test error, followed by XGboost - it's worth looking deeper into these algorithms to understand what they're doing. (This will help us with parameter tuning - also, it's always nice to know more things!)
### Gradient Tree Boosting and XGBoost ###
This next section gets slightly more involved; there's some scary mathematical notation, but the fundamental principles are quite intuitive!
Most of the information here on boosting was taken from [_An Introduction to Statistical Learning with Applications in R_]({https://d1wqtxts1xzle7.cloudfront.net/60707896/An_Introduction_to_Statistical_Learning_with_Applications_in_R-Springer_201320190925-63943-2cqzhk.pdf?1569478040=&response-content-disposition=inline%3B+filename%3DAn_Introduction_to_Statistical_Learning.pdf&Expires=1608995359&Signature=gwbUT4vRsvApjxQvKcsL~Og~58zWn7-7eGNSJ~JNVItVp5Cpjea4TmnsuED0J7TsqPPBQZM7ETsSgm~0L4UYPcuJMOb~8qeph3inbWjbnb9hOW2CqCAaqEnUfw7m0BPxGIB2RCdl0FS3YEccDGaFdLlC1jb5V6lnplq4XJQqNX3BdYRJ0kf9rjMbWSFTCvd20ycwA007rZWkemtYstqCbu6f5spIXCe24DBlwzxPFLkb2QwGCDaHwZ5RP9Z-mrB3af-UafktNxYtB8AdhpAvY68o5h-A~Tr08roXWgomxKTlFJSQwKmjdhI5cRnCTO9ZzJlRcxL3H6fr-cWJajg4Cg__&Key-Pair-Id=APKAJLOHF5GGSLRBV4ZA}) and [_The Elements of Statistical Learning_]({https://web.stanford.edu/~hastie/ElemStatLearn/}) - the latter is widely regarded as the gold standard reference for fundamental machine learning, and the former is a less in-depth version by the same authors, intended for beginners. I'd read ISLR before, but found the explanation in the boosting section to be slightly lacking, and referred to ESL to learn more.

Gradient tree boosting works by acting on _decision trees_; you can find more about them and how they're trained from the sources I've linked above. For now, it's sufficient to know that decision trees are considered _weak classifiers_, i.e. they perform only slightly better than random guessing. Fortunately, boosting can improve their performance, and does so by involves combining many weak classifiers to make a more robust classifier. The first instance of this was AdaBoost, which involves sequentially training new classifiers on the mistakes of old ones.
#### AdaBoost as an Example of a Boosting Model ####
Here's how AdaBoost works: each classifier trains on a random subset of the training data. After training one classifier, AdaBoost takes the training observations it misclassified and increases the probability they are selected for the next classifier, so the next classifier can try training on them.

After training all our "committee" of classifiers, we take a weighted "vote" of the entire committee, with the classifiers with lower error rate having a bigger say in the vote, and classifiers with error rate greater than 50% (worse than random chance!) having a _negative vote_ - in other words, whatever that classifier says, we do the opposite.
#### Similarities between Boosting and Additive Models ####
An additive model is a model that outputs an _additive expansion_, which are simply linear combinations with coefficients $$\beta_m$$ of a _basis function_ $$b(x; \gamma_m)$$. The generic form of an additive expansion looks something like this:

<center>
$$ f(x) = \sum_{m=1}^M\beta_mb(x;\gamma_m)$$
</center>

The basis function $$b(x;\gamma)\in\mathbb{R})$$ can be any function that takes an input $$x$$ and is parameterized by a some parameters $$\gamma$$. Say the basis function is a linear function $$b(x;\gamma) = \gamma_0 + \gamma_1x$$; then, its additive expansion takes the form:

<center>
$$ f(x) = \sum_{m=1}^M\beta_mb(x;\gamma_m) = \beta_1(\gamma_{10}+\gamma_{11}x) + \beta_2(\gamma_{20}+\gamma_{21}x) + \ldots + \beta_M(\gamma_{M0}+\gamma_{M1}x)$$
</center>

Fitting an additive model involves minimizing a _loss function_ $$L(y, f(x))$$. The function can be something like weighted least squares, where we have the actual value of the observation $$y$$ and our prediction $$f(x)$$ - the loss function then captures how "off" the prediction was from the actual observation. Fitting the model can then be expressed as an optimization problem:

<center>
$$ min_{(\beta_m,\gamma_m)^M_1} \sum^N_{i=1}L(y_i,\sum^M_{m=1}\beta_mb(x_i;\gamma_m))$$
</center>

This looks pretty scary, particularly due to the double summation, but it's mostly due to the notation used - the logic is quite intuitive! All this is saying is that we find the loss of the predictor (which is expressed as an additive expansion) for each of $$N$$ training observations, sum those losses, and then find the set of parameters $$\beta_1, \beta_2 \ldots \beta_M$$ and $$\gamma_1, \gamma_2 \ldots \gamma_M$$ of the additive expansion that minimizes this sum.

How is this relevant? Well, the authors of ESL found that fitting a boosting model was fundamentally the same as fitting an additive model.
If we take the basis formula to be a weak classifier, the additive expansion is exactly the weighted vote of our committee of weak classifiers, with the parameters $$\beta_i$$ representing the weights that classifier $$i$$ holds in the final vote, and $$\gamma_i$$ corresponding to the set of parameters of $$i$$. To fit our committee to training data, we need to minimize the error of our weighted vote - in other words, we need to minimize a loss function, which is exactly the problem we talked about above!

We now know that fitting a boosting model can be representing as an optimization problem, but how do we solve it? It turns out to be very difficult to optimize all our parameters at once, so we have to use an approximation.
#### Fitting Additive Models ####
Remember: we are trying to solve the problem:
<center>
$$ min_{(\beta_m,\gamma_m)^M_1} \sum^N_{i=1}L(y_i,\sum^M_{m=1}\beta_mb(x_i;\gamma_m))$$
</center> So that we can get the parameters for a predictive function in the form
<center>
$$ f(x) = \sum_{m=1}^M\beta_mb(x;\gamma_m)$$
</center>
The algorithm we use is called _Forward Stagewise Additive Modelling_, and works as follows:
1. Set $$f_0(x)=0$$
2. For $$m=1,2\ldots M$$:
	*  Compute the coefficients $$(\beta_m,\gamma_m)$$ that minimize <center> $$\sum^N_{i=1}L(y_i,f_{m-1}(x_i) + \beta b(x_i;\gamma))$$ </center>
		* I.e. find the coefficients that minimize the loss of our existing function plus an additional basis function with a coefficient $$\beta$$, across all of our $$N$$ training observations.
	*  Set <center> $$f_m(x) = f_{m-1}(x) + \beta_mb(x;\gamma_m)$$ </center>
		* I.e. _update_ our function by adding it to the weighted basis function using the coefficients we've just found above.

Instead of the computationally expensive task of calculating all the $$\beta_m$$ and $$\gamma_m$$ at once, the algorithm calculates each $$\beta$$ incrementally, combines it with a basis function and adds it to our existing function - it then uses the loss of this composite function to calculate the subsequent $$\beta$$ values.  

#### Gradient Tree Boosting ####
The problem with the algorithm above lies in step 2: although we've reduced the number of terms we have to find in our optimization problem, it's still extremely difficult unless we have a very simple loss function.

The problem is made somewhat easier if we use trees as the basis function. A tree with $$J$$ _terminal nodes_ can be expressed formally as:

<center>$$T(x;\Theta) = \sum^J_{j=1}\gamma_jI(x\in R_j)$$</center>

Here's what all these greek letters mean:
* $$R_1, R_2 \ldots R_J$$ representing the $$J$$ spaces that the tree partitions each combination of predictor values into.
* $$\gamma_j$$ is the output of the tree for a particular node $$j$$.
* The function $$I()$$ returns 1 if the statement within is true, and 0 otherwise. Thus, if an observation $$x$$ belongs to a region $$R_j$$, then its predicted value $$f(x)$$ is given as $$\gamma_j$$.
* $$\Theta$$ are the parameters of the tree: it represents $$\{R_j,\gamma_j\}_1^J$$, which are the regions $$R$$ and outputs $$\gamma$$ of _each_ of the $$J$$ terminal nodes.

Substituting this into the FSAM algorithm above, we get the following optimization step for fitting a boosted tree:
<center> $$\Theta_m = min_\Theta\sum^N_{i=1}L(y_i, f_{m-1}(x_i)+T(x_i;\Theta))$$ </center>

This is still a pretty difficult optimization problem, but we can solve it by thinking about _gradient descent_. For those unfamiliar with gradient descent, I'd reccommend taking a look at Andrew Ng's Coursera on Machine Learning. The algorithm goes something like this:

 Say we are stuck on a mountain range, and want to get back to civilisation at the bottom of the mountains - but there's a heavy fog, so we can't see anything around us. We can use our feet to feel around us for the _steepness_ of the slope, finding which direction points upwards, and then proceed $$n$$ steps in the _opposite_ direction. After taking these steps, we feel around for the steepness again, and then repeat the process iteratively until we reach the base of the mountain.

Gradient descent lets us minimize a loss function in the same way. We initialize random parameters, then calculate the gradient ('steepness') of the function at that point by differentiation. We then adjust the parameters in the opposite direction of the steepest gradient, and do this iteratively until we reach a minimum loss. This is particularly useful when we have to optimize many parameters, but have a _differentiable_ function to optimize.

Gradient boosting works analagously,  but instead of optimizing in _parameter space_, we are optimizing in _function space_ - we want to find the optimal tree to add to our output function such that total loss is reduced. We have a _gradient vector_ $$g_m$$, whose components are:

<center> $$ g_{im} = [ \frac{\partial L(y_i, f(x_i))}{\partial f(x_i)} ]_{f(x_i)=f_{m-1}(x_i)} $$</center>

We update loss function in the _opposite direction_ of gradient. It's very easy to calculate gradient compared to our optimization problem earlier - so we can now simply fit a tree to our negative gradient instead by minimizing its squared error. Our optimization problem then reduces to:

<center> $$\Theta_m = min_\Theta\sum^N_{i=1}(-g_{im}-T(x_i;\Theta))^2 $$ </center>

which is computationally much more simple.

#### Regularization of Boosted Trees ####
When training machine learning algorithms, people often forget about the _regularization term_. This is a measure of the _complexity_ of the model. When training the model, it is penalized for being overly complex, which aims to prevent our model from overfitting the training data.

We can do this by minimizing an _objective function_ in place of our loss function. The objective function takes the form:

<center> $$ obj(\theta) = L(\theta) + \Omega(\theta) $$ </center>

where $$\theta$$ are parameters to be optimized, and $$\Omega(\theta)$$ is a regularization term. We have to minimize the objective function by minimizing both the loss term (representing how badly the model fits the data) and the regularization term (representing how complex the model is). In theory, by minimizing the objective function, we then produce a model that both fits well and is not needlessly complicated.

#### XGBoost ####
XGBoost is an alternative solution to the tree boosting optimization problem above. Recall we had to solve the following:

 <center> $$\Theta_m = min_\Theta\sum^N_{i=1}L(y_i, f_{m-1}(x_i)+T(x_i;\Theta))$$ </center>

and used gradient boosting, which is a natural extension of gradient descent to functions instead of parameters.
#### XGBoost Parameter Optimization ####

### Random Forests ###
Random forests involve multiple random decision trees, with each tree being grown on a random sample of the original data, and with a random subset of features being selected at each node to generate the best split.



Although parameter tuning doesn't play as big a role in training a Random Forest, we can still do some tuning below:
{% highlight R %}
{% raw %}
library(mlr3tuning)
library(paradox)

# Set parameters to tune
tune_ps <- ParamSet$new(list(
  ParamInt$new("max.depth", lower=1, upper = 32),
  ParamInt$new("min.node.size", lower=1, upper=1000)
))

learner.rf <- lrn("classif.rpart")

# Select a performance measure and a resampling strategy
measure <- msr("classif.ce")
resamplings <- rsmp("cv", folds=3)

# Select a budget - We will terminate when tuning does not improve
evals <- trm("stagnation")

# Select a tuner
tuner <- tnr("grid_search")

rf.tuned.model <- AutoTuner$new(
  learner = learner.rf,
  resampling = resamplings,
  measure = measure,
  search_space = tune_ps,
  terminator = evals,
  tuner = tuner
)

# Compare tuned learner against default value learner
grid <-  benchmark_grid(
  task = tfidf.task,
  learner = list(rf.tuned.model, lrn("classif.rpart")),
  resampling = rsmp("cv", folds = 3)
)

bmr <- benchmark(grid)
bmr$aggregate(msrs(c("classif.ce", "time_train")))
{% endraw %}
{% endhighlight %}

![image5](/assets/lovesongs/image5.png)

However, we see only marginally better performance from the untuned model (which had classification error of 0.22665).

In addition to the tf-idf feature set, we can also look at how well the untuned algorithms perform on our tf and bigrams feature sets:

![image6](/assets/lovesongs/image6.png)

![image7](/assets/lovesongs/image7.png)

**Test Error by Classification Algorithm and Feature Selection Method**

| Method | Term Frequency | TF-IDF | Bigrams |
|--------|----------------|--------|---------|
| **Naive Bayes** | 0.59951 | 0.36555 | 0.43707 |
| **Random Forest** | 0.25501 | 0.22665 | 0.22829 |
| **XGBoost** | 0.30545 | 0.31763 | 0.32089 |

Here, a combination of TF-IDF and Random Forests seem to work the best, although without fine-tuning each combination we won't know the optimal pair for sure.

## Conclusion ##
We've seen how to scrape for songs and lyrics, generate features and build models in R. In the next part, we'll take at look at more sophisticated techniques for feature engineering and modelling using Neural Networks.
