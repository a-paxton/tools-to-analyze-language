---
title: 'Introduction to APIs for text analysis'
author: "A. Paxton (*University of Connecticut*)"
output:
  html_document:
    keep_md: yes
    number_sections: yes
---

An API---or *application programming interface*---is a formal avenue created 
by an organization or website to programmatically access their data. We could
most directly contrast an API with *web scraping*, which we will cover in a 
future session.

To use an API, you must submit a *request* to the system. That request
includes a set of information that has been specified by the API developers.
Minimally, the request will describe the kind of data that you want to access
(in the manner outlined by the API). For some systems, this must also include an
*API key*, which is an authenticated permission identifier that allows you to
access data. 

After submitting your request, you will receive a *response* that contains the
data you've requested. Your next job will be to parse the response into usable
data.

In this exercise, we'll walk through the process of using an API directly from R.

***

# Preliminaries

First, we'll need to prepare for our exercises. We'll do this by loading the
packages we need, including---if needed---installing those packages. For the
sake of the exercise, we'll load the packages silently, meaning that we won't
clutter our R markdown output with a bunch of warnings and output messages
from the loading and installation process.


```r
# clear the workspace (useful if we're not knitting)
rm(list=ls())
```




```r
# specify which packages we'll need
required_packages = c("tidyverse",
                      "httr",
                      "jsonlite")

# install them (if necessary) and load them
load_or_install_packages(required_packages)
```

***

# Finding APIs

Not every website, repository, or organization has an API available for users to
access data, but those that do have them tend to make them easily available. You
might start by checking the website for an API information link. This might be
at the bottom of the webpage or available in a "Developers" menu of the website.
If you can't find one, try using the "find" function in your browser for `API`,
or try searching `API` in the website's search function (if they have one).

If you still can't find it, try doing a general search on your favorite search
engine with the name of the website and API. (As you may already know, putting
both terms in separate double-quotes will ensure you only get webpage results
that include both terms.)

If you *still* can't find anything, it may be time to turn to web scraping---but,
again, we won't be covering that here. (Again, keep in mind matters of
ethicality and legality, including the *hiQ Labs v. LinkedIn* ruling in the US.)

***

# Submitting a request

For this exercise, we'll be accessing the [U.S. Consumer Financial Protection
Bureau Consumer Complaint
Database](https://www.consumerfinance.gov/data-research/consumer-complaints/).
This open dataset provides information about complaints about specific consumer
financial services and products, including narratives from complainants about
what happened in their own words. For analyses of language, there are plenty of
things we *might* want to analyze---like comparing different ways that people
describe complaints from different states or across different kinds of companies
or even over time. For this example, let's just choose one organization.

To grab the information from the results, we'll be using the `httr` library's
`GET` function.

## Identify API URL

To get started, we need to figure out more about this API. A good place to start
is to take a look at the website. If you scroll down to the bottom of the linked
website (as of February 2022), you'll see a link: ["For instructions and
examples, refer to our API documentation."](https://cfpb.github.io/api/ccdb/)

Once you click on the  API link, you'll see a new website with lots of
information, including helpful links to additional documentation about the API
("API documentation") and a description of fields in this dataset ("Field
reference"). After looking at the API information, we know the URL of our
dataset (labeled as "Server" on "API documentation"). Let's create a variable to
hold onto that.


```r
# specify the API location
complaint_data_location = "https://www.consumerfinance.gov/data-research/consumer-complaints/search/api/v1/"
```

## Identify filtering parameters for request



Let's say that we want to grab all complaints against Equifax, Inc. that were
filed with a complaint narrative. In looking at the metadata list, we've found
the variable name for company (`company`) and for the inclusion of narrative
text (`has_narrative`).

Unfortunately, though, the dataset doesn't have a great description of all
possible values. We can do some of that by poking around in the [interactive
database search
tool](https://www.consumerfinance.gov/data-research/consumer-complaints/search/)
and seeing how the URL changes as we input new options. From this, we find out
that Equifax is included in the dataset as `EQUIFAX, INC.` and that the
`has_narrative` value must be equal to `true` (not `TRUE` or `1` or `True`). If
we didn't have the interactive tool, it would require a lot more interactive
exploration to try to figure out what we need to include---not impossible, but a
lot harder. This might feel clunky or inelegant, but keep in mind that a lot of
working with data always requires persistence and creativity---just like doing a
literature review or piloting an experimental paradigm does!

To use `httr::GET`, we need to prepare the variable in a `list` item. When we
input the values, we need to respect the data type in R and in the API. Given
that `EQUIFAX, INC.` is a string, we'll need to remember to encapsulate it in
double-quotes so that R doesn't get angry with us.


```r
# specify what we want to filter by
return_variables = list(company="EQUIFAX, INC.",
                        has_narrative = "true")
```

## Create and submit the request

Finally, let's put together the pieces we've already created and submit the
request to the API.


```r
# create and submit the request
request = GET(url = complaint_data_location,
              query = return_variables)

# show us what we received
request
```

```
## Response [https://www.consumerfinance.gov/data-research/consumer-complaints/search/api/v1/?company=EQUIFAX%2C%20INC.&has_narrative=true]
##   Date: 2022-02-09 21:18
##   Status: 200
##   Content-Type: application/json
##   Size: 339 kB
```
Great! It looks like our submission worked. It's useful to check for errors by
ensuring that the `Status` value of the response is `200`. You can do this by
manually inspecting the output (as seen above) or by calling it from the 
returned value, as shown below:


```r
# programmatically grab the response value
request$status_code
```

```
## [1] 200
```

Now, just like we used the browser before to check some of our progress, we can
use the browser again to get an idea of what the results look like. We can copy
and paste the "Response" URL (output above) into a browser to see what the
database is giving us in its raw, unprocessed form. This can allow quick
iterating for values and test queries to see if we submit something that won't
work or gives zero hits. You don't *have* to do this, of course, but it's
another tool in your tool kit for using APIs.

## A quick note about API keys or tokens

As mentioned in the beginning of this exercise, some APIs may gate access to
their data by issuing API keys or tokens that must be included in a request. In
some cases, the keys may be required to access the API at all; in other cases,
they may only be required if a user is requesting data above a specific rate or
above a specific volume. The idea behind the token is that it identifies the
requester to the organization. Any API that requires a key should have detailed
information about how to request one and any restrictions for use; be sure to
consult the documentation for specific information from each organization, since
these policies vary widely.

***

# Debugging

Before we move onto parsing the data, let's take a moment to consider some possible
issues with our requests.

## Problems with variable names

Let's say that we had a typo in our request, using `hasnarrative` instead of
`has_narrative` for a variable name.


```r
# let's say that we forgot the underscore in the variable
return_variables_with_name_typo = list(hasnarrative = "true")

# we'll create and submit our faulty request
request_with_name_typo = GET(url = complaint_data_location,
                             query = return_variables_with_name_typo)

# show us what we received
request_with_name_typo
```

```
## Response [https://www.consumerfinance.gov/data-research/consumer-complaints/search/api/v1/?hasnarrative=true]
##   Date: 2022-02-09 21:18
##   Status: 200
##   Content-Type: application/json
##   Size: 78.2 kB
```

As you can see, it is possible for us to correctly submit something from the R
side that the API won't accept. In this case, `httr::GET()` received everything
in the correct format and therefore submitted our request to the API. However,
the API sent back an error to let us know that there was an unrecognized
variable name. If you receive this kind of error, be sure to go back to the list
of variables and check for typos.

## Problems with variable values

It's important to contrast the typo in a variable name with a typo in the
variable *value*. In this example, let's say that we correctly specify the
variable name (`timely`) but that we have a typo in our desired value (`Yas`
instead of `Yes`).


```r
# let's say that we forgot the underscore in the variable
return_variables_with_value_typo = list(timely = "Yas")

# we'll create and submit our faulty request
request_with_value_typo = GET(url = complaint_data_location,
                             query = return_variables_with_value_typo)

# show us what we received
request_with_value_typo
```

```
## Response [https://www.consumerfinance.gov/data-research/consumer-complaints/search/api/v1/?timely=Yas]
##   Date: 2022-02-09 21:18
##   Status: 200
##   Content-Type: application/json
##   Size: 1.75 kB
```

Again, `httr::GET()` had no problems submitting this request. However, we don't
get the nice error message from the API this time. Instead, because we submitted
something with existing variable names, it simply winnowed down the list and
returned every record with a value of `Yas` in the `timely` variable---that
is to say, zero records. This is a more obvious problem if the typo resulted in
a nonexistent value in the variable (because it will return zero records), but
it could pose a more serious problem if the typo results in a valid value
*other* than the one you intended. This highlights the importance of checking
our output for expected values *and* checking your code for accuracy.

*** 

# Parsing output

Now that we have our intended response (saved in the `request` variable), we'll
need to convert it into something usable. (To avoid cluttering the output, the
rendered or `knit` R markdown does not show the results. However, you can see
the results if you run the code interactively.)


```r
# convert the response's values to a dataframe in one go
response_df = (data.frame(
  fromJSON(
    content(request, as = "text"), 
    flatten=TRUE)))
```

I've prevented this chunk from rendering to avoid throwing errors when trying to
knit, but this code would give us some trouble! The code above has worked for
other APIs, but it looks like it won't work here. Here's what we'd get:


```r
Error in (function (..., row.names = NULL, check.rows = FALSE, check.names = TRUE, : arguments imply differing number of rows: 1, 0, 25
```

It isn't terribly unusual that the same code that worked for a different API
wouldn't work for this one. The output and structure of each API may differ,
since everyone has their own database or server structure (and the APIs are ways
of accessing those servers and databases). We'll need to do a little sleuthing
to figure out how we get access to those data in a manageable format. Visually
inspecting the output can help us, either by clicking in the Environment pane or
by using the `View()` function can be helpful here.


```r
# convert the response's values to text
response = content(request, as = "text")
```

```
## No encoding supplied: defaulting to UTF-8.
```

```r
response_flattened = fromJSON(response, flatten=TRUE)

# inspect the output -- again, hidden from rendered R markdown due to space
head(response_flattened)
```

Okay, we're getting somewhere! It looks like the item that we have is actually
10 nested items together. (This shows up in the Environment pane as `List of 10`
and also as 10 boxes in the R markdown.) Investigating the results a bit, it
looks like what we want to analyze here will be in the
`_source.complaint_what_happened` variable. Using the data viewer, we see that
it's located within the `hits` list. Let's see what happens when we go there.


```r
# grab the target list
response_target_list = response_flattened$hits

# take a peek at what we get -- again, hidden from rendered R markdown due to space
head(response_target_list)
```

We're getting closer! It looks like there are still two items, though,
meaning that the `response_flattened$hits` was actually a list of two items.
From scrolling through the results, it looks like the `_source.complaint_what_happened` variable
is in an object called `hits`. Let's zoom in.


```r
# grab the results we want
response_df = response_flattened$hits$hits

# inspect results -- which we'll show in the rendered R markdown
head(response_df)
```

```
##                _index _type     _id _score       sort
## 1 complaint-public-v1  _doc 4999421      1 1, 4999421
## 2 complaint-public-v1  _doc 4999380      1 1, 4999380
## 3 complaint-public-v1  _doc 4999058      1 1, 4999058
## 4 complaint-public-v1  _doc 4999043      1 1, 4999043
## 5 complaint-public-v1  _doc 4999015      1 1, 4999015
## 6 complaint-public-v1  _doc 4999013      1 1, 4999013
##                                                                _source.product
## 1 Credit reporting, credit repair services, or other personal consumer reports
## 2 Credit reporting, credit repair services, or other personal consumer reports
## 3 Credit reporting, credit repair services, or other personal consumer reports
## 4 Credit reporting, credit repair services, or other personal consumer reports
## 5 Credit reporting, credit repair services, or other personal consumer reports
## 6 Credit reporting, credit repair services, or other personal consumer reports
##                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        _source.complaint_what_happened
## 1                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    XXXX XXXX acct # XXXX - disputed multiple times came back verified, I am not now nor have i ever been a customer of this company, the credit bureau has failed to provide an original signed copy of any agreement as required and are therefore reporting information that is not 100 % accurate
## 2                                                                                                                                                                                                                                                                                                               Hello! My name is XXXX XXXX. My current address is at XXXX XXXX XXXX XXXX XXXX CA XXXX. \nI have sent out numerous disputes trying to remove inaccurate names, addresses, and accounts from my credit profile. \nBelow are the dates, account ( XXXX ) and account number ( XXXX ) .This request has been denied and I would like these inaccurate information to be removed. \n\n\n\nIN XXXX XXXX AND EQUIFAX XXXX XXXX XXXX XXXX XXXX XXXX XXXX IN XXXX AND XXXX XXXXXXXX XXXX XXXX XXXX XXXX XXXX IN XXXX ONLY XXXX XXXX XXXX XXXX XXXX XXXX XXXX  IN EQUIFAX ONLY XXXX XXXX XXXXXXXX XXXX IN XXXX AND EQUIFAX ONLY XXXX I would also like these inaccuraXXXX information to be updated XXXX \n\n\n\nIN XXXX XXXX AND EQUIFAX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXXXXXX XXXX XXXX XXXXXXXX IN XXXX XXXX AND EQUIFAX XXXX XXXX XXXX XXXX XXXX XXXX ONLY XXXX XXXX XXXX XXXX
## 3                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               Fraud somebody used my identity and now I have 4 inquirys on my report I didnt make. \n\nXXXX XXXXXXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX
## 4                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              I am a victim of identity theft. XXXX has fraudulent accounts on my credit profile that need to be removed immediately.
## 5 Account Name : XXXX Account Number : XXXX Balance : {$4900.00} This is a formal complaint against Equifax, XXXX, and XXXX located in XXXX XXXX XXXX XXXX GA XXXX, XXXX XXXX XXXX XXXX TX XXXX, XXXX XXXX XXXX XXXXXXXX PA XXXX. This company has repeatedly violated my consumer rights under the Fair Credit Reporting Act and has caused me much unnecessary financial AND XXXX XXXX \n\nI am not questioning whether I owe this debt or not, I am challenging the consistencies of the reporting of this account to the credit bureaus. I am aware that under FCRA, the credit reporting agencies must report any account 100 % accurate and complete. I checked my credit report again today and I do not see any changes or corrections at all that the credit bureaus are supposed to be doing in order ensure the maximum compliance and accuracy of what they are reporting. \n\nFor starters, they're reporting an erroneous & unverifiable account on my credit report and not to mention an account in which I've asked for proof of claim and in which they have not been able to provide, per the FCRA. Despite my efforts to resolve this for several months now, Equifax, XXXX, and XXXX have completely ignored my communications and legal submissions to remove this inaccurate information from my credit report.
## 6                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     I became a victim of identity theft couple of years ago. After that incident I have noticed many incorrect and unauthorized items appeared in my report. So, I am requesting that you delete all the following ACCOUNTS AND INQUIRIES from my credit report immediately. They are a result of ID theft. Beside these items, I also want to delete incorrect names and addresses also. I never used these names and never lived in these addresses. Please remove these items which are not authorized by me as soon as possible.
##   _source.date_sent_to_company
## 1    2021-12-10T12:00:00-05:00
## 2    2021-12-10T12:00:00-05:00
## 3    2021-12-10T12:00:00-05:00
## 4    2021-12-10T12:00:00-05:00
## 5    2021-12-10T12:00:00-05:00
## 6    2021-12-10T12:00:00-05:00
##                                                                      _source.issue
## 1 Problem with a credit reporting company's investigation into an existing problem
## 2 Problem with a credit reporting company's investigation into an existing problem
## 3                                             Incorrect information on your report
## 4                                             Incorrect information on your report
## 5 Problem with a credit reporting company's investigation into an existing problem
## 6 Problem with a credit reporting company's investigation into an existing problem
##   _source.sub_product _source.zip_code _source.tags _source.has_narrative
## 1    Credit reporting            70068         <NA>                  TRUE
## 2    Credit reporting            94530         <NA>                  TRUE
## 3    Credit reporting            60555         <NA>                  TRUE
## 4    Credit reporting            31904         <NA>                  TRUE
## 5    Credit reporting            38127         <NA>                  TRUE
## 6    Credit reporting            63103         <NA>                  TRUE
##   _source.complaint_id _source.timely _source.consumer_consent_provided
## 1              4999421            Yes                  Consent provided
## 2              4999380            Yes                  Consent provided
## 3              4999058            Yes                  Consent provided
## 4              4999043            Yes                  Consent provided
## 5              4999015            Yes                  Consent provided
## 6              4999013            Yes                  Consent provided
##   _source.company_response _source.submitted_via _source.company
## 1  Closed with explanation                   Web   EQUIFAX, INC.
## 2  Closed with explanation                   Web   EQUIFAX, INC.
## 3  Closed with explanation                   Web   EQUIFAX, INC.
## 4  Closed with explanation                   Web   EQUIFAX, INC.
## 5  Closed with explanation                   Web   EQUIFAX, INC.
## 6  Closed with explanation                   Web   EQUIFAX, INC.
##       _source.date_received _source.state _source.consumer_disputed
## 1 2021-12-10T12:00:00-05:00            LA                       N/A
## 2 2021-12-10T12:00:00-05:00            CA                       N/A
## 3 2021-12-10T12:00:00-05:00            IL                       N/A
## 4 2021-12-10T12:00:00-05:00            GA                       N/A
## 5 2021-12-10T12:00:00-05:00            TN                       N/A
## 6 2021-12-10T12:00:00-05:00            MO                       N/A
##   _source.company_public_response
## 1                              NA
## 2                              NA
## 3                              NA
## 4                              NA
## 5                              NA
## 6                              NA
##                                         _source.sub_issue
## 1 Their investigation did not fix an error on your report
## 2 Their investigation did not fix an error on your report
## 3                     Information belongs to someone else
## 4                     Information belongs to someone else
## 5 Their investigation did not fix an error on your report
## 6                    Investigation took more than 30 days
```

Fantastic! Our dataframe looks well-formed and ready to go.

***

# Next steps

Of course, just because we have a dataframe, doesn't mean we can jump right into 
analyses. You'll want to be sure to go through all of the steps of cleaning and
preparing your dataset (e.g., converting variables from `chr`, looking for missing
values, transforming your data), which we'll go through in later sessions.

***
