---
title: Scraping A Website With Python - A Short Guide to a Short Script
date: 2025-09-20 21:16
categories: [Development,Scripts]
tags: [scripts, programming, scraping, html, development, python, json, web, data, asp.net core, .net]
image: https://strgdsysburtcher.blob.core.windows.net/burtchernet/images/ffvii-shinra.webp
---

## I Need That Content

Despite working in and around the web platform for many years, I've (strangely) never had cause to extract data from the front end of a website - at least not to the extent where it's been worth building a tool to do so programatically.

Recently I've been completely rebuilding a client's website, including the back end. I like the architecture wrangling and design, less so the content stuff, but the nature of the mission meant I had a lot of blog-type News posts to port. The platform (a proprietary CMS) it currently sits on doesn't have a export function which can pull out all their historic content in a useful way - nor does it offer direct db access. 

So, no sneaking the data out the back. Rather than delve into a boring admin process of seeing if that could be changed, I thought it could be more fun (and faster) to see how short I could make a script which would grab me that data and twist it into an importable format. Busting on in the front, as it were.

I'd already set up the new back-end with a CRUD API so I knew the shape I wanted the data in; all I had to do was pop it out. Here's how I did it with what I hope are first-time friendly explanations.

## Let's Bust On In!

My home stack is .NET and everything that goes with it: enterprisey architecture, Azure deployments, strongly-typed C# and so on. For this kind of project, that's just all too much. **Python** is really well suited to these kind of one-shot scripts and I've come to like the freedom of using something un-fussy when I'm making something that doesn't really need maintenance, or collaboration - just a tool that only I will ever care about. It doesn't need to be elegant, it just needs to *work*.

There's a million ways to do this - this is how I did it.

### Planning the Script
The script needed to:
- Get me all the existing news urls, according to a pattern
- Grab the content html & images from each page, ignoring the template scaffolding
- Tidy up any oddities
- Get the data into a structured format ready to be programatically submitted to my API

Actually the last step *could* have been "submit directly to the API" but a) I wanted to preserve an archive and b) it's never a bad idea to have the ability to make a few find/replace changes manually. These kind of scripts so often give you 98% of what you want and re-running the whole thing just to encompass an edge case I missed isn't part of the game. This is supposed to be a break from the unit testing and future proofing of my production codebases, after all.

Still - it's a simple enough plan. The next step was to leverage existing libraries best suited to the job. Not reinventing the wheel is always a good thing, and the other delight of these one-shot scripts is worrying about dependencies and long term reliability is not really a thing (not to say the packages I chose below are in any way flaky: they're all well looked after and presumably here to stay).

### Packages

I'll briefly run through the packages / libraries one by one.
```python
import requests
from bs4 import BeautifulSoup
import json
import re
```
**Requests** is perfect for this: make http requests, get responses. If you're new to this, think "this is the computer visiting the url for me, and returning all the data". [Super easy to use](https://pypi.org/project/requests/).

**BeautifulSoup** is the real magic of the script. This [fab library](https://pypi.org/project/beautifulsoup4/) helps you traverse the DOM programatically, or in human speak, it can look at all the HTML data returned in the request (**D**ocument **O**bject **M**odel) in a highly sophisticated way, identifing and manipulating different elements as the program instructs. Read on to see some good examples.

**json** is the python module for encoding and decoding json (for my structured data output)

**re** is the python module for regular expressions, 'regex' (for helping find all the content urls)

### Flow & Functions
Scripts like this, I know, don't really need to be broken up much, but I'm a clean coder at heart so I have to stick to some of the basics. I broke up the plan into four functions (be grateful it's not more):

```python
def loop_through_archive(maxpages) # Work with the existing site's Pagination
def discover_news_urls(url) # Pull out href's matching a /news pattern
def process_page(url) # Get the stuff, make it readable, save it (could've been split up more)
def clean_up_string(string) # Deal with any weirdness
```

#### loop_through_archive
This is the most straightforward function; it's really just a looping wrapper for the `discover_news_urls()` function. It's got a `maxpage` variable (an *int*, in my mind but in anarchic, typeless python-land, *anything at all*) just so it can neatly stop at the final page.
```python
BASE_URL = 'http://burtcher.net'
def loop_through_archive(maxpage):
    all_news_urls = set()
    for page in range(1, maxpage + 1):
        url = f"{BASE_URL}/news?page={page}"
        print(f"Discovering news URLs on page {page}...")
        news_urls = discover_news_urls(url)
        print(f"Found {len(news_urls)} news URLs on page {page}")
        all_news_urls.update(news_urls)
    return all_news_urls
```
_In ASP.NET Core, we ILogger everything. I love printing directly to the console instead. So edgy._
{: .syntax-caption }
Very simply, we:
- Create a collection of somethings (`set()`) with the name `all_news_urls`
- Loop through a `range` from 1 to `maxpage + 1`
- In each iteration of the loop, bump up the pagenumber on the website's news archive `url`.
- Create a new collection called `news_urls` and use it to catch whatever `discover_news_urls()` puts out when we feed that archive page's url to it.
- Add any discovered urls to the collection of `all_news_urls` and return it to the calling function.

#### discover_news_urls
This is where I get to use the fun libraries. The goal is to find any specific news pages by looking for occurences of the (slightly unusual) pattern "https://burtcher.net/{slug}/news" in the text of each archive page.

So assuming `url` is the archive page where we expect to find a few links to historic articles, we need to visit it:
```python 
r = requests.get(url)
r.raise_for_status()
```
That's all there is to using the **requests** package (for something this simple anyway) - make a GET request with `.get()`. I then use the `raise_for_status()` method which will basically throw an exception if the http response isn't a success code. This is a quick & useful thing for a low stakes script like this where I want it to break quickly if I've gotten the plumbing wrong and am feeding it bad urls (whereas in a big application there'd be a whole error handling and logging bit to shove in here).

So, if there's no error we can assume we've gotten some page content into the `r` variable. To do something with it, we need to bring out the Soup.

```py
soup = BeautifulSoup(r.text, "html.parser")
```
This creates an instance of BeautifulSoup to *decode* and then *parse* the HTML (passed in with `r.text` from the request). The `soup` object is a "parse tree" - a DOM that soup can read and manipulate, which is what we want. We're going to run a fork through that soup and pick out the bits we need: in this case, urls to news articles. To do that, we'll need to establish the pattern.

Hooray, regex:
```py
NEWS_URL_PATTERN = re.compile(r"^https?://burtcher\.net/[^/]+/news/")
```
Calling the `compile()` method on `re` (the regular expression library mentioned earlier) and passing in a regex string creates a pattern which we can use to search stuff.

> Note: I have never found regex syntax simple or memorable and explaining it is not something you want me to try and do (even if the above is basically the simplest example you could imagine). Perhaps it was always too easy to look up, so it never went in properly - but whatever the reason, it just will not stick in my head, despite it being brilliantly useful and the existence of some [excellent resources](https://www.regular-expressions.info/quickstart.html) to make it easy to learn. There are many things to say about AI but in this one very specific instance I can say I am excited that this is no longer something I have to care very much about.
{: .prompt-emphasis }

With the pattern established and the soup ready for sifting, lets loop through soup:
```py
urls = set()
    for a in soup.find_all("a",href=True):
        href = a["href"]
        if href.startswith("/"):
            href = BASE_URL + href
        elif href.startswith("http"):
            pass
        else:
            href = BASE_URL + "/" + href

        if NEWS_URL_PATTERN.match(href):
            urls.add(href)
    return sorted(urls)
```
What's going on here?
- Create a new collection called `urls` to store any that we find.
- `soup.find_all("a",href=True)` returns a collection of `<a>` elements which match the filter `href=True`, i.e., anchor tags which have an href attribute. This is the collection we then loop through.
- For each of these tags, we first see if the link starts with "http". If it does, we *skip* it (moving onto the next iteration of the loop with the `pass` keyword). This is because, in the case of this site, all the links we want to collect are relative (e.g. "/news/whatever"), so any http links are going off-site.
- For these discovered relative links, we prepend our domain and check to see if the result matches our pattern with `NEWS_URL_PATTERN.match(href)`. 
- If it's a match, we can assume it's a news article url, and add it to our array.
- Finally we return a sorted list using the native `sorted()` method. Why? (If you're wondering why, that's fair. Habit, really. In this instance there's no real reason to sort the results. Beware habits. But also don't sweat the small stuff, sort if you want to sort.)

After those loops are done running, we're left with the first layer of the prize: **a list of real life news urls** that we didn't have to go figure out ourselves. Nice. 

#### process_page



