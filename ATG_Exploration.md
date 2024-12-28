# ATG Exploration

Objective: Identify local ATG (Automatic Tank Gages) to locate a nearby gas station that has gas.

In Context, I started this project right after a bad storm caused a run on gas stations, only half of which were open anyway due to widespread power outages. Approximately a tenth of the gas stations had power, devestating supply while demand grew, since people now needed their normal gas and gas for generators.

With such an instane little apocolypse going on, I decided to try and use this as an oppurtunity to do "Hacker Stuffs" in a helpful way. Hence the invention of this project, which aimed to build a process for identifying local ATGs (Automatic Tank Gages). ATGs are often internet facing (unfortunately), and provide controls as well as status notes for gas stations. Now, for the purposes of this project, I only cared about the status signals these systems give off, not the possible control mechanims (curious adventurer, not black hat hacker).

So with that noted, let's get started.

## Step 1: Tools and Research

The Tool for this sort of recon is no doubt, Shodan. I, somehow, didn't already have a Shodan Membership, so that got fixed.

> ### [Shodan Search Engine](shodan.io)
> Shodan is a search engine for finding specific devices that are connected to the internet. It's common to see it used to search for exposed webcams, routers, ICS systems, and in our case, ATGs.
> 
> If you haven't payed with it before, I highly recommend setting up a free acount and spending some time in the web interface. It can be an enlightening adventure. However, do use caution: there is no filtering on adult content, and many things (especially the search for "webcam") may be more then you asked for.

Now, for this kind of analysis I definitely needed to be able to play around and have some flexibility, so I opted to use the Shodan API instead of the web interface.

>### API Installation and Basic Usage
> ### Installation (Ubuntu)
> 1) `sudo apt get install python3`
> 
> 2) `sudo apt get install python3-shodan`
>
> ### Basic Usage
>
> `shodan count <search>` - Get Count of results from search
>
> `shodan download [--limit <limit or -1>] <results File Name> <search>` - Do a search, download the results (up to limit, or -1 for no limit)
>
> `shodan convert <File.json.gz> <new type>` - Convert default archived json format into various other formats (i.e. CSV, geo.json, images, xlsx)
>
> **For more information, look at Shodan's [Developer Site](https://developer.shodan.io/)**

Now, For ATGs, there's a few peices of information that could be useful. To start this, I used a query from [jakejarvis on Github](https://github.com/jakejarvis), as part of his [awesome-shodan-queries repository](https://github.com/jakejarvis/awesome-shodan-queries):

```
"in-tank inventory" port:10001
```

From here, I identified the most common port for this activity: 10001. I was also able to take a look at the first set of results and draw out commonalities. One example of an ATG response:

```
I20100
JUN 20, 2023 10:12 PM

ROMEOVILLE BP 202   
418 N WEBER         
ROMEOVILLE,IL       
                    

IN-TANK INVENTORY       

TANK PRODUCT             VOLUME TC VOLUME   ULLAGE   HEIGHT    WATER     TEMP
  1  RUL EAST              8540      8508     1155    74.09     0.00    65.33
  2  RUL WEST              8004      7971     1691    69.96     0.00    65.75
  3  GOLD                  5628      5608     4067    51.37     0.00    64.95
  4  DIESEL                5069      5051      828    72.62     0.00    67.83
```

With these formats, I noted a general outline:

```
I20100
<date>

<station details>
<station Address>

IN-TANK INVENTORY

TANK PRODUCT             VOLUME TC VOLUME   ULLAGE   HEIGHT    WATER     TEMP
<data data data>
```

This gave me some new parameters to search by:
- `I20100`- A protocol indicator used by the ATG standard, acting as a standard "Status" request
- `IN-TANK INVENTORY`
- Station Address- Allows me to filter geographically, by city, state, or zip pretty easily
- Station Details- Name of station, allowing me to search by known brands locally, in case their address isn't present

## Step 2: Search and Parse

### Search 1

```bash
OK port:10001 country:US
```
*Result Count: 8559*

*Saved Count: 5615*

This first search was VERY noisy, more items than predicted communicate on port 10001. This included oracle databases, wireless cameras, and several honeypots (which were self-identified honeypots, not that that makes much sense...).

Needless to say, I didn't get far parsing this one. I gave up and moved onto search 2 pretty quickly.

### Search 2

```
tank port:10001 country:US
```
*Result Count: 4074*

*Saved Count: 2745*

This search resulted in more Tank readings than not, which was a positive outcome. However, these search results weren't complete- according to my console, I downloaded 2745 results of the 4074 results that were available. I double checked my first search, finding a similar lack of full results. Upon investigating further, I also found there is a significant lack of documentation about this issue- to the point I couldn't find anything on how to change my search to return more values.

This search was missing known data points I had seen previously, so I knew I didn't have the complete dataset, so this was also fairly useless. I'll dive more into this later, spoiler alert.

### Search 3

```
i20100 port:10001 country:US
```

*Result Count: 4800*

*Saved Count: 946*


This search returned the same problems as before. So, I attempted to run it via the web interface this time instead, and it actually made the problem even worse. This time I downloaded about one fifth of the data. At this point in the process, I'm pivoting to look at what's actually happening behind the scenes to cause this problem.

## Analysis

### "Fewer saved then requested"

```python
    # A limit of -1 means that we should download all the data
    if limit <= 0:
        limit = total

    with helpers.open_file(filename, 'w') as fout:
        count = 0
        try:
            cursor = api.search_cursor(query, minify=False, fields=fields)
            with click.progressbar(cursor, length=limit) as bar:
                for banner in bar:
                    helpers.write_banner(fout, banner)
                    count += 1

                    if count >= limit:
                        break
        except Exception: # This is my problem
            pass

        # Let the user know we're done
        if count < limit:
            click.echo(click.style('Notice: fewer results were saved than requested', 'yellow'))
        click.echo(click.style(u'Saved {} results into file {}'.format(count, filename), 'green'))
```
> Source code pulled from the [Shodan-Python Github, \_\_main\_\_.py](https://github.com/achillean/shodan-python/blob/master/shodan/__main__.py)

The lack of results being saved to my device began to get irritating, so I moved away from searching for a minute to check why this was occuring. After some investigating, I found there was zero documentation on this particular error, so I had to instead go to the source code to see if I could deduce the issue from there.

A quick search for the notice, "Notice: fewer results were saved than requested", found the above section of code. The for loop nested into this code appears to be the file saving loop. As we can see, if this loop ever fails, it does so silently, throwing an exception which it simply passes through. With this in mind, I theorize that at some point in the loop, something throws an exception and ends the search early. But now we have a new problem: What throws the loop?

And naturally, I had to be wrong on the first try. I edited the local version of the shodan downloader to print the exception instead of running the non-action `pass` in the except flow. This resulted in no new output. (Just to check I was doing this right, I put a similar print statement outside the except flow and made sure that showed up, which it did)

So if the exception isn't the cause, what is? My best guess was a mismatch between Shodan's results count and actual downloading mechanisms. Somewhere, in the collection that the for loop runs around, is a mismatch of length- it thinks it contains oh so many items, but upon iteration it contains less. Bizzare, and unforunately, nothing I can fix. So for now, this will have to reamin a mystery.

## Conclusion

I unfortunately was unable to achieve my objective this round, but I was able to glean some information from this adventure. For one, I now have new experience in Shodan, both with the web interface and it's CLI counterpart. I also have access to the Shodan API, which I can use in future adventures.

Another take-away from this is an understanding of ATGs and their widespread exposure. Even if I couldn't see every data point, there were way too many exposed ATGs in Shodan, especially since they are a part of critical infrastructure. This feels like a pretty serious gap in security (that admittedly several people have written about, including [Rapid7](https://www.rapid7.com/blog/post/2015/01/22/the-internet-of-gas-station-tank-gauges/)- even [*twice*](https://www.rapid7.com/blog/post/2015/11/18/the-internet-of-gas-station-tank-gauges-take-2/)).

Finally, one last take-away was an adventure in trying to read another developer's python code. Thankfully, the shodan main method is pretty straight-forward, and although a lot of the work is hidden behind helper objects and api calls, the general structure can still be understood with a little elbow grease.