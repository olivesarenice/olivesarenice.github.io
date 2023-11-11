---
title: Real Estate Tableau Dashboards
tag: [tableau, data-analysis]
category: projects
---

## Summary

This writeup will cover how I set up a suite of dynamic private Tableau dashboards embedded in a public-facing Wordpress site. It will not dive deep into the design of the dashboard as that was specific to the analyses needed for this instance.

We will walkthrough each step:

0. Purpose
1. Data source
2. Dashboard design
3. Custom Tableau functions
4. Data extract scheduling
5. Private source - Public usage configuration

## Purpose

I was contacted to create a set of real estate market dashboards that could be used by real estate agents and the general public to trend and analyse various real estate transactions and price indices. The goal was to have a single website that hosted multiple dashboards related to real estate and macroecomonic trends, sort of like [this site](https://intothecryptoverse.com/)

{% include img-wrap name="crypto.png" caption="Into The Cryptoverse user page" %}

There were 3 requirements:
1. The first set of dashboards had to be ready in 1 month
2. All dashboards must have the ability to auto-calculate and display the % change between 2 selected points on the graph, and draw a line across the points
3. The dashboards must be hosted within the company's existing Wordpress site, meaning dashboards will need to be embedded directly.

The reason why I chose to stick with Tableau was because the company already had a Tableau Online (cloud) license. Tableau also had the option for storing the data privately, and only allowing public access via JWT, thus restricting the usage of the dashboards to strictly within the website.

I also considered other options such as Looker Studio (Google), but ultimately the E2E functionality of Tableau, including data loading and dashboard deployment options, came out on top.


## Data source/ collection

The 13 dashboards that were created shared 2 main types of data sources:
- Transaction data - from Singapore's REALIS database and HDB database
- Macroeconomic data - from various national websites

All data is stored in an AWS Redshift database.

**REALIS transaction** data was downloaded and cleaned using a Python script which did the following:
1. Auto login to REALIS website
2. User manually triggers SingPass MFA (unavoidable)
3. Continue auto-navigate to data table to enter query date filters based on latest entry in Redshift database.
4. Download the data files
5. Pre-process data - re-format/ re-type columns, add new labels/ columns, perform any pre-calculations needed in analysis later
6. Upload clean data to Redshift

{% include img-wrap name="realis.png" caption="REALIS database" %}

**HDB transaction** data was downloaded directly from [data.gov.sg](https://beta.data.gov.sg/collections/189/view) and similarly pre-processed and delta-loaded.

**Macroeconomic data** was collected from a variety of national databases like [FRED](https://fred.stlouisfed.org), [OECD](https://data.oecd.org), and [Singapore Data Portal](https://data.gov.sg/)

For the purposes of the dashboards, it was easier to store all macro data (time-series) in long format in a single table, with the `series_name` having its own column. 
Their metadata was tagged accordingly in a separate table which was used to join them later on in Tableau.

Metadata:

| series_id | series_name                        | country | indicator_type | series_segment | description                                                     | data_frequency | official_source                                                                                      |
|----------|-----------------------------------|---------|----------------|----------------|-----------------------------------------------------------------|----------------|------------------------------------------------------------------------------------------------------|
| 1        | macro_us_re_index_all             | us      | index          | All            | FHFA House Price Index (HPI)                                    | q              | [FHFA House Price Index Datasets](https://www.fhfa.gov/DataTools/Downloads/Pages/House-Price-Index-Datasets.aspx#qpo)                   |
| 2        | macro_us_re_index_resale          | us      | index          | Resale         | Case-Shiller U.S. National Home Price Index (CSUSHPINSA)        | m              | [Case-Shiller U.S. National Home Price Index](https://fred.stlouisfed.org/series/CSUSHPINSA)  |
| 3        | macro_us_re_index_new             | us      | index          | New            | New Single-Family Houses Sold Price Index                      | q              | [New Single-Family Houses Sold Price Index](https://www.census.gov/construction/cpi/)         |
| 4        | macro_us_stock_sp500              | us      | stock          |                | S&P 500 Index                                                  | m              | [S&P 500 Index Historical Data](https://www.investing.com/indices/us-spx-500-historical-data) |
| 5        | macro_us_fedfunds                 | us      | interest       |                | Federal Funds Effective Rate (FEDFUNDS)                        | m              | [Federal Funds Effective Rate](https://fred.stlouisfed.org/series/FEDFUNDS)                  |
| 6        | macro_us_cpi                      | us      | inflation      |                | Consumer Price Index for All Urban Consumers: All Items in U.S. City Average (CPIAUCSL) | m              | [Consumer Price Index for All Urban Consumers](https://fred.stlouisfed.org/graph/?g=sL9U)  |


Time-series:

| series_name              | date       | value  |
|--------------------------|------------|--------|
| macro_us_re_index_all   | 2021-04-01 | 335.69 |
| macro_us_re_index_all   | 2021-07-01 | 349.50 |
| macro_us_re_index_all   | 2021-10-01 | 357.98 |
| macro_us_re_index_all   | 2022-01-01 | 374.73 |
| macro_us_re_index_all   | 2022-04-01 | 395.01 |
| macro_us_re_index_resale | 1995-01-01 | 80.03  |
| macro_us_re_index_resale | 1995-02-01 | 79.99  |
| macro_us_re_index_resale | 1995-03-01 | 80.08  |
| macro_us_re_index_resale | 1995-04-01 | 80.37  |

## Dashboard design

Not much to cover here. Design of the dashboards used basic aggregation over record-level transaction data. The dashboards fall into 3 types:

**Aggregation of price/ volume of transactions based on different categories to pivot and filter the data - by property type, region, sales type, etc.**

{% include img-wrap name="by-region.png" caption="Price-volume by Geographical Region" %}

{% include img-wrap name="by-prop.png" caption="Price-volume by Property Type" %}

**Scatter plot of each transaction's price over a timeline**

{% include img-wrap name="txns.png" caption="Transaction prices of 2 similar projects (different coloured)" %}

**Basic time-series comparison of various macroeconomic trends by quarter**

{% include img-wrap name="indices.png" caption="SG Real Estate Price Index vs. US and SG Stock Index" %}


## Custom Tableau functions

The core requirement for these dashboards was that the user could click on any 2 points and it would return the % change and draw a line to connect the points.

This was not doable in other platforms, and a custom coded feature in Wordpress would not be able to interact with the dashboards which were embedded. Hence, this had to be improvised in Tableau itself.

Additionally, since the dashboards allowed users to select data from periods that could be a few months, all the way to decades, it was important to have a date axis that scaled automatically, switching between months, quarters, and years, so that the graphs are not too sparse when a short period was chosen, and not overcrowded when a long period was chosen.

*None of these functions are new, I just found good discussions on Tableau forums and implemented them in my dashboard.*

{% include img-wrap name="calc-gif.gif" caption="Demo of the line and calculator functions" %}

{% include img-wrap name="dynamic-gif.gif" caption="Demo of dynamic date period" %}

### % Calculator

This can be achieved by using *Parameters* whose values are set by dashboard *Actions*.

1. Define 2 parameters: [Start Value] and [End Value]
2. Define 2 actions on each dashboard you want this functionality on
    - Select Start: Upon selecting a point, this will change [Start Value] to that point's value. When selection is 'cleared' i.e. point is deselected, reset the value to 0. This lets us clear our current calculation/line.
    - Hover End: Upon hovering over a point, this will change [End Value] to that point's value. When selection is 'cleared' i.e. no longer hovering over that point, retain the value of the latest point. This lets us preserve the calculation/line even when the cursor is elsewhere.
    With this combination of actions, we now have a way to dynamically define the 2 points for calculations.
3. Create a calculation on these 2 parameters. In this case I used the % difference between the 2 points.
4. Place the calculator results in another sheet so that it can be used inside the dashboard.
5. Place the calculator sheet into the dashboard. It will now dynamically calculate.

{% include img-wrap name="parameters.png" caption="Define parameters" %}
{% include img-wrap name="actions.png" caption="Actions in Dashboard" %}
{% include img-wrap name="select-start.png" caption="Select Start" %}
{% include img-wrap name="hover-end.png" caption="Hover End" %}


### Draw line

This builds on the calculator method of using parameters and actions. The drawn line is basically a second measure that is overlayed over the main measure. 

To draw a line, we need the (X1,Y1) and (X2, Y2) and connect the 2 points. We know Y1 and Y2 which are the [Start Value] and [End Value] from previously. In this case, our X1 and X2 are the [Start Date] and [End Date], which are 2 new parameters we need to define, exactly the same as how we defined [Start Value] and [End Value], except the value taken is the date instead.

Now we can define a new measure called `[LINE]` which creates 2 data points when the appropriate points have been selected. The `[Start Date]!=''` is there so that the line is not drawn when the starting point is not selected. Recall that we set Action `[Start Date] = ''` when the selection is cleared.

    IF [Dynamic Date Period] = [Start Date] AND [Start Date]!='' THEN [Start PSF ($)] 
    ELSEIF [Dynamic Date Period] = [End Date] AND [Start Date]!='' THEN [End PSF ($)]
    ELSE NULL
    END

When set as a second measure in existing price vs. date graphs, this translates to a data table of:

| Dynamic Date Period              | LINE     |
|--------------------------|------------|
| [Start Date]   | [Start PSF ($)]  |
| [End Date]  | [End PSF ($)] |

Tableau then draws a line across these points. Note that you have to set `LINE` as a dual-axis to the main graph, turn on Synchronize Axes, and Format `LINE` to always connect the points (see below)

{% include img-wrap name="connect-points.png" caption="Force points to connect" %}

Since calculations are happening constantly, everytime the user mouses over a point, this makes the dashboard somewhat laggy, but it isn't so bad as to compromise user experience.

### Dynamic date period

This creates a measure that automatically changes its values (Year, Qtr, Month) depending on the current date range of the filtered data. We will use this new measure as the X-axis inplace of the original datestamps.

We define a new measure called [Dynamic Date Period] with the following calculation, where `[Sale Date]` is the original datestamp of a record:

    IF {MAX([Sale Date])} - {MIN([Sale Date])} < (365*3) 
        THEN str(year([Sale Date])) + "-" + IF MONTH([Sale Date])<10 THEN "0" ELSE "" END+STR(MONTH([Sale Date]))
    ELSEIF {MAX([Sale Date])} - {MIN([Sale Date])} < (365*7) 
        THEN str(year([Sale Date])) + " Q" + str(DATENAME('quarter', [Sale Date])) 
    ELSE 
        str(year([Sale Date]))
    END

This calculation checks the difference between `MAX` and `MIN` dates and adjusts the resulting date value for that record. Thus, if the period is:
- Less than 3 years, aggregate data by month
- Between 3 - 7 years, aggregate data by quarter
- More than 7 years, aggregate data by year

The rest of the functions are there to properly format the date strings so that they will be displayed in proper order on the axis.

Lastly, [Dynamic Date Period] is set as the x-axis, and the original date stamps [Sale Date] is set as the filter (since these are the actual dates). 

**IMPORTANT !!!:** You need to **add the filter to context** so that the filtered values get used in [Dynamic Date Period]

{% include img-wrap name="add-context.png" caption="Add filter to Context" %}

## Data extract scheduling

A key feature of Tableau is the ability to use data extracts which run much, much faster than querying the database, even if the data is stored locally. For dashboards that run on Tableau Online (no compute scalability), having to rely on adhoc queries for each user interaction would be disastrously slow. Hence, data extracts are a must for this application. Tableau allows the scheduling of extracts at any frequency. In this case, I used a daily refresh that occurs overnight to avoid disruption to users in the day.

Here's a [quick guide](https://help.tableau.com/current/online/en-us/schedule_add.htm) to extract scheduling.

### Side note on Tableau's columnar file format:
This [post](https://www.tableau.com/blog/understanding-tableau-data-extracts ) covers the details of Tableau data extracts (TDE). In summary, TDEs load much faster than regular query by storing each column as a file. In Tableau, all calculations rely on aggregations which use the GROUP BY command. In doing so, Tableau needs to access each row by index, read the column name to get the column's value, and repeat, to aggregate them. With colunnar files, Tableau needs to access the required column only once, then read straight down the column to aggregate.

## Private source - Public usage configuration

Lastly, we need a way to expose the dashboards stored in the company's private Tableau cloud, to public users. This will be through `iframe` embedding. Tableau makes this easy by simply passing the dashboard share link to the `iframe` object in the webpage's HTML like so:

    <iframe src="YOUR_DASHBOARD_EMBED_LINK">
    </iframe>

However, the complication is that private dashboards require a registered Tableau user to sign in before viewing it. Since we are on Tableau Cloud, we can't be giving out our usernames to the public, so there must be another way to authenticate user within the context of the website.

{% include img-wrap name="blocked.png" caption="What the user would see without authentication" %}

This is where JWT authentication comes in. Tableau has a feature called ['Connected Apps'](https://help.tableau.com/current/server/en-us/connected_apps.htm) which allows an application to be authenticated without any user sign in. So in this case, Wordpress (one of it's plugins) is the external Connected App which will supply a JWT to the dashboard webpage. This JWT is created from secret and keys hardcoded in Wordpress during the initial setup of the Connected App in Tableau. The webpage sends requests for the `iframe` content from Tableau along with the JWT, so Tableau knows that the request sender is trusted.

{% include img-wrap name="connectedapp.png" caption="Connected App structure" %}

And so, all users loading the dashboards through the site can now view it without having to log into Tableau, and the company keeps the dashboard contents strictly within the website context.

*Note that Wordpress does **NOT** have a built in JWT generator for Tableau's embedding API, you will have to write your own*

Specific details on embedding can be found [here](https://help.tableau.com/current/api/embedding_api/en-us/docs/embedding_api_auth.html).

## Closing

With that, we put a gallery of dashboards which each have their own webpage and loaded in `iframe` containers for the public to view.

{% include img-wrap name="embedded.png" caption="What the user sees on the site" %}

This is my first time building a public product, and it has not been formally launched. But I am interested to see the user feedback in terms of UI/ UX. It was a rush job to design everything, including the dashboard analysis, formatting, captions, etc in 1 month, but it taught me a lot about the various improvisations and hacks that we can use with Tableau to get unconventional functions. 

More importantly, I also learned how to deploy dashboards via cloud while maintaining security with proper authentication in the public domain.

<br>
<br>

*That's all for tonight, ciao.*