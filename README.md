### Project Overview
see [here](https://www.youtube.com/watch?v=xzDcPKJ7-qA) for app walk through.

It's no news that the stock market has been quite volatile recently. From the pandemic caused by covid, to
the events surrounding elections, and even with recent news of covid vaccines, so many things could cause a stock price to move big. I've reecntly been doing some research
into the stock market and one thing I find my self always trying to do it find out what stocks spiked on a particular day, or what catlyst caused a stock to spike on some time given time frame.

This app provides a simple seach interface for a user to select a date and the see stocks that moved big(positive or negative) on that given date. And then the use can select a
stock from given list and be directed to some possible news that caused the spike.

I also provide another interface to perform a search based on a sigle company stock where the user can give a date range to see if the stock has made big spikes(larger than 15% move) in the given range.
The user can then click through to see see the different spikes for the date(or date range).

NOTE: I'm currently only dealing with US stocks for my app.
NOTE: Any IP address or url specified below to view live app is currently outdated since app is no longer live.



### Contents of this markdamobi_project_submission folder.
1. markdamobi-project-rails.zip - zipped copy of my rails code. Actually code on webserver is deployed with copy on my github.
2. markdamobi-project-node.zip - zipped copy of my nodejs code. deployed copy can be found on server.
3. ProcessNewStockPrice.zip - code spark-scala code to process kafka topic. uberjar deployed on hdfs name node.
4. install_rails_app_on_new_server.md - for posterity, instructions I used to setup a new webserver for my rails deployment.
5. README.md - description of project.

### Usage
To use this app, you can find it on our single web server [here](http://ec2-3-15-219-66.us-east-2.compute.amazonaws.com:3334/), or on our loadbalanced servers [here](http://mpcs53014-loadbalancer-217964685.us-east-2.elb.amazonaws.com:3334/). I have included a demo video as part of my submission to show how to use the application.


### Design/Architecture.
This application is built using the Lambda architecture design pattern. For the batch layer,  I use HDFS to store my master data set. The serving layer is backed by HBase. And finally, the speed layer is done using spark to process new messages that come into my kafka stream.

The web portion of the app is split into 2 main pieces. A Ruby on Rails portion, and a NodeJS portion. The rails portion handles all frontend organization and routing, and the NodeJS portion is simple an API interface to serve views from my HBase tables. 
#### Rails Application
My rails applications contains all the code for the frontend and interface of my applications. It also contains code to interact with my NodeJs API.
It also constains code that I use to write new data to Kafka. I picked rails as the main interface for my application because it's easy to use and it's most familiar to me.

I deployed this using capistrano. Deployed code can be found under `/var/www/markdamobi-project-rails/current` in the web servers. The deployment for my rails application was
mostly based on [this](https://www.phusionpassenger.com/library/deploy/standalone/automating_app_updates/ruby/) and as part of my submission, I've included the steps I used to
setup each of the servers so that I can have an easy deploy by running `bundle exec cap production deploy` locally when I need to update code.


#### NODEJS Application
My NodeJS application is simply an API to get info from the databases to server data to my web app. I originally considered doing everything in rails but I was having a hard time finding an up-to-date ruby gem to interact with hbase. So I decided to have separate API for the hbase interaction. One good thing with this though is that I can scale the
API independent of the rails web app and I can also build other things on this API if needed.

I used CodeDeploy to deploy my nodejs API to the servers. Can be found at `/home/ec2-user/markdamobi/markdamobi-project-node` in the webservers.

I currently expose 3 end points in my API.
1. `/api/data` - This is useful for returning list of stocks making significant moves for a single day. it takes in date as query params.
2. `/api/data/single_data` - This is used to return list of possible spikes for a single stock for a give day or date range. takes in ticker, from_date and optional to_date as query params.
3. `/api/us_tickers` - This is for getting list of available US tickers and can also be used to check if a specific ticker is available by including a `ticker` query param. I use this end point to get list of tickers which I use to generate data for in my [speed layer](#speed-layer).

All these endpoints interact with my hbase tables to get data. See [databases](#databases) for more details about my databases.

One thing that is worth mentioning is that right now, this API is public. Ideally, I should have made it private. I think it would have been useful to setup some API tokens to prevent unauthorized access.


#### Spark Scala Script
I have some spark-scala code for processing my kafka topic as part of the [speed layer](#speed-layer). The uberjar can be found at `/home/hadoop/markdamobi/project/target/uber-ProcessNewStockPrice-1.0-SNAPSHOT.jar` in the namenode of our hdfs cluster.


#### Databases.

I'm storing my master dataset in HDFS and using [Hive as an interface](#creating-hive-tables) to do that. And the I create a couple of useful [Hbase views](#hbase-serving-layer-views) for my serving layer. My speed layer is done with kafka and spark as described [below](#speed-layer).



### Data
My data wih some basic stats.
- https://www.kaggle.com/aceofit/stockmarketdatafrom1996to2020/download
- 104124 price files.
- 211 million price rows before cleaning.
- 110 million price rows after some cleaning.
- 27 million price rows relevant for US.
- 1996-01-02 - 2020-08-06 dates for US.


### Ingestion Process.

My ingestion process is described below:


- download the file locally.
- unzip file. (2.3GB ~> 13GB)
- convert `Tickers.xlsx` to csv (`Tickers.csv`).
- Original folder structure was
  ```
  archive/
    Data/
      Data/*
    Tickers.xlsx
  ```
  make new structure as:
  ```
  data/
    Prices/
      *.csv
    Tickers/
      Tickers.csv
  ```
- Using some ruby script, remove Headings from every csv file before ingesting into HDFS.
```ruby
ticker_files = Dir.glob("./data/Tickers/*.csv")
price_files = Dir.glob("./data/Prices/*.csv")
ticker_files.each { |f| system("sed -i '' 1d #{f}") }
price_files.each { |f| system("sed -i '' 1d #{f}") }
```

- Recompress files as `data.zip`.(~ 30 mins)
- upload to s3 bucket under `s3://markdamobi-mpcs53014/project/stock_market_data/data.zip` (~ 18 mins)
- download to namenode (I know this is bad, but I couldn't figure out the best way right now.)

  `aws s3 cp s3://markdamobi-mpcs53014/project/stock_market_data/data.zip /home/hadoop/markdamobi/project/stock_market_data/`

- unzip the contents of the file into `/home/hadoop/markdamobi/project/stock_market_data/`.

- transfer files from name node to hdfs.

`hdfs dfs -put /home/hadoop/markdamobi/project/stock_market_data/data/ /tmp/markdamobi/project/stock_market_data/`


NOTE: One thing I realized after the fact was that Hive has an option to ignore first line of CSV when creating a table. This means that I didn't have
to run an ruby script to pre-process the prices files.



### Creating Hive tables.

```
-- map csv for prices

create external table markdamobi_prices_csv(
  effective_date String,
  open double,
  high double,
  low double,
  close double,
  adj_close double,
  volume bigint
)
row format serde 'org.apache.hadoop.hive.serde2.OpenCSVSerde'

WITH SERDEPROPERTIES (
   "separatorChar" = "\,",
   "quoteChar"     = "\""
)
STORED AS TEXTFILE
  location '/tmp/markdamobi/project/stock_market_data/data/Prices';

-- create orc table for efficiency.
create table markdamobi_prices(
  ticker string,
  effective_date date,
  open double,
  high double,
  low double,
  close double,
  adj_close double,
  volume bigint
)
stored as orc;

-- Copy the CSV table to the ORC table. also extract ticker symbol from input file name.

insert overwrite table markdamobi_prices
select
  regexp_extract(INPUT__FILE__NAME, '.*/Prices/(.*).csv',1),
  cast(effective_date as date),
  open, high, low, close, adj_close, volume
from markdamobi_prices_csv
where open is not null and open != 0
and close is not null and close != 0
and effective_date is not null and effective_date != 'null'
and high is not null
and low is not null
and adj_close is not null
and volume > 0;

-- map csv for tickers

create external table markdamobi_tickers_csv(
  ticker string,
  name string,
  stock_exchange string,
  category_name string,
  country string
)
row format serde 'org.apache.hadoop.hive.serde2.OpenCSVSerde'

WITH SERDEPROPERTIES (
   "separatorChar" = "\,",
   "quoteChar"     = "\""
)
STORED AS TEXTFILE
  location '/tmp/markdamobi/project/stock_market_data/data/Tickers';



-- create orc table for efficient reading.

create table markdamobi_tickers(
  ticker string,
  name string,
  stock_exchange string,
  category_name string,
  country string
)
stored as orc;

-- Copy the CSV table to the ORC table.

insert overwrite table markdamobi_tickers select * from markdamobi_tickers_csv
where country is not null and country != '';

```


- At this point, my ground truth tables will be `markdamobi_prices` and `markdamobi_tickers`.

prices table contains `211765465` rows(`211759566` after removing some blanks). tickers table contains `105813` rows.


### Hbase Serving Layer Views.

#### First View - Percentage bands.
for each day, group stock movements into percentage changes. For example. Say on 05/12/2018 AAPL opened at 100 and closed ar 120.
I'll have a row in my hbase view that looks like
05/12/2018:20:AAPL,ticker,name,open,high,low,close,percent_change


NOTE: for a future improvement, I'd probably make the percentage change to be between previous close and high of day, And previous close and low of day(in case bad news).
I think this would produce more interesting results because a lot of times, companies release news outside of market hours.

```
create table markdamobi_spikes_hive(
  date_percent_ticker string,
  ticker string,
  name string,
  open double,
  close double,
  effective_date date,
  percent_change double,
  country string
)
STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
WITH SERDEPROPERTIES ('hbase.columns.mapping' = ':key,info:ticker,info:name,info:open,info:close,info:effective_date,info:percent_change,info:country')
TBLPROPERTIES ('hbase.table.name' = 'markdamobi_spikes_hbase');

insert overwrite table markdamobi_spikes_hive
select concat(effective_date,':',floor((close-open) * 100 / open),':',mp.ticker) as date_percent_ticker,
  mp.ticker,
  mt.name,
  open,
  close,
  effective_date,
  ((close-open) * 100 / open) as percent_change,
  country
from markdamobi_prices as mp
join markdamobi_tickers as mt on mp.ticker = mt.ticker
where (country = 'USA') and ((((close-open) * 100 / open) >= 15) OR (((close-open) * 100 / open) <= -15));

```


In order to support the other search form for query by ticker name, I'm creating this addition table that puts the ticker before the date and percent.
```
create table markdamobi_spikes_hive_by_ticker(
  ticker_date_percent string,
  ticker string,
  name string,
  open double,
  close double,
  effective_date date,
  percent_change double,
  country string
)
STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
WITH SERDEPROPERTIES ('hbase.columns.mapping' = ':key,info:ticker,info:name,info:open,info:close,info:effective_date,info:percent_change,info:country')
TBLPROPERTIES ('hbase.table.name' = 'markdamobi_spikes_hbase_by_ticker');

insert overwrite table markdamobi_spikes_hive_by_ticker
select concat(mp.ticker,':',effective_date,':',floor((close-open) * 100 / open)) as date_percent_ticker,
  mp.ticker,
  mt.name,
  open,
  close,
  effective_date,
  ((close-open) * 100 / open) as percent_change,
  country
from markdamobi_prices as mp
join markdamobi_tickers as mt on mp.ticker = mt.ticker
where (country = 'USA') and ((((close-open) * 100 / open) >= 15) OR (((close-open) * 100 / open) <= -15));

```


I also export the US tickers table to hbase.
```
create table markdamobi_us_tickers_hbase(
  ticker string,
  name string
)
STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
WITH SERDEPROPERTIES ('hbase.columns.mapping' = ':key,info:name')
TBLPROPERTIES ('hbase.table.name' = 'markdamobi_us_tickers_hbase');


insert overwrite table markdamobi_us_tickers_hbase select ticker, name from markdamobi_tickers where country = 'USA';
```


### News Engine
One of the main purposes of this App is to help provide insights on caalysts that cause significant movement in price. In order to this, I have a News Engine in my rails app
which uses a tool called `googler` to search for news related to the stock on a specific date. Right now, I return the top 10 results.


### Speed Layer.
The speed layer is responsible for processing data streams in real time. But I don't currently have any live feed to update stock prices in real time, so I provide a link in the app to insert new price data to test out real time streaming/processing. 
The new data is queued in my kafka topic `markdamobi_stock_price_info`. The below command is used to start up the spark session to process messages in my kafka topic:


```
spark-submit --master local[2] --driver-java-options "-Dlog4j.configuration=file:///home/hadoop/ss.log4j.properties" --class StreamStockInfo /home/hadoop/markdamobi/project/target/uber-ProcessNewStockPrice-1.0-SNAPSHOT.jar b-1.mpcs53014-kafka.fwx2ly.c4.kafka.us-east-2.amazonaws.com:9092,b-2.mpcs53014-kafka.fwx2ly.c4.kafka.us-east-2.amazonaws.com:9092
```

Note that the spark session above should be started before putting things in the topic. I'm sure there's an easy fix, but at the time of writing, I wasn't able to get it to work in the reverse way.

Another important thing to note is that in addition to the submit form I provide for adding new data, I provide a script that can be used to randomly generate new data for a given date for all US stocks I'm tracking.
This script is provided as a rake task. TO use this, follow the steps below:
1. start the `spark-submit` shown above.
2. go into one of our webservers, and then `cd /var/www/markdamobi-project-rails/current`
3. run `bundle exec rake generate_new_stock_data -- --start_date=<date> RAILS_ENV=production`.

eg.
```
bundle exec rake generate_new_stock_data -- --start_date=2020-09-01 RAILS_ENV=production
```

Note that a date range can also be provided.
```
bundle exec rake generate_new_stock_data -- --start_date=2020-10-01 --end-date=2020-10-05 RAILS_ENV=production
```

Note: before running the rake task, open the spark session with command above.


### Future Considerations.

There are a few things I could do to improve my process/App.

- Improve my ingestion process so that I'm not doing so many things manually, and also reduce the load on the HDFS name node.
- Expand the feature set in my app with some new ideas. eg. add support for other stocks besides US stock.
- Add more searching capability to increase usefulness. Such as filtering by a specific industry, and filtering for a specific percentage range. I think it will also be nice
  to be able to filter by Market cap so we know we're comparing apples to apples because larger companies don't make big spikes like mich smaller companies.
- Change the way I'm calculating percentage change. Right now, I'm using the difference between open and close price for a given day. I think it'll be more accurate and interesting
if I use use difference between previous day's close price and current day's highest price. this is because news catalysts often get announced when the market is closed.
So sometimes, the big moves happen outside of market hours and the intra day change may not always reflect what happened.
- Improve my news engine. Right now, I just return the first 10 results I find. I think it'll be a good idead to make the news engine more robust and maybe do some processing
  or maching learning with the news to ensure that I'm getting relevant info. Maybe even add some sentiment analysis on the news to verify that it matches the stock price spiking up or down.
- Add some security layer to my API. Right now, for simplicity my nodejs API is open. This means that anyone has access to it. I think it will be a good idea to add some sort of API authentication layer(maybe API tokens) so that only authorized users/apps can make use of it.