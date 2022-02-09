---
title: 'Dictionary and bag-of-words approaches in R'
author: "A. Paxton (*University of Connecticut*)"
output: 
  html_document:
    keep_md: yes
---

A foundational approach to analyzing language corpora is to count the number and
types of individual words that occur in the corpus. These are generally known as
*bag-of-words approaches*, since they don't care about syntax or word order.
*Dictionary-based approaches* are related to bag-of-words approaches; dictionary
approaches categorize words according to pre-defined lists of words, again
without regard to word order. Perhaps the most popular dictionary approach is
Linguistic Inquiry and Word Count (LIWC; Pennebaker, Francis, & Booth, 2001), a
piece of commercial software with various grammatical and psychological
dictionary lists.

In this tutorial, we will use example text from the `janeaustenr` library: It
includes 6 books from Jane Austen, with one line of the printed text per line in
the dataframe. To process the dataset and implement the bag-of-words approach,
we will rely heavily on the `tidytext` package (which plays nicely with the
`tidyverse` ecosystem); to implement the dictionary-based approach, we will use
both `tidytext` and various packages designed to work with the `quanteda`
package ecosystem.

My thanks go to the documentation for all of the packages used here (especially
the `tidytext` vignettes!), which were incredibly helpful in creating this
tutorial.

***

# Preliminaries

First, let's get ready for our analyses. We do this by clearing our workspace
and loading in the libraries we'll need. It's good to get in the habit of
clearing your workspace from the start so that you don't accidentally have
clashes with unneeded variables or dataframes that could affect your results.

As with our other tutorials, we'll use a function here to check for all required
packages and---if necessary---install them before loading them. Implementing
this (or a similar) function is a helpful first step, especially if you plan on
sharing your code with other people.


```r
# clear the workspace (useful if we're not knitting)
rm(list=ls())
```





```r
# specify which packages we'll need
required_packages = c("tidyverse",
                      "stringr",
                      "dplyr",
                      "quanteda.textmodels",
                      "quanteda",
                      "tidytext",
                      "stopwords",
                      "janeaustenr")

# install them (if necessary) and load them
load_or_install_packages(required_packages)
```

***

# Data processing

***

## Inspecting the data

Our data are already fairly nicely processed for us, thanks to the work done in
creating the `janeaustenr` package. However, let's begin---as we always
should!---by having a look at our data.

We'll start by saving the output of the built-in function `austen_books()` to
something new and then inspecting the output. Note how we're using informative
variable names. Do a favor for others (and Future You!) and *always* use
informative variable names, along with plenty of comments. I find that it's
sometimes harder to retrace my steps through poorly commented code than it is to
just rewrite the whole thing from scratch.


```r
# let's see what we've got
book_df = austen_books()
head(book_df)
```

```
## # A tibble: 6 × 2
##   text                    book               
##   <chr>                   <fct>              
## 1 "SENSE AND SENSIBILITY" Sense & Sensibility
## 2 ""                      Sense & Sensibility
## 3 "by Jane Austen"        Sense & Sensibility
## 4 ""                      Sense & Sensibility
## 5 "(1811)"                Sense & Sensibility
## 6 ""                      Sense & Sensibility
```

### Jump in

Our inspection isn't giving us too much to start here, since so many of these
initial lines are blank. In the next chunk, try to get the code to display a
more representative subset of the text.


```r
# use this chunk to write code that will display a more 
# representative subset of the text -- in any way you want!
```

## Labeling units of the text

Dictionary and bag-of-words approaches often label data with proportions or
percentages of the target words, rather than raw counts. (***Consider**: Why
might this be?*) For researchers interested in communication, this might mean
that we would look at the turn or utterance level. In the case of the current
dataset (i.e., books), we might look at the chapter number or page number, but
this version of the data does not include page numbers. We do, however, have
chapter numbers. Let's extract those here.


```r
# create a new variable called `chapter`
book_df = austen_books() %>%
  
  # give each book their own starting point
  group_by(book) %>%
  
  # convert chapter numbers to a numeric variable
  mutate(chapter = cumsum(str_detect(text, regex("^chapter [\\divxlc]",
                                                 ignore_case = TRUE)))) %>%
  ungroup()

# show us what we've got
head(book_df)
```

```
## # A tibble: 6 × 3
##   text                    book                chapter
##   <chr>                   <fct>                 <int>
## 1 "SENSE AND SENSIBILITY" Sense & Sensibility       0
## 2 ""                      Sense & Sensibility       0
## 3 "by Jane Austen"        Sense & Sensibility       0
## 4 ""                      Sense & Sensibility       0
## 5 "(1811)"                Sense & Sensibility       0
## 6 ""                      Sense & Sensibility       0
```

## Prepare text

We should convert our text to a *long-form* dataframe, which includes one word
per line. We can do this easily with the `unnest_tokens()` call. 


```r
# get one token per row
tidy_book_df = book_df %>%
  unnest_tokens(word, text)

# take a peek at what we have
head(tidy_book_df)
```

```
## # A tibble: 6 × 3
##   book                chapter word       
##   <fct>                 <int> <chr>      
## 1 Sense & Sensibility       0 sense      
## 2 Sense & Sensibility       0 and        
## 3 Sense & Sensibility       0 sensibility
## 4 Sense & Sensibility       0 by         
## 5 Sense & Sensibility       0 jane       
## 6 Sense & Sensibility       0 austen
```
This call also takes care of a few other important steps in text processing.
(***Consider**: Compare the original `book_df` to `tidy_book_df`. How else does the `word`
column differ? What else has been done to it with this variable?*)

## Handling stopwords

Another decision that we'll need to make is how we want to deal with very common
words, also known as "stopwords." These words are often disregarded from analyses
because they lack specificity. Let's take a look at them and see what we mean.


```r
# let's choose to use the `snowball` corpus
stopword_df = get_stopwords()

# let's see what we've got here
head(stopword_df)
```

```
## # A tibble: 6 × 2
##   word   lexicon 
##   <chr>  <chr>   
## 1 i      snowball
## 2 me     snowball
## 3 my     snowball
## 4 myself snowball
## 5 we     snowball
## 6 our    snowball
```

You'll need to decide what words---if any---you want to remove from your dataset
before proceeding. In dictionary-based approaches, removal of specific words
isn't necessary, strictly speaking: You can simply not include those words in
your specified dictionaries. However, in some bag-of-words analyses, you might
want to consider removing some very common words: For example, if you are
examining the most common words in a dataset, you might not be interested in
function words (e.g., articles, prepositions) and may only be interested in
content words (e.g., nouns, verbs).

### Jump in

Decide whether you care about including all words in the dataset. In the next
chunk, try removing some target subset of words from the dataset. It could be
the words from `stopword_df` or some other words you notice might not make sense
to include in the dataset, based on your inspection of the data.


```r
# use this chunk to write code that will remove unwanted words 
# in any way you choose from the `tidy_book_df() datase
```

***

# Data preparation

***

## Using `tidytext`

We can use a number of tools within `tidytext` to implement bag-of-words
approaches. For example, we can get a list of the most common words in the
whole dataset or by subsets of the dataset.


```r
# get word count of the overall corpus
word_count_overall = tidy_book_df %>% 
  count(word, sort = TRUE) 

# take a peek at the top words overall
head(word_count_overall)
```

```
## # A tibble: 6 × 2
##   word      n
##   <chr> <int>
## 1 the   26351
## 2 to    24044
## 3 and   22515
## 4 of    21178
## 5 a     13408
## 6 her   13055
```

(***Consider**: Assess how your choice in removing text from the `tidy_book_df`
may have affected the word count.*)


```r
# get word count by book
word_count_book = tidy_book_df %>%
  group_by(book) %>%
  count(word, sort = TRUE) %>%
  ungroup()

# check out by-book word counts
head(word_count_book)
```

```
## # A tibble: 6 × 3
##   book           word      n
##   <fct>          <chr> <int>
## 1 Mansfield Park the    6206
## 2 Mansfield Park to     5475
## 3 Mansfield Park and    5438
## 4 Emma           to     5239
## 5 Emma           the    5201
## 6 Emma           and    4896
```

We can also use `tidytext` to count the number of words in pre-defined
dictionaries. For an easy example, let's tally the number of negative-sentiment
words across chapters across the books in the corpus.


```r
# let's grab the negative-sentiment words from the built-in corpus
negative_words = get_sentiments("bing") %>%
  filter(sentiment == "negative")

# what words do we have?
head(negative_words)
```

```
## # A tibble: 6 × 2
##   word       sentiment
##   <chr>      <chr>    
## 1 2-faces    negative 
## 2 abnormal   negative 
## 3 abolish    negative 
## 4 abominable negative 
## 5 abominably negative 
## 6 abominate  negative
```

```r
# let's grab the top words in each chapter in each book
tidy_word_counts = tidy_book_df %>% ungroup() %>%
  group_by(book, chapter) %>%
  summarize(words = n())
```

```
## `summarise()` has grouped output by 'book'. You can override using the `.groups` argument.
```

```r
# let's figure out the negative words present in each chapter
negative_words_by_chapter = tidy_book_df %>% ungroup() %>%
  
  # leave only negative words in the dataframe
  semi_join(negative_words) %>%
  
  # count the total negative words in each chapter in each book
  group_by(book, chapter) %>%
  summarize(negativewords = n()) %>%
  
  # join again with the original dataframe to recover the other info
  left_join(tidy_word_counts, by = c("book", "chapter")) %>%
  
  # get a ratio of the negative words to total words in each chapter
  mutate(ratio = negativewords/words) %>%
  
  # remove any of the front-of-book matter
  filter(chapter != 0) 
```

```
## Joining, by = "word"
## `summarise()` has grouped output by 'book'. You can override using the `.groups` argument.
```

```r
# check out our dataset
head(negative_words_by_chapter)
```

```
## # A tibble: 6 × 5
## # Groups:   book [1]
##   book                chapter negativewords words  ratio
##   <fct>                 <int>         <int> <int>  <dbl>
## 1 Sense & Sensibility       1            35  1571 0.0223
## 2 Sense & Sensibility       2            31  1970 0.0157
## 3 Sense & Sensibility       3            42  1538 0.0273
## 4 Sense & Sensibility       4            64  1952 0.0328
## 5 Sense & Sensibility       5            20  1030 0.0194
## 6 Sense & Sensibility       6            20  1353 0.0148
```

And there we have it---a dictionary-based approach implemented within `tidytext`!

### Jump in 

The analyses that we've done so far are at the chapter level, but
that's a very coarse scale or level of aggregation. Consider the data, and
decide on a new level of analysis that is at least somewhat natural to the data
(keeping in mind that you should be able to justify your answer). Create a new
variable (with an informative variable name) that preserves the appropriate level
of numbering or continuity, preserving at least the book level. Write new code
to run dictionary or bag-of-words analyses on this new level.

***

## Using `quanteda.dictionaries`

The `liwcalike()` function in the `quanteda.dictionaries` package provides for a
LIWC-like output from user-specified dictionaries and a handful of basic
categories (e.g., punctuation, 6-letter words). To use this, we'll need to
install some packages directly from GitHub, since these packages haven't been
made available for distribution on CRAN.


```r
# first, we install the `devtools` package
load_or_install_packages("devtools")

# then, we'll install a few packages that we'll need
devtools::install_github("quanteda/quanteda.corpora")
devtools::install_github("quanteda/quanteda.sentiment")
devtools::install_github("kbenoit/quanteda.dictionaries")

# finally, we'll load them in
library(quanteda.corpora)
library(quanteda.sentiment)
library(quanteda.dictionaries)
```

This package requires that we create specific dictionary objects to be used with
the `liwcalike()` function, but it comes with a few built-in options. For now,
we'll use the English version of the NRC Word-Emotion Association Lexicon (NRC
Emotion Lexicon; Mohammad & Charron, 2010, 2013). 


```r
# load in the NRC dictionary
data_dictionary_NRC <- as.dictionary(data_dictionary_NRC)

# take a look at the dictionary
head(data_dictionary_NRC)
```

```
## Dictionary object with 6 key entries.
## Polarities: pos = ""; neg = "negative" 
## - [anger]:
##   - abandoned, abandonment, abhor, abhorrent, abolish, abomination, abuse, accursed, accusation, accused, accuser, accusing, actionable, adder, adversary, adverse, adversity, advocacy, affront, aftermath [ ... and 1,227 more ]
## - [anticipation]:
##   - abundance, accelerate, accolade, accompaniment, achievement, acquiring, addresses, adore, adrift, advance, advent, adventure, advocacy, airport, alertness, alerts, alive, allure, aloha, ambition [ ... and 819 more ]
## - [disgust]:
##   - aberration, abhor, abhorrent, abject, abnormal, abominable, abomination, abortion, abundance, abuse, accusation, actionable, adder, adultery, adverse, affliction, affront, aftermath, aggravation, aghast [ ... and 1,038 more ]
## - [fear]:
##   - abandon, abandoned, abandonment, abduction, abhor, abhorrent, abominable, abomination, abortion, absence, abuse, abyss, accident, accidental, accursed, accused, accuser, accusing, acrobat, adder [ ... and 1,456 more ]
## - [joy]:
##   - absolution, abundance, abundant, accolade, accompaniment, accomplish, accomplished, achieve, achievement, acrobat, admirable, admiration, adorable, adoration, adore, advance, advent, advocacy, aesthetics, affection [ ... and 669 more ]
## - [negative]:
##   - abandon, abandoned, abandonment, abduction, aberrant, aberration, abhor, abhorrent, abject, abnormal, abolish, abolition, abominable, abomination, abort, abortion, abortive, abrasion, abrogate, abscess [ ... and 3,304 more ]
```

Before moving on, be sure to look at the dictionary output above. Knowing your
tools is just as important as knowing your data! You'll need to know your
dictionary categories in order to make sense of what comes next.


```r
# run LIWC-like analysis on books
liwcalike_book_df = quanteda.dictionaries::liwcalike(book_df$text,
                                                     dictionary = data_dictionary_NRC)

# check out our output
head(liwcalike_book_df)
```

```
##   docname Segment WPS WC Sixltr   Dic anger anticipation disgust fear joy
## 1   text1       1   3  3  33.33 66.67     0            0       0    0   0
## 2   text2       2   0  0   0.00  0.00     0            0       0    0   0
## 3   text3       3   3  3   0.00  0.00     0            0       0    0   0
## 4   text4       4   0  0   0.00  0.00     0            0       0    0   0
## 5   text5       5   3  3   0.00  0.00     0            0       0    0   0
## 6   text6       6   0  0   0.00  0.00     0            0       0    0   0
##   negative positive sadness surprise trust AllPunc Period Comma Colon SemiC
## 1        0    66.67       0        0     0    0.00      0     0     0     0
## 2        0     0.00       0        0     0    0.00      0     0     0     0
## 3        0     0.00       0        0     0    0.00      0     0     0     0
## 4        0     0.00       0        0     0    0.00      0     0     0     0
## 5        0     0.00       0        0     0   66.67      0     0     0     0
## 6        0     0.00       0        0     0    0.00      0     0     0     0
##   QMark Exclam Dash Quote Apostro Parenth OtherP
## 1     0      0    0     0       0       0      0
## 2     0      0    0     0       0       0      0
## 3     0      0    0     0       0       0      0
## 4     0      0    0     0       0       0      0
## 5     0      0    0     0       0       0      0
## 6     0      0    0     0       0       0      0
```

It looks---like it says!---a lot like LIWC's output, just with our dictionary
categories tucked in there.

### Jump in

Take a look at the output from the `liwcalike()` function. What information is
missing that was in our original `book_df` dataframe? Why does it not appear in
the `liwcalike_book_df` dataframe? How might we be able to get that information
back into a single dataframe?
