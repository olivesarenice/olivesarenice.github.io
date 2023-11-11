---
title: BlueSG Station Availability Predictions
tag: [web-scraping, pi]
category: data analysis
---

## Summary

I attempted to predict the availability of BlueSG (BSG) cars based on past availability of cars and parking spaces displayed on the [BlueSG live map](https://www.bluesg.com.sg/stations-map). 

{% include img-wrap name="1-bsg-map.png" caption="BlueSG Live Map" %}

This writeup covers how I:
1. Found a way to extract the raw JSON from the live map
2. Collected and stored the data in a RaspberryPi every 5 minutes.
3. Tested various machine learning models on it.

Spoiler: I didn't manage to find a model which performed well enough to predict at the granularity that was needed for the use-case to be useful. I was also running into storage and write issues on the Pi, and I had to stop scraping after a few weeks. People normally don't post incomplete/ failed projects, but I thought I got pretty far, especially in the data collection and analysis, so I thought I'd just post this anyway. Maybe someone can continue where I stopped.

## Motivation

### My pain point

As a frequent (and somewhat dependent) user of BSG car sharing services, I often have trouble using their services because there are seldom cars available for rent at my location.

I frequently have to decide between waiting for a car to be available at my nearest station, booking an available one much further away, or just giving up and taking a cab instead. Often, this happens when I really need to get somewhere fast, and BSG cars can be unavailable for hours, making my wait futile. 

**What if I could know if a BSG car is likely to be available at the time that I need it?**

Of course, BSG with a full view of their data can model this easily, but users don't have access to this data. 
This project was borne out of frustration trying to reserve a BSG car on what is a highly opaque booking system. 

{% include img-wrap name="2-none-available.png" caption="Example of station with no available cars" %}

**The goal is to understand car availability trends to make usage less uncertain.**


### More background

BlueSG is an electric car sharing service much like bike sharing but with more constraints. 

The difference is that car rentals have to start at one charging station and end at another station. Each of the 382 stations around Singapore can has enough space for 2 to 4 cars, depending on the station. 

Users pay a subscription fee to have access to the service and pay for rentals on a per-minute basis. Users can reserve a car for up to 30 mins. Within this 30 mins, they have to start the rental at the station. They can also reserve a parking slot at the destination for up to 45 minutes. Reservations can be cancelled at any time. 

If there are no cars/ slots available, the user can queue to be the next reservation in line once a car becomes available. BlueSG charges $0.42/minute of rental, on top of a monthly $8 subscription fee to use the service.

## Data Collection

The first step of any model is compilation of a quality dataset. Since there are no existing data sources, I had to collect my own time-series data over a period.

### BlueSG Live Map

As a customer-facing website, BSG rightly only displays the current availability of their stations on their live map. Historical data is unavailable for the public.

The live map is equally difficult to interact with because the UI forces the user to click on individual stations before revealing the station's current status. However, I knew that the site was loading all data and selectively displaying it, rather than loading them individually.

### Easy tokens

If we take a look at the source code of the site, we can see there are 2 API calls to the `api.bluesg.com.sg` domain.
1. `/oauth2/token` - this passes a generic bearer string that is used in OAuth to generate an API access token. I checked on different browsers and networks and found that they use a fixed bearer - I assume this is convenient for all public users accessing the map. Technically, since the same bearer string is used everytime, we can just hardcode the access token too.

Side note: the bearer `d2ViX3N0YXRpb25zX21hcDp3ZWJfc3RhdGlvbl9tYXA=` is base64 which when decoded is just `web_stations_map:web_station_map`. Very convenient username and password.

{% include img-wrap name="3-oauth-1.png" caption="Authorisation API" %}
{% include img-wrap name="3-oauth-2.png"  %}
{% include img-wrap name="3-oauth-3.png"  %}

2. `/v2/station/?filter=cars` - this returns all station data as JSON, which is later used by the map to interactively load details when the user clicks on the map. It requires an `access-token` to be passed, the same token from the previous call.

{% include img-wrap name="4-stations-1.png" caption="Station API" %}
{% include img-wrap name="4-stations-2.png"  %}
{% include img-wrap name="4-stations-3.png"  %}

Thus, the data scraping is straightforward. Simply replicate the OAuth call with the fixed bearer, then pass the resulting `access-token` to the `station` call to get the current station data.

Thought: if they left the API `access-token` exposed, does that mean that the other API endpoints, whatever they are, can be accessed by the public too? I haven't had the time to test it out yet, but we could potentially enumerate some endpoints and see if they can be accessed.

### Data clean and write

Because the raw JSON contained lots of unneeded details, I also did some processing to extract only the columns needed:
`# of cars` and `# of slots` and also tagged each row with the `station` they belonged to, and the current timestamp. 

{% include img-wrap name="5-original-df.png" caption="Raw data"   %}
{% include img-wrap name="6-clean-df.png" caption="Cleaned data"   %}

Lastly, to reduce storage even further, I used integer `id` in place of the actual station name. The mapping of each `id` to the station details were stored in a separate table and only joined when needed for a query.

    timestamp | station_id | cars | slots

The data was then inserted into `SQL`.

I put the above functions into a Python script and then left it to run.

```python


    import requests
    import pandas as pd
    import datetime

    # IMPORT THE SQALCHEMY LIBRARY's CREATE_ENGINE METHOD
    from sqlalchemy import create_engine
    import pymysql
    from sqlalchemy import text

    def getRawData():
        current_time = datetime.datetime.now()

        token_url = "https://api.bluesg.com.sg/oauth2/token/"
        headers = {
            "Authorization": "Basic d2ViX3N0YXRpb25zX21hcDp3ZWJfc3RhdGlvbl9tYXA=",
            "Content-Type": "application/x-www-form-urlencoded",
            "Origin": "https://membership.bluesg.com.sg"
        }

        body = {"grant_type" : "client_credentials"}

        token = requests.post(token_url, headers = headers, data=body)
        token = token.json()['access_token']

        req_url = "https://api.bluesg.com.sg/v2/station/?filter=cars"
        req_head = {
                    "Authorization" : "bearer " + token,
                    "Origin": "https://membership.bluesg.com.sg"    
                }

        req_body = {"mode":"cors"
                    }
        results = requests.get(req_url, headers = req_head, data = req_body)

        results = results.json()['results']
        raw_df  = pd.DataFrame(results)
        keep_cols = ['id','status','cars', 'slots','cars_counter']
        cl_df = raw_df[keep_cols].copy()
        cl_df = pd.concat([cl_df,cl_df['cars_counter'].apply(pd.Series)], axis=1)
        cl_df.drop(['cars_counter'],axis=1,inplace=True)
        cl_df['timestamp'] = current_time
        
        return cl_df

    if __name__ == '__main__':
    # DEFINE THE DATABASE CREDENTIALS
        user = '****'
        password = '****'
        host = 'X.X.X.X'
        port = 3306
        database = 'test_database'

        engine = create_engine(url="mysql+pymysql://{0}:{1}@{2}:{3}/{4}".format(user, password, host, port, database))
    
        table = 'test_bluesg'
        with engine.connect() as conn:
            print(engine)
            df = getRawData()
            df.to_sql(table, engine, if_exists = 'append')
            print(pd.read_sql(table,conn).tail())
            print('DONE')
```

### Infrastructure

In my first iteration of this data scraper, I used AWS in the following configuration such that the entire pipeline including scheduling, scripting, and database hosting, was hosted in the cloud. 

{% include img-wrap name="7-aws-arch.png" caption="AWS Architecture" %}

I went on holiday for a month, happily letting the services run and came back to a bill of ...

{% include img-wrap name="8-aws-bill.png" %}

So that's that. I guess I'm banned from Amazon now.

Subsequently, I decided to run the scraper off a local RaspberryPi with 32GB of storage (all for less than SGD$100), a more sustainable option. This was much easier to handle since the Python scripts were being run locally by `cron` and the data stored directly in the Pi's `MariaDB` instance. It also made data access easier since the data was being transferred within my local network. No more dealing with AWS bastions and NAT gateways.

The setup was as simple as `cron` scheduling:

    pi@raspberrypi:/home $ crontab -l
    # */5 * * * * /usr/bin/python3 /home/pi/rpi_bsg_querywrite.py

Now that the data collection was setup, I just had to wait. Due to some connection issues in the Pi, I had some periods where data was not collected. Overall, I had a maximum of 2 weeks where the data was consistently being collected every 5 minutes, so I used that as the dataset for modelling.

## Exploratory Analysis

Before going into modelling, I did some basic analysis to see if there were any trends in the data across time and space.

Some metrics I defined were:

**Station activity**: How many rentals/ returns occurred at the station in a given period? A rental/return is defined as when the `cars` value changes from its previous value. 

{% include img-wrap name="9-activity.png" %}

Interesting that the most active stations are in the outskirts of the city.

**Station availability**: How many `cars` are there on average in a given time period? 

{% include img-wrap name="10-avail.png" %}

Cars in the North are the least utilised, while those in North-East and East are the most utilised.

I also looked at how car availability changed over the week:

{% include img-wrap name="11-timetrend.png" %}

Here we can see a clear trend for weekdays where cars are least available at the start of working hours (9-11am). Weekends have a different trend where cars are unavailable for the entire day.

How about the movement of cars across space? Although we don't know the start and ending stations of each car, we do know the change in each station. When visualised, we can see a clear trend of cars 'moving' from outskirts to the center in the morning, and back out again at night.

{% include img-wrap name="12-timelapse.gif" caption = "24 hour timelapse starting at 02:40AM" %}

Finally, we can also estimate the scale of BSG's operations by looking at how many cars are on the road at any given time. 

We first assume that BSG built their parking stations to accomodate 50% capacity at any point in time so that they have ample capacity in case drivers like to cluster at specific locations depending on the time (which is true from our analysis).

We sum up the total number of `cars` at all stations at any given time, and then multiply by the capacity which will total to ~ 750 cars in total that BSG owns.

From there, we can estimate the total number of cars on the road at any time by subtracting `cars` from the 750. 

{% include img-wrap name="13-totaltrend.png" caption="Estimated cars on road by time" %}

At the time of analysis, BSG has an average of 106 rentals ongoing at any time. At $0.42/ min, that is $23 million / year in revenue!

## Modelling

In the analysis, I looked at the stations in aggregate, but the real complexity lies in the time-series trends of each station:

If we look at `cars` for each station, the trend looks almost random.

{% include img-wrap name="14-cars-1.png" caption="Cars available over time for selected stations" %}
{% include img-wrap name="14-cars-2.png" %}

 `cars` can only take 1 out of 5 (sometimes 3 or 4 depending on the station) values. The value also doesnt change very often.

Rather than a time-series trend, this data looks more like random state changes from `0 --> 1` or `3 --> 2`. So here lies the important question, should this data be modelled as a state-change or a continuous/ ordinal variable?

Adding to the complexity is that the value is neither continuous nor ordinal. It is simply a count of `cars`. 

Going back to the objective: what are we trying to model?

> Given the `cars` value of all other stations at T, predict the `cars` value of station (S) at time T+1?

I made the decision **not** to do a moving average over the `cars` value becuase the end goal of this prediction must be a decisive value:

> 0 = No, this station will have NO cars at T+1 <br> Non-0 = Yes, this station will have cars at T+1

Having a moving average is not useful for the user since the prediction needs to be valid specifically in the next 5 mins.

Before I attempted to fit a model, I also had to decide how to engineer the features:
- How should the values be normalised?
- Should I use T-1, T-2, or T-3, to predict values at T? 
- Can I difference them to look at their state change rather than their absolute count?
- For time-series models, can I leverage on the half-daily oscillations observed in EDA to get a valid lag value?
- Can the information provided by all other stations at T-1 be summarised into another feature that has better correlation with station (S)?


Here comes the hard part. I tested a variety of multi-variate models with combinations of the above feature engineering including:
- Random Forest
- Gradient Boosting
- AutoML with GridSearch (restricted to ML algorithms)
- LSTM
- Combined VAR-LSTM
- Combined RF-LSTM neural network
- Hidden Markov Model
- Spatio-temporal LSTM models

All to no avail. The most promising model - RF-LSTM had an RMSE of ~0.7 predicting a value which ranged from 0 to 4, which is basically useless since it would be easier to just flip a coin every 5 minutes to get your prediction. 

The NN also looked like it was overfitting, because test scores didn't improve much from the get go.

{% include img-wrap name="15-nn-hist.png" caption="Model training history" %}

I was iterating over the same notebook (bad practice), and don't have saved results of each method. But I do remember the general lessons from each.

**The question whether to use a time-series model vs. correlation model:**

 I decided that correlation models would make more sense given the context of the data. Since the station state changes are basically random when viewed individually, there is little to no auto-correlation. 

The current state of all other stations gives much more information about station (S) than S's own history. This is because the number of cars in total are fixed, the only unknowns are how many cars there are on the road and where and when they will be returned to a station. This is still a massive unknown, which is why I suspect there is too little for the model to predict well.

I forgot to save an image of the prediction vs. actual overlays so you'll have to trust me that it was bad.

## Further Research

I asked a local Data Science group for recommendations on what else I could do to get better model results:

> Simplify the task: Instead of predicting the number of cars available, just predict if the station will have no cars (0) or some cars (1). 

> 2 weeks of data may not be long enough. Try a longer context window or larger dataset.

> Stick to simpler models like XGBoost and RF.

> Aggregate the data. Instead of predicting on 5 minute intervals, try predicting hourly intervals.

The one that makes the most sense to me is to simplify this into a binary classification problem and use a simpler model like RF on it. I believe it will come down to proper feature engineering, as the station-specific data is too random to be useful on its own. But 

As for the database management, I am looking at time-series databases like Influx which are more optimised and uses less storage. I cover the usage of InfluxDB as a database for another PoC project (link coming soon) which combined StockTwits sentiment data and actual prices for various cryptocurrencies.

Finally, **if** I manage to train a somewhat useful prediction model, I plan to deploy the backend on the same Pi with periodic model retraining. Front-end will use Plotly Dash since it comes with a built-in webserver and easy to design interactive charts.

*That's all for tonight, ciao.*


