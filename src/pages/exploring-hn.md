# Exploring Hacker News Data

See the source code of this post in [here](https://github.com/adhikasp/eda-with-me/blob/main/src/pages/exploring-hn.md?plain=1)

## About [Hacker News](https://news.ycombinator.com/)

Taken from [Wikipedia](https://en.wikipedia.org/wiki/Hacker_News), 

> Hacker News (sometimes abbreviated as HN) is a social news website focusing on computer science and entrepreneurship. It is run by the investment fund and startup incubator Y Combinator. In general, content that can be submitted is defined as _"anything that gratifies one's intellectual curiosity."_

I found myself check the site almost daily, as frequently the top submission there is interesting to me.

Lately, I am playing around with [TimescaleDB](https://www.timescale.com/) and PostgreSQL, mainly on how to operate them, how to create an effective query, and using them for [OLAP](https://en.wikipedia.org/wiki/Online_analytical_processing) workload. I want to have real dataset and HN data seems to be an interesting one.

## Collecting the data

I am having difficulty finding a ready-to-use and uptodate HN datadump. There was [HN BigQuery dataset](https://bigquery.cloud.google.com/table/fh-bigquery:hackernews.stories), but last updated is 2015. Luckily HN have API, I can just scrape it on my own!

HN API documentation can be found [here](https://github.com/HackerNews/API). It was a memory dump of HN data structure exposed through public Firebase endpoint. To scrape it, I create a Python script that iterate over the API and insert it into PostgreSQL, you can find my script in **[adhikasp/hackernews-scape](https://github.com/adhikasp/hackernews-scape)**.

My initial script is very straightforward, use the Python `request` library to make HTTP call and iterate for all of the items. It goes so slow and the estimated time take weeks! To make the process more efficient, I use [asyncio](https://docs.python.org/3/library/asyncio.html) and [aiohttp](https://docs.aiohttp.org/en/stable/) (an async http client) as I found the HTTP call waiting time is the biggest bottleneck in my process. This make it easy for me to create a parallel fetch worker without the need to dirtying my hand (read: properly understand) with python native multiprocessing code. In the end, collecting the data took about 4 days of non-stop scrapping.

## Usage trend of Hacker News activities over time

```hn_total_submission
SELECT COUNT(*) FROM items
```

```hn_total_stories
SELECT COUNT(*) FROM items
WHERE type = 'story'
```

```hn_total_comments
SELECT COUNT(*) FROM items
WHERE type = 'comment'
```

HN is a pretty lively community. Up until now, there is <Value data={data.hn_total_submission}/> submissions items on HN. This consist of <Value data={data.hn_total_stories}/> stories, <Value data={data.hn_total_comments}/> comments, and other type (poll, job posting, and poll options) . Let's see how the community content grow over time.

```hn_activities_monthly
SELECT date_trunc('month', time) AS date, count(*) AS post_count
FROM items 
WHERE time > '2000-01-01' -- Don't include "deleted" item which have timestamp 1970 by default
GROUP BY date
ORDER BY date ASC
```

<LineChart 
    data={data.hn_activities_monthly}  
    x=date 
    xAxisTitle=Year
    y=post_count
    yAxisTitle="Post Count"
/>

```hn_activities_monthly_grow
select 
    ROUND(AVG(post_count))
from ${hn_activities_monthly} AS subquery
```

```hn_activities_monthly_max
SELECT date, post_count 
from ${hn_activities_monthly} AS subquery
ORDER BY post_count DESC
LIMIT 1
```

When HN was first launched on <Value data={data.hn_activities_monthly} fmt=month/> <Value data={data.hn_activities_monthly} fmt=year/>, there are only <Value data={data.hn_activities_monthly} column="post_count"/> stories and comments. On average, HN content grow by <Value data={data.hn_activities_monthly_grow}/> submissions/month with its recent peak of <Value data={data.hn_activities_monthly_max} column=post_count/> submissions in single month, on <Value data={data.hn_activities_monthly_max} fmt=month/> <Value data={data.hn_activities_monthly_max} fmt=year/>.

## About Hacker News user

```num_of_active_user_last_month
SELECT 
    COUNT(DISTINCT("by")) as active_user_count, 
    date_trunc('month', current_date - interval '1' month) as last_month_date
FROM items
WHERE date_trunc('month', current_date - interval '1' month) <= "time" AND "time" <= date_trunc('month', current_date)
```

To have that number of monthly submission, there must be quite significant number of users too. As of the <Value data={data.num_of_active_user_last_month} fmt=date column=last_month_date/>, there is <Value data={data.num_of_active_user_last_month} column=active_user_count/> users active in the site.

Interestingly, only minor amount of this userbase post a comment or story. Let's see the distribution of users activity, binned according to total submission (post + story) created by them.


```hn_user_activity_dist
SELECT CONCAT(bin::text, '-', (bin * 10)::text, ' submissions') AS bin, qty
FROM (
    SELECT 
        pow(10, floor(ln(post_count) / ln(10))) AS bin,
        count(pow(10, floor(ln(post_count) / ln(10)))) AS qty
    FROM (
        SELECT "by" AS username, COUNT(*) AS post_count 
        FROM items
        WHERE "by" IS NOT NULL AND "by" <> ''
        GROUP BY "by"
    ) AS subquery_1
    GROUP BY
        bin
) AS subquery_2
```

<BarChart 
    data={data.hn_user_activity_dist} 
    x=bin 
    y=qty
/>

This is very similar with [1% rule](https://en.wikipedia.org/wiki/1%25_rule) of thumb that state only 1% of website users create the majority of content. You can see that majority of users only have 1-10 submission (including me :)), and on the other extreme, a small percentage of users have tens of thousand submission. 

```estimated_num_of_user_plus_lurker
SELECT (active_user_count * 10) AS est_count
FROM ${num_of_active_user_last_month} as subquery
```

Please do note that this numbers don't count the lurkers or silent reader, people who only view the content and never post anything. If we take account of that and interpolate based on 1% rule, I estimate that HN have <Value data={data.estimated_num_of_user_plus_lurker} column=est_count/> users.

Also, I am curious about who is power user in HN is, lets take a look:

```top_hn_user
SELECT "by" AS username, COUNT(*) AS post_count 
FROM items 
WHERE "by" IS NOT NULL AND "by" <> ''
GROUP BY "by"
ORDER BY post_count DESC
LIMIT 25
```

<BarChart 
    title="Top 25 HN user by post"
    data={data.top_hn_user} 
    x=username 
    xAxisTitle=Username
    y=post_count
    yAxisTitle="Post Count"
/>

I never be one who really take note of who post something. From that list I only recognize _dang_ as the moderator of HN. He sure do a lot of moderating over the year :)

## Looking at the submission in HN

As I said, there is lots of interesting things posted at HN. The [frontpage of HN](https://news.ycombinator.com/news) is ranked with regard to score (upvote by users) and inverse of time to keep the list fresh. You can see the algorithm in [this post](https://news.ycombinator.com/item?id=1781013) and read a more explanation on how it works [in here](https://medium.com/hacking-and-gonzo/how-hacker-news-ranking-algorithm-works-1d9b0cf2c08d). I am curious on what is the most top voted submission in HN, so let's see it.

## Top 25 Hacker News Submission

```top_hn_submissions
SELECT 
    title, 
    "by" AS author,
     score, 
     concat('https://news.ycombinator.com/item?id=', id) AS url, 
     ROW_NUMBER() OVER (ORDER BY score DESC) 
FROM items
WHERE 
      "by" IS NOT NULL AND "by" <> ''
  AND "type" = 'story'
ORDER BY score DESC
LIMIT 25
```

{#each data.top_hn_submissions as submission}  

{submission.row_number}. [{submission.title}]({submission.url}) by [{submission.author}](https://news.ycombinator.com/user?id={submission.author}) - score {submission.score}  

{/each}

## User with highest accumulated score across stories

Beside top voted content, I am curious do certain people consistently create an interesting content by HN reader standard? We can check it out by ranking the users based on sum of their story scores here.

```top_hn_content_producer
SELECT "by" as author, SUM(score) AS total_score
FROM items
WHERE type = 'story'
GROUP BY "by"
ORDER BY total_score DESC
LIMIT 25
```

<BarChart 
    title="Top 25 HN user by accumulated score"
    data={data.top_hn_content_producer} 
    x=author 
    xAxisTitle=Username
    y=total_score
    yAxisTitle="Accumulated scores"
/>

## Most "unpopular" submission

```top_hn_controversial_submissions
SELECT 
    title, 
    "by" AS author, 
    (descendants/score) AS comment_per_score, 
    concat('https://news.ycombinator.com/item?id=', id) AS url, 
    ROW_NUMBER() OVER (ORDER BY (descendants/score) DESC, descendants DESC) 
FROM items
WHERE
      "by" IS NOT NULL AND "by" <> ''
  AND "type" = 'story'
  AND score > 0 AND descendants > 0
  AND (descendants/score) > 0
  AND NOT dead
ORDER BY (descendants/score) DESC, descendants DESC
LIMIT 5
```

Looking at the good stuff above, I also wonder, what is the bad stuff posted in HN? Looks like lot of straightforward rule-breaking or offending submission is deleted by moderator, marked as `deleted` in the database. There is no score attached so I don't find a good way to surface the significant one among them. But, I can try to find "unpopular" content, which I define as submission that is not deleted, have relatively low score, but attracting lots of comment. I feel this mean people want to talk about it but it is not necessarly good as they are not upvoting it (or lot of upvote vs downvote) happen.

{#each data.top_hn_controversial_submissions as submission}  

{submission.row_number}. [{submission.title}]({submission.url}) by [{submission.author}](https://news.ycombinator.com/user?id={submission.author}) - comment/score ratio {submission.comment_per_score}  

{/each}

## Activity pattern in HN

<ECharts config={activityHeatmapOption}/>

The time above is in UTC. [The majority of HN users lives in North America and next is Europa](https://news.ycombinator.com/item?id=30210378) and the bulk of activities looks like happening on the daytime in America, on the workday... Hmm I guess HN often used as proscatination tools at work time :)

## What's next?

There is lots of interesting things I can do that I thought upon while working on this. Here are the things that I will explore in future post.

- I found some interesting insight trying to optimize PostgreSQL for analytics/OLAP queries. Sometime index can help so much, sometime [Continuous Aggregate](https://docs.timescale.com/timescaledb/latest/how-to-guides/continuous-aggregates/about-continuous-aggregates/) query is slower than query against the base table!

- Finding the "formula" of what makes a submission is scored highly in HN?

- Explore creating a recommendation system based on content that I have read and like in HN. Then it can recommend me similar stuff deep inside the HN database.

- I want to improve the scrapper code to always updating data, and maybe using Firebase SDK as the HN API docs recommend it for more efficient network usage. 

- I want to share this PostgreSQL datadump over torrent so other can use it without scrapping API.

- I want to create frontend implementation of HN with my own DB as the backing. 

- Create a "distributed HN" with [ActivityPub](https://www.w3.org/TR/activitypub/). New independent services can populate their database not via the original HN API, but via swarm of people that already have data (like my data dump)


<script>
const hours = [
    '12a', '1a', '2a', '3a', '4a', '5a', '6a',
    '7a', '8a', '9a', '10a', '11a',
    '12p', '1p', '2p', '3p', '4p', '5p',
    '6p', '7p', '8p', '9p', '10p', '11p'
];
// prettier-ignore
const days = [
    'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday', 
];

// SELECT json_agg(json_build_array("hour","weekday","post_count")) FROM (
// SELECT 
//     extract(hour from time) as hour,
//     (extract(isodow from time) - 1) as "weekday",
//     COUNT(*) as post_count
// FROM items 
// WHERE time > '2022-02-01' 
// GROUP BY (extract(isodow from time) - 1), extract(hour from time)
// ORDER by (extract(isodow from time) - 1) ASC, extract(hour from time) ASC
// ) AS subquery

const data_point = [[0, 0, 2990], [1, 0, 3211], [2, 0, 2863], [3, 0, 2681], [4, 0, 2484], [5, 0, 2385], [6, 0, 2266], [7, 0, 2050], [8, 0, 2080], [9, 0, 2048], [10, 0, 1845], [11, 0, 1616], [12, 0, 1458], [13, 0, 1344], [14, 0, 1860], [15, 0, 1926], [16, 0, 1988], [17, 0, 2016], [18, 0, 2209], [19, 0, 2576], [20, 0, 2981], [21, 0, 3467], [22, 0, 4034], [23, 0, 4088], [0, 1, 5485], [1, 1, 5244], [2, 1, 4695], [3, 1, 4538], [4, 1, 4421], [5, 1, 4191], [6, 1, 3728], [7, 1, 3142], [8, 1, 2691], [9, 1, 2438], [10, 1, 2270], [11, 1, 2108], [12, 1, 1951], [13, 1, 1882], [14, 1, 2122], [15, 1, 2263], [16, 1, 2480], [17, 1, 2448], [18, 1, 2468], [19, 1, 2764], [20, 1, 3622], [21, 1, 4518], [22, 1, 5015], [23, 1, 5823], [0, 2, 5600], [1, 2, 5777], [2, 2, 5712], [3, 2, 5044], [4, 2, 4778], [5, 2, 4120], [6, 2, 3539], [7, 2, 3215], [8, 2, 2692], [9, 2, 2502], [10, 2, 2264], [11, 2, 2300], [12, 2, 2014], [13, 2, 1847], [14, 2, 2042], [15, 2, 2175], [16, 2, 2175], [17, 2, 2208], [18, 2, 2183], [19, 2, 2649], [20, 2, 3366], [21, 2, 4483], [22, 2, 4858], [23, 2, 4945], [0, 3, 5456], [1, 3, 5264], [2, 3, 4754], [3, 3, 4551], [4, 3, 4347], [5, 3, 4263], [6, 3, 3868], [7, 3, 3185], [8, 3, 2699], [9, 3, 2343], [10, 3, 2293], [11, 3, 2042], [12, 3, 2125], [13, 3, 1938], [14, 3, 2235], [15, 3, 2492], [16, 3, 2388], [17, 3, 2223], [18, 3, 2271], [19, 3, 2667], [20, 3, 3616], [21, 3, 4610], [22, 3, 4906], [23, 3, 5556], [0, 4, 5476], [1, 4, 5028], [2, 4, 4759], [3, 4, 4612], [4, 4, 4406], [5, 4, 4003], [6, 4, 3548], [7, 4, 3052], [8, 4, 2577], [9, 4, 2405], [10, 4, 2339], [11, 4, 2114], [12, 4, 1993], [13, 4, 1911], [14, 4, 2220], [15, 4, 2436], [16, 4, 2320], [17, 4, 2425], [18, 4, 2363], [19, 4, 3047], [20, 4, 3739], [21, 4, 4520], [22, 4, 4752], [23, 4, 4994], [0, 5, 5193], [1, 5, 5131], [2, 5, 4571], [3, 5, 4521], [4, 5, 3869], [5, 5, 3685], [6, 5, 3183], [7, 5, 2686], [8, 5, 2192], [9, 5, 2010], [10, 5, 1972], [11, 5, 1715], [12, 5, 1717], [13, 5, 1536], [14, 5, 1744], [15, 5, 1922], [16, 5, 1754], [17, 5, 1632], [18, 5, 1793], [19, 5, 1982], [20, 5, 2490], [21, 5, 2794], [22, 5, 2912], [23, 5, 3058], [0, 6, 3090], [1, 6, 3179], [2, 6, 2927], [3, 6, 2995], [4, 6, 2697], [5, 6, 2225], [6, 6, 2485], [7, 6, 2318], [8, 6, 1947], [9, 6, 1752], [10, 6, 1673], [11, 6, 1364], [12, 6, 1284], [13, 6, 1178], [14, 6, 1239], [15, 6, 1279], [16, 6, 1430], [17, 6, 1403], [18, 6, 1565], [19, 6, 1788], [20, 6, 1947], [21, 6, 2185], [22, 6, 2590], [23, 6, 2813]]

var activityHeatmapOption = {
  tooltip: {
    position: 'top'
  },
  grid: {
    height: '50%',
    top: '10%'
  },
  xAxis: {
    type: 'category',
    data: hours,
    splitArea: {
      show: true
    }
  },
  yAxis: {
    type: 'category',
    data: days,
    splitArea: {
      show: true
    }
  },
  visualMap: {
    min: 0,
    max: 6000,
    calculable: true,
    orient: 'horizontal',
    left: 'center',
    bottom: '15%'
  },
  series: [
    {
      name: 'Number of submission',
      type: 'heatmap',
      data: data_point,
      label: {
        show: false
      },
      emphasis: {
        itemStyle: {
          shadowBlur: 10,
          shadowColor: 'rgba(0, 0, 0, 0.5)'
        }
      }
    }
  ]
};
</script>
