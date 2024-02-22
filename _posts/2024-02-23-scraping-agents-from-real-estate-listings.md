---
layout: post
title: "Scraping Agents from Real Estate Listings"
categories: misc
---
While looking through rental properties on [realestate.com.au](http://realestate.com.au "‌"), I noticed that some leasing agents were posting more listings than others. This made me think about how I could obtain an accurate frequency histogram of the leasing agents' number of listings.

## Finding the agent name in the HTML file

I was looking for a bit of text in a listing panel that consistently showed the text of the agent name. Inspecting HTML corresponding to a listing panel, I found that the `alt` attribute of the `img` tag of the agent logo in the listing consistently contained the test of the agent name. That logo `img` tag consistently had the class `branding__image`:

```html
<img class="branding__image" alt="Xceed Real Estate - HERDSMAN" src="downloaded1_files/logo_011.png">
```

This is the sort of thing that is easy to extract using Beautiful Soup in Python, it eats "find x tag with attributes y" type of tasks for breakfast.

I like to test scraping code on an offline file before I let it loose on the web. So I downloaded the first page of listings as an HTML file, and scanned it using the following snippet:

```python
exfile = open('agents.html')
exsoup = bs4.BeautifulSoup(exfile,'html.parser')
agents = exsoup.find_all("img", {"class": "branding__image"})

for agent in agents:
    print(agents['alt'])
```

So far so good, the above printed out all the agent names.

## Frequency histogram of agents

Now for a frequency histogram. Googling how to get a frequency histogram gets you pages full of how to do it in a single line of [pandas](https://pandas.pydata.org/ "‌") (bravo pandas). But I happened to be working on one of my low-spec machines (a 2013 Celeron Haswell with 4GB/64GB RAM/HDD) so I didn’t want to install pandas. The ‘proper recommended’ way to install pandas is via [Anaconda](https://www.anaconda.com/ "‌"), and I didn’t really want to go down this path on my little Celeron and limited HDD. I did however remember in the back of my mind, an elegant way ('python pattern') of doing a frequency histogram using a Python dictionary. [Charles Severance](https://www.dr-chuck.com/ "‌") spoke of it when I did his popular online course [Python for Everybody](https://www.coursera.org/specializations/python "‌") back in 2015. [I found the reference](https://eng.libretexts.org/Bookshelves/Computer_Science/Programming_Languages/Python_for_Everybody_(Severance)/10%3A_Tuples/10.06%3A_The_most_common_words "‌"), and implemented it here, an elegant way to count frequencies using a python dictionary:

```python
‌ag_freq = dict()

for tag in agents:
    agent = tag['alt']
    if agent in ag_freq:
        ag_freq[agent]+=1
    else:
        ag_freq[agent]=1
```

It got what I wanted from the first downloaded page of results (without having to install anaconda). So now it’s time to let it loose on the web.

## Live scraper: headers and cookies

I had previously scraped [jazzstandards.com](http://jazzstandards.com "‌") and Wikipedia without issues, so I got straight to it, starting with page 1 of the results to test:

```python
‌exfile = requests.get("https://www.realestate.com.au/rent/in-carlisle,+wa+6101/list-2",headers=headers,cookies=cookies)
print(exfile.status_code)
```

However this gave an error. A [429](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/429 "‌") in fact. This puzzled me for a while, because the downloaded page worked fine just then! It seemed that [realestate.com](http://realestate.com "‌") has a way of detecting bots (automatic web crawlers) that [jazzstandards.com](http://jazzstandards.com "‌") and Wikipedia don’t. After consulting the online oracles at [stackoverflow.com](http://stackoverflow.com "‌") and elsewhere, it turned out that my python code triggered a 429 because it didn’t send the headers that a browser would have.

Getting the headers is straightforward, go to the webpage in question, open the Inspector of your browser of choice, and go to the Network tab. There, find the entry corresponding to the HTML file, right click, copy as cURL and paste it into [curlconverter.com](http://curlconverter.com "‌"). Et voila, the headers that will hopefully help my bot get through ('these are not the droids you are looking for'). But pasting this into my code, it still didn’t work! Still got a 429. The [curlconverter.com](http://curlconverter.com "‌") output did however have a cookies entry that I didn’t copy-paste (I only did the headers). Sure enough, when I copy-pasted both the headers and the cookies code from curlconverter into my code, it worked! Hip hip hurrah for the [200 error code](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/200 "‌").

## Automating the navigation

Now that the code worked for the first page, it was time to automate the navigation to obtain results across multiple webpages, I needed the code to go to each page, scrape the real estate agent names, and terminate at the last page. I decided that the condition for termination should be the non-appearance of a ‘Next’ button. On [realestate.com.au](http://realestate.com.au "‌"), the next page `<a>` had a title attribute `Go to Next Page`:

```html
‌<a href="https://www.realestate.com.au/rent/in-carlisle,+wa+6101/list-4" class="ButtonBase-sc-18zziu4-0 Link__LinkWithButtonVariant-sc-8zfb96-0 fxTsFD cmIERf" title="Go to Next Page" rel="next">
```

I used this the terminating condition in a loop that started from the first page, then kept following the link on the ‘Next’ button if it exists:

```python
‌# Initialise
ag_freq = dict()
nextURL = "https://www.realestate.com.au/rent/in-carlisle,+wa+6101/list-1"

# Loop through pages
while nextURL != "terminate":

    exfile = requests.get(nextURL,headers=headers,cookies=cookies)
    print(nextURL, exfile.status_code)
    exsoup = bs4.BeautifulSoup(exfile.text,'html.parser')

# Do all the scraping here...

    next = exsoup.find_all("a",{"title": "Go to Next Page"})
    print(next)
    if not next :
        nextURL = "terminate"
    else:
        nextURL = "https://www.realestate.com.au" + next[0]['href']
```

So there you have it, some DIY coding that constructed a frequency histogram of the real estate agent that were posting rental listings. It helped me target the real estate agent that had the most listings, which indicated they had  decent rentals team. You can find the [code on my GitHub page](https://github.com/dmudigdo/lessor-agents).
