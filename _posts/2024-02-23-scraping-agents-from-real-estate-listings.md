---
layout: post
title: "Scraping Agents from Real Estate Listings with Python and Beautiful Soup"
categories: scraper
---
While looking through [rental properties on realestate.com.au](https://www.realestate.com.au/rent/in-carlisle,+wa+6101/list-1), I noticed that some leasing agents were posting many more listings than others. This made me wonder how I could obtain a frequency table of the leasing agents' number of listings. That would be a useful indicator of which agents were more prolific than others.

## Finding the agent name in the HTML file

To begin, given a single page of listing results, I needed to find the text corresponding to the agent name in each listing panel. Inspecting the HTML of a listing panel, I found the agent logo `img` tag with the class `branding__image`. This `img` tag had an `alt` attribute that consistently contained the agent name (in the case below, it is 'Xceed Real Estate - HERDSMAN'):

```html
<img class="branding__image" alt="Xceed Real Estate - HERDSMAN" src="downloaded1_files/logo_011.png">
```

HTML attributes such as these are easy to extract using [Beautiful Soup](https://www.crummy.com/software/BeautifulSoup/bs4/doc/) in Python, it eats "find tag x with attributes y" tasks for breakfast.

I like to test scraping code on an offline copy before I let it loose on the web. So I downloaded the first page of real estate listings, then scanned it with Beautiful Soup for the agent names:

```python
exfile = open('agents.html')
exsoup = bs4.BeautifulSoup(exfile,'html.parser')
agents = exsoup.find_all("img", {"class": "branding__image"})

for agent in agents:
    print(agents['alt'])
```

So far so good, this printed out all the agent names on a single downloaded page.

## Frequency table of agents

Now for the frequency table. Googling how to get a frequency table gets you pages of how to do it in a single line of [Pandas](https://pandas.pydata.org/) (bravo Pandas). But I happened to be working on a low-spec machine (a 2013 Celeron Haswell with 4GB/64GB RAM/HDD), so I didn’t really want to install Pandas. The ‘proper recommended’ way to install Pandas is via [Anaconda](https://www.anaconda.com/), and I didn’t want to go down this path, as Anaconda would choke my little Celeron and limited HDD. I considered installing Pandas alone using [pip](https://pypi.org/project/pip/), but then even better, I remembered an elegant way ('Python pattern') of doing a frequency table using a [Python dictionary](https://docs.python.org/3/tutorial/datastructures.html#dictionaries). [Charles Severance](https://www.dr-chuck.com/) spoke of it when I did his popular online course [Python for Everybody](https://www.coursera.org/specializations/python) some time ago. I found [the reference](https://eng.libretexts.org/Bookshelves/Computer_Science/Programming_Languages/Python_for_Everybody_(Severance)/10%3A_Tuples/10.06%3A_The_most_common_words) (the Severance reference if I may), and implemented it here:

```python
‌ag_freq = dict()

for tag in agents:
    agent = tag['alt']
    if agent in ag_freq:
        ag_freq[agent]+=1
    else:
        ag_freq[agent]=1
```

It got what I wanted from the first downloaded page of results (without having to install Anaconda or Pandas). So now it’s time to let it loose on the web.

## Live scraper: headers and cookies

I had previously [scraped jazzstandards.com](https://github.com/dmudigdo/jazzstandards1000) and Wikipedia without issues, so I got straight to it, starting with just page 1 of the results to test:

```python
‌exfile = requests.get("https://www.realestate.com.au/rent/in-carlisle,+wa+6101/list-1")
print(exfile.status_code)
```

However this gave an error. A [429](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/429) ('Too many requests') in fact. This puzzled me for a while, because the downloaded page worked fine just then. And I only made one request! It seemed that realestate.com.au had a way of detecting bots (automatic web crawlers) that jazzstandards.com and Wikipedia did’t. After consulting the online oracles at stackoverflow.com and elsewhere, it turned out that my code triggered a 429 because it didn’t send the headers that a browser would have.

Getting headers is straightforward: go to a webpage, open the Inspector, and go to the Network tab. There, find the entry corresponding to the HTML file, right click, copy as cURL and paste it into [curlconverter.com](http://curlconverter.com). This will give you headers that act as a ['these are not the droids you are looking for'](https://www.youtube.com/watch?v=532j-186xEQ). But pasting this into my code, it still didn’t work! Still got a 429. The curlconverter.com output did however have a cookies entry that I hadn't copy-pasted (only did the headers). Sure enough, when I copy-pasted both the headers and the cookies from curlconverter.com into my code, it worked! Ah, the relieving [200 error code](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/200) ('OK). Here is the code improved from the previous one:

```python
headers = lots_of_stuff_from_curlconverter.com
cookies = lots_of_other_stuff_from_curlconverter.com

‌exfile = requests.get("https://www.realestate.com.au/rent/in-carlisle,+wa+6101/list-1", headers=headers, cookies=cookies)
print(exfile.status_code)
```



I later found out that the header-cookie combination expired, not sure how long later exactly, but the next day the header-cookie gave another 429, and I had to obtain a fresh header-cookie for it to work again.

## Automating the navigation

Now that the code worked for the first online page, it was time to automate the navigation. I needed the code to go to the first page, scrape the real estate agent names, go to the next page(s), then terminate at the last page. I decided that the condition for termination should be the absence of a ‘Next’ button. On realestate.com.au, the next page `<a>` had a title attribute `Go to Next Page`:

```html
‌<a href="https://www.realestate.com.au/rent/in-carlisle,+wa+6101/list-4" class="ButtonBase-sc-18zziu4-0 Link__LinkWithButtonVariant-sc-8zfb96-0 fxTsFD cmIERf" title="Go to Next Page" rel="next">
```

So, I used this as the terminating condition in a loop that started from the first page, then kept following the link on the ‘Next’ button if it existed:

```python
‌# Initialise
ag_freq = dict()
nextURL = "https://www.realestate.com.au/rent/in-carlisle,+wa+6101/list-1"

# Loop through pages
while nextURL != "terminate":

    exfile = requests.get(nextURL,headers=headers,cookies=cookies)
    print(nextURL, exfile.status_code)
    exsoup = bs4.BeautifulSoup(exfile.text,'html.parser')

# Omitted: do all the scraping here...

    next = exsoup.find_all("a",{"title": "Go to Next Page"})
    print(next)
    if not next :
        nextURL = "terminate"
    else:
        nextURL = "https://www.realestate.com.au" + next[0]['href']
```

(Note that in Python, `if not next` is the same as `if next = []`, pretty snazzy)

So there you have it: a frequency table of the real estate agents from online rental listings. It helped me target the real estate agent that had the most listings, which indicated they had a decent rentals team. You can find the [code on my GitHub page](https://github.com/dmudigdo/lessor-agents).
