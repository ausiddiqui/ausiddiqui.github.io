---
layout: post
title: "New York City MTA data exploration"
comments: true
description: "NYC MTA provides relevant but improveable urban data"
keywords: "dummy content"
---

The New York City MTA has a number of interesting data sets, including API enabled ones only available through free registration. However,
> the turnstile level data is available for download and use here: http://web.mta.info/developers/turnstile.html

The turnstile data creates a number of interesting challenges in understanding residents of New York and visitors. The data consists of 
> entries and exits out of city subway station turnstiles in the form of counters. The turnstiles are supposed to have a frequency of
> approximately 3 hours between a new count number being recorded. Information about lines are also embedded in relation to the stations
> served, 

### Process and Assumptions ###
The MTA turnstile data was used, primarily analyzing the ENTRIES at the turnstile level. Two time periods were analyzed, the primary being
> the week of January 7 – 13, 2017 and the comparison week of December 3 – 9, 2016. Using pandas dataframes to identify the most
>popular stations and subway lines. The LINENAME column was stripped and separated into multiple columns to separate and identify the 
>relationships between stations and lines. It was assumed that the ENTRIES assigned to a particular turnstile in a particular station which
>carries passengers from different lines are uniformly distributed across different lines. No additional weights were assigned to different 
>lines. As the ENTRIES field is a counter, the difference between two timestamps represents the number of actual entries between those
>occurrences. As part of the data cleaning process when negative values or values greater than 20,000 were found between two timestamps, 
>they were set to zero. In terms of the problem statement, it was assumed that for musicians at the School subway stations would be ideal
>locations to perform, whereas for dancers subway trains themselves would be the best location.

Results
The resulting analysis found that the most popular stations are the those in central
>Manhattan, especially those with a large number of lines. This is consistent between weekend
>and weekdays. A few stations that are very popular would not be ideal for the music students,
>especially if they live nearby the school which is in the Upper Westside of New York – these
>included the stations in Flushing, Jackson Heights or Howard Beach/JFK. In terms of lines, it
>was discovered that generally the stations with more lines tended to carry more passengers.
>The ‘1’ line in particular which is right outside the steps of the school is by far the most popular
line.
