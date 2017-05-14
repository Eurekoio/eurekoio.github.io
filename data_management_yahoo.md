# Yahoo Finance Data

## Fetching Data

Yahoo finance has always been a great resource for free financial information. It was really the one and only option until Google finance came along and unlike other parts of Yahoo, management didn't screw up Yahoo Finance as much as the rest of the site.

Yahoo finance has always made their historical financial data available for download in csv format and it's a breeze to download.

In Python, here's how you fetch data from Yahoo Finance for a stock like Apple (ticker: AAPL).

```python
import urllib

base_url = "http://ichart.finance.yahoo.com/table.csv?s=%s"
response = urllib.urlopen(base_url % 'AAPL')
data = response.readlines()
print data
```

That's it! If you run this in the Jupyter Notebook, you should see something like this:

```python
['Date,Open,High,Low,Close,Volume,Adj Close\n',
'2017-02-17,135.100006,135.830002,135.100006,135.720001,22084500,135.720001\n',
'2017-02-16,135.669998,135.899994,134.839996,135.350006,22118000,135.350006\n',
'2017-02-15,135.520004,136.270004,134.619995,135.509995,35501600,135.509995\n',
'2017-02-14,133.470001,135.089996,133.25,135.020004,32815500,135.020004\n',
'2017-02-13,133.080002,133.820007,132.75,133.289993,23035400,133.289993\n',
'2017-02-10,132.460007,132.940002,132.050003,132.119995,20065500,132.119995\n',
'2017-02-09,131.649994,132.449997,131.119995,132.419998,2834990
...
```

`urllib` allows you to open arbitrary network resources. Here we build a url by using the Yahoo Finance url, pass in a ticker symbol and user `urllib.urlopen` to download the `csv` historical price data for Apple. We use `readlines`, which give us a list of "lines" that we can parse through.

### OHLC

I'll go through this quickly for those who are new to financial data. This comma delimited file is split into columns with headers Date, Open, High, Low, Close, Volume, Adj Close. This data structure is commonly referred to as OHLC, which is short for open, high, low, close. Here's a quick summary of each column.

**Date:** The date of the financial data in the corresponding row

**Open:** This is the first price at which the security was traded at 9:30AM Eastern Standard Time.

**High:** This is the highest price that the security traded at during the trading day.

**Low:** This is the lowest price that the security traded at during the trading day.

**Volume:** This is the total amount of shares that exchanged hands on the date.

**Adj Close:** If there is an adjustment in reconciling the last trade, it would show up here. This doesn't happen as much as it used it.

This data structure is pervasive in financial data so get comfortable with it. Let's move on to parsing!

## Parsing

We've got our data, now what do we do with it? Rather than insert each line into the database separately we're going use Postgresql's `copy from` function to perform a bulk insert. Before we do that we have to write the data to file.

Let's take a look at our data and think about how we want to parse the data and set up our table.

```python
Date,Open,High,Low,Close,Volume,Adj Close
2017-02-13,133.080002,133.820007,132.75,133.289993,22926400,133.289993
2017-02-10,132.460007,132.940002,132.050003,132.119995,20065500,132.119995
```

It's obvious that we'll need a table with the columns `date, open, high, low, close, volume, and adj_close`. But what about our ticker symbol. One way to attack the problem is to create a table for each ticker symbol. Although this will work, if we have more than a couple ticker symbols, managing all those tables would be a mess and you would probably have to keep track of your data in the  application layer.

For sanity's sake, we're going to use a single table for our historical data with all of the columns mentioned above and another column labeled `ticker`. This requires us to split the data on each line and add the ticker symbol at the beginning of each line before we write it to disk. In our example, a written line should look like this:

```python
AAPL, 2017-02-13,133.080002,133.820007,132.75,133.289993,22926400,133.289993
AAPL, 2017-02-10,132.460007,132.940002,132.050003,132.119995,20065500,132.119995
```

Now that we know what we're trying to do let's take a look at some code.

```python
# initialize data.txt
f = open('/tmp/data.txt', 'w')
f.close()

# Parse and write data
for d in data:
    Date, Open, High, Low, Close, Volume, Adj_Close = d.split(',')
    data_string = '%s,%s,%s,%s,%s,%s,%s,%s' % (TICKER, Date, Open, High, Low, Close, Volume, Adj_Close)
    f = open('/tmp/data.txt', 'a')
    f.write(data_string)
    f.close()
```
First we initialize `/tmp/data.txt` so that we have an empty file we can append. If you don't do this, you might have data left over from the last time you ran this script.

For every line `d` in `data`, we split the line on the commas so that each field gets mapped to their corresponding variable. Next, `data_string` represents a new line with the ticker symbol prepended to it. This new line then gets written to `/tmp/data.txt`.

Now that we have our data written to disk let's copy it to our Postgresql cluster.

## PostgreSQL

The fastest way to get your data into Postgresql is to use the `COPY` command. The `COPY` command is like a batch insert but much faster and you should use every time you find yourself performed serial inserts.

The `COPY` command requires the number of columns in the data to match the number of columns in the table that you're copying to. So, let's create a table for a our data.

```sql
CREATE TABLE yahoo_data (
  ticker TEXT
, dt DATE
, open NUMERIC
, high NUMERIC
, low NUMERIC
, close NUMERIC
, volume NUMERIC
, adj_close NUMERIC
);
```

Here we just create a table with column names that mirror the header in our csv file. Once you have this table set up, all you need to do is copy your data to the table with the following command:

```sql
COPY yahoo_data FROM '/tmp/data.txt' WITH DELIMITER ',' CSV HEADER;
```

Here we use `COPY FROM` with the `WITH DELIMITER ','` option that tells Postgresql that our data is comma delimited. The `CSV HEADER` option tells Postgresql to ignore the header. You can ignore the header while writing the data to disk but I found that it's easier to use the `CSV HEADER` option instead.

That's it! You know have stock data that you can query and analyze to your heart's content. If you query your data you should have something that looks like this:

```sql
historical_stock_data=# select * from yahoo_data limit 10;
 ticker |     dt     |    open    |    high    |    low     |   close    |  volume  | adj_close  
--------+------------+------------+------------+------------+------------+----------+------------
 AAPL   | 2017-02-17 | 135.100006 | 135.830002 | 135.100006 | 135.720001 | 22084500 | 135.720001
 AAPL   | 2017-02-16 | 135.669998 | 135.899994 | 134.839996 | 135.350006 | 22118000 | 135.350006
 AAPL   | 2017-02-15 | 135.520004 | 136.270004 | 134.619995 | 135.509995 | 35501600 | 135.509995
 AAPL   | 2017-02-14 | 133.470001 | 135.089996 |     133.25 | 135.020004 | 32815500 | 135.020004
 AAPL   | 2017-02-13 | 133.080002 | 133.820007 |     132.75 | 133.289993 | 23035400 | 133.289993
 AAPL   | 2017-02-10 | 132.460007 | 132.940002 | 132.050003 | 132.119995 | 20065500 | 132.119995
 AAPL   | 2017-02-09 | 131.649994 | 132.449997 | 131.119995 | 132.419998 | 28349900 | 132.419998
 AAPL   | 2017-02-08 | 131.350006 | 132.220001 | 131.220001 | 132.039993 | 23004100 | 131.469994
 AAPL   | 2017-02-07 | 130.539993 | 132.089996 | 130.449997 | 131.529999 | 38183800 | 130.962201
 AAPL   | 2017-02-06 | 129.130005 |     130.50 | 128.899994 | 130.289993 | 26845900 | 129.727549
(10 rows)
```

## Summary

Although getting data from different service providers will require you to write different adapters and probably in a different programming language, the basic premise will always be the same: fetch, parse, upload. You could just download a slug of data every time you need it but I found that maintaining the price data in a database makes it easier to deal with when backtesting and it facilitates working with larger data sets.
