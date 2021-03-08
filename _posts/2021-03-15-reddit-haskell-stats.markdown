---
title: Reddit Haskell Stats
published: false
---

Having posted a lot of blog entries to the [Haskell Subreddit](https://reddit.com/r/haskell) lately with mixed results.
So I wrote a script to gather all `/r/haskell` posts for the last year and visualize them.

It pages the API at `https://api.pushshift.io/reddit/search/submission/`, parses the `created_utc` and `score` fields
from the response, and converts the data to CSV fields including the day of the week and UTC hour of the day.

The script, including the CSV output, is here:
[https://gist.github.com/dfithian/4842b420640c6a793b52cdda012cb391](https://gist.github.com/dfithian/4842b420640c6a793b52cdda012cb391).

Here are the results!

## Hour of day

Volume ramps up over the course of the day. Interestingly, it's highest at 16 UTC, which is around the time most folks
in Europe are leaving work. There's a bump right before bedtime for a lot of folks in North America.

![/assets/volume-by-hour-of-day.png](/assets/volume-by-hour-of-day.png)

The average score by submission hour is interesting. Submissions that land just as Europe is waking up are popular (7
UTC), as are submissions right after they stop working (18 UTC).

![/assets/avg-score-by-hour-of-day.png](/assets/avg-score-by-hour-of-day.png)

## Day of week

Volume is highest at the beginning and end of the week, with the lowest days being Wednesday, Saturday, and Sunday.

![/assets/volume-by-day-of-week.png](/assets/volume-by-day-of-week.png)

Apparently nobody goes on `/r/haskell` on Saturday, which makes sense as people like to do other things over the
weekend.

![/assets/avg-score-by-day-of-week.png](/assets/avg-score-by-day-of-week.png)

## The most opportune time to submit

Based on the above, it seems like the most opportune time to post would be weekdays around 7 UTC, given the low volume
at that time.

_Past returns do not indicate future performance._
