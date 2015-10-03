---
layout: post
title:  "Messing with MTA Subway Turnstile Data"
author: "Sher Minn C"
date:   2015-10-03 11:20:00
event: "Week 7 at Recurse Center"
categories: projects

---

So I haven't been as good at blogging for the past month. My last post was on week 3 at RC, yikes. In the meantime, I have:

* implemented [Hough transform](https://github.com/piratefsh/image-processing/tree/f/hough) line and shape detection
* built a small [What is the color of ___?](http://piratefsh.github.io/what-color-is/#puppies) app
* mucked around with [CSS3 animations](http://codepen.io/piratefsh/pen/YyGJYy)
* attempted the [Matasano Crypto Challenge](http://cryptopals.com) of which, I am still stuck on set 1, challenge 6. 

And most recently poked around some MTA Subway data. 

## Doing stuff with the MTA data

For the uninitiated, the [Metropolitan Transit authority, MTA](http://www.mta.info/) runs the subway and buses in NYC. And they have an [API](http://web.mta.info/developers/). Whattt. 

### Turnstile data!
I didn't play with the API though. I was more interested in their [Turnstile data](http://web.mta.info/developers/turnstile.html) which has weekly logs of cumulative entries and exits per turnstile per subway station at some time interval. 

Which means, perfect for subway traffic analysis. Only that the data was in 200,000 line long, 20MB txt files in two different formats. For one week's worth of numbers. Nnnnggghhh.

### Scraping, parsing, cleaning with SQL!
I spent quite a bit of time writing code to parse the data and to dump it into a database. Let me just say that the data was not at all pretty. Logging intervals vary by stations and happen and different times of the day. The turnstile counters also get reset every now and then on different days. 

I wrote a handy command line tool to scrape MTA data so you can specify a date range and it will grab the files from that range and dump the raw data in a db. 
You can find it on Github here and a sample dataset for Sept 2015 here:

<a href="https://github.com/piratefsh/mta-turnstile-scraper" class='btn btn-primary'>Scraper</a>
<a href="https://piratefsh.github.io/assets/files/mta-turnstile-sept.db" class='btn btn-secondary'>Sept 2015 data</a>

I also wrote a script to clean the data and do some simple cleaning. More details on the README:

<a href="https://github.com/piratefsh/mta-turnstile-cruncher" class='btn btn-primary'>Data cleaner</a>

Because of all the wonky different formatting, writing simple tests to make sure that my code was working turned out to pay dividends, especially when I started adding edge cases and working with new things like SQLite. This was also my first time writing SQL queries and learning things like:

* how to select and group stuff!
* queries are hella slow because it's reading from disk and should be minimized
* learning how to think in tables. grouping data and working with those chunks speeds things up a lot.

### Thinking in tables
I talked to from [Jesse](https://github.com/jessechen), an RC alum who was super helpful with thinking about the data. The main thing I was trying to do was to find the number of entries/exits for a particular timeframe (which wasn't explicit because MTA only gives you the running cumulative numbers at every entry). 

Instead of having to look through the entire database for the previous entry to find the difference (which took way too long, about an hour with 200,000 rows), he suggested grouping the data by unique turnstile (about 4000+ of them), sorting by date, then lookup for the previous entry would just be the one before, i.e. O(1). In total that would only be 1 query to get list of unique turnstiles, followed by 4000+ queries for each turnstile, which turned out to be waaay faster (under 5 minutes).

Also, working with Python has been a pleasure, especially when it comes to writing simple assert tests, something I haven't been able to make into a habit with Javascript.

### I have useful data, now what?
Now that we have cleaner data, we can do stuff with it! Some preliminary data. 

__Note: Take it with a grain of salt because I haven't checked for outliers__

#### Top 10 most popular stations for Sept 2015*

Note the repetition in 42 ST station names -- I'm not sure how or why this is, so ideas are welcome. 

I'm assuming that `42 ST-PA BUS TE` is the one that runs on the ACE line and `42 ST-TIMES SQ` is on the 123 line. The subsequent lines are just lines that they are connected to?

    ID        UNIT      STATION               LINENAME              TOTAL_ENTRIES       
    --------  --------  --------------------  --------------------  --------------------
    635141    R011      42 ST-PA BUS TE       ACENQRS1237           1305321095061       
    586808    R080      57 ST-7 AVE           NQR                   1288256345231       
    714660    R084      59 ST-COLUMBUS        1ABCD                 739883225413        
    713284    R033      42 ST-TIMES SQ        1237ACENQRS           656792232455        
    705651    R028      FULTON ST             2345ACJZ              563220526064        
    680062    R453      23 ST-6 AVE           FM                    521178110228        
    729663    R131      23 ST                 6                     517934378933        
    726965    R044      BROOKLYN BRIDGE       456JZ                 509834714918        
    773955    R110      FLATBUSH AVE          25                    496945235519        
    716443    R452      72 ST                 123                   494767882795  

#### All stations on the G line in descending popularity

Yes, I take the G a lot, and kind of like how small and unpopular it is! Look at the ridership difference between `COURT SQ` and `42 ST` above. Seems like there are a lot of people who live in LIC/Astoria that take the G all the way up. I'm surprised that `HOYT` doesn't rank higher because it transfers to the `ACE`, but then again, most people would have gone on elsewhere and transferred without going through the turnstiles. We're losing out on a lot of transfer data here.



    UNIT   STATION     LINENAME  TOTAL_ENTRIES         TOTAL_EXITS         
    -----  ----------  --------  --------------------  --------------------
    R359   COURT SQ    EMG       42686776964           3097964946          
    R256   NASSAU AV   G         30329584142           9830425266          
    R269   BEDFORD/NO  G         28813348373           8496318383          
    R204   CHURCH AVE  FG        28133085368           7156064502          
    R258   4 AVE       DFGMNR    8800979414            6510770868          
    R220   CARROLL ST  FG        7087445681            4230062938          
    R268   METROPOLIT  GL        6618093970            4236728648          
    R129   BERGEN ST   FG        6034371654            3762354812          
    R217   HOYT/SCHER  ACG       5016216558            5133762835          
    R288   7 AV-PARK   FG        4235522401            1337593203          
    R317   CLINTON-WA  G         3303797746            716951611           
    R286   MYRTLE-WIL  G         3230666594            2101013721          
    R241   15 ST-PROS  FG        3159222729            1395058404          
    R299   BROADWAY    G         2432778191            1802209267          
    R287   CLASSON AV  G         2340248666            2287034710          
    R239   GREENPOINT  G         2286162763            1402662188          
    R318   FULTON ST   G         2056953955            2119567171          
    R289   FT HAMILTO  FG        1725898227            590853872           
    R270   SMITH-9 ST  FG        1704099310            1274924271          
    R316   FLUSHING A  G         1180127765            1159380049          
    R360   VAN ALSTON  G         919771530             1042047252

I also have other snippets of data on the [Github README](https://github.com/piratefsh/mta-turnstile-cruncher).

## What's next
Gonna work on extracting more interesting data and a visualizer! 

If you like trains, check out [fun facts from the MTA](http://web.mta.info/nyct/facts/ffsubway.htm) and a previously created [turnstile visualizer](http://chriswhong.com/open-data/visualizing-the-mtas-turnstile-data).

Also, missing the Summer 2s already, but on the flipside we had a huge new batch of Fall 2 RCers join us this Monday! It's already past the halfway point of RC and time feels like it's running out too fast. 

Back to code!

_*From dates of 2015-08-29 to 2015-09-25_