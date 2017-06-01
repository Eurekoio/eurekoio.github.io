# Data Inspection

Before we start testing algorithms, we should take a look at our data. For this walkthrough we're going to use the daily data for the XLF ETF. The XLF ETF is an exchange traded fund consists of a basket of financial stocks. I chose this ETF because it's been around since 1998 so there's almost 20 years worth of daily financial data. Since its inception the market has had two once-in-a-lifetime downturns. During the 2008 crash, a couple components of the XLF ETF went bankrupt so we can see how that affects our trading strategies.

!> Code for the following section is available in the [github repo](https://github.com/Eurekoio/research/tree/master/regression/000).

# Data Retrieval 

The following bit of code will retrieve data from our database and create a pandas dataframe that we can work with. 

```python
SQL = """
SELECT
  dt
, ret
, ABS(ret) AS abs_ret
, SUM(ret) OVER (ORDER BY dt) AS cum_ret
, MUL(ret_factor) OVER (ORDER BY dt) AS compound_ret
, adj_close
FROM
(
    SELECT
      dt
    , (adj_close - LAG(adj_close, 1) OVER (ORDER BY dt) ) / LAG(adj_close, 1) OVER (ORDER BY dt) AS ret
    , 1 + ((adj_close - LAG(adj_close, 1) OVER (ORDER BY dt) ) / LAG(adj_close, 1) OVER (ORDER BY dt)) AS ret_factor
    , adj_close AS adj_close
    FROM xlf_etf_data
) a
ORDER BY dt
"""

df = pd.read_sql_query(SQL, con=conn)
```

For those of you who may have only dealt with a database through an ORM your entire career, let me introduce you to some SQL and more specifically window functions. Window functions are indispensible when analyzing financial data since much of the research associated with financial markets center around windows of data. If you haven't dealt much with window functions, take a look at [this tutorial](http://www.postgresqltutorial.com/postgresql-window-function/) before moving on. 

The best way to walkthrough SQL sub queries is to start with the innermost query so let's take a crack at it.

```sql
SELECT
  dt
, (adj_close - LAG(adj_close, 1) OVER (ORDER BY dt) ) / LAG(adj_close, 1) OVER (ORDER BY dt) AS ret
, 1 + ((adj_close - LAG(adj_close, 1) OVER (ORDER BY dt) ) / LAG(adj_close, 1) OVER (ORDER BY dt)) AS ret_factor
, adj_close AS adj_close
FROM xlf_etf_data
```

In this query we're getting the date (`dt`), returns (`ret`), factored return (`ret_factor`) and the adjusted close (`adj_close`). The date and adjusted close should be self explanatory. Returns are calculated based on `(today's close - yesterday's close) / yesterday's close`. The factored returns adds `1` to the return and is used when calculated our compounded returns below. We'll run another SQL query that uses data from this subquery. 

```sql
SELECT
  dt
, ret
, ABS(ret) AS abs_ret
, SUM(ret) OVER (ORDER BY dt) AS cum_ret
, MUL(ret_factor) OVER (ORDER BY dt) AS compound_ret
, adj_close
FROM
(
    ...
) a
ORDER BY dt
```

In this pass, we calculate the absolute return, cumulative return and compound return. We'll use these to visualize our data in the following sections.

!> It's best to minimize the number of passes when writing subqueries. As a result, it's almost always more performant to duplicate code in a subquery rather than run another query.

## Data profile

Pandas has a nifty `describe` method attached to your dataframe object that let's you get a quick summary of data. Let's run it and take a look at the results.

```text
$ df.describe()
               ret      abs_ret      cum_ret  compound_ret    adj_close
count  4574.000000  4574.000000  4574.000000   4574.000000  4575.000000
mean      0.000337     0.012189     0.633268      1.191564    15.611933
std       0.020107     0.015994     0.374472      0.311003     4.074639
min      -0.173613     0.000000    -0.475585      0.329465     4.316813
25%      -0.007352     0.003328     0.365585      0.950934    12.464097
50%       0.000448     0.007735     0.594965      1.197367    15.687239
75%       0.008048     0.015309     0.865041      1.408304    18.452312
max       0.313869     0.313869     1.546321      1.903007    24.934158
```

The describe method gives you the number of observations, the mean of each column, the standard deviation, min, max and percentiles. Let's further breakdown the percentiles.

```text
$ df.describe(percentiles=[.05, .25, .75, .95])
               ret      abs_ret      cum_ret  compound_ret    adj_close
count  4574.000000  4574.000000  4574.000000   4574.000000  4575.000000
mean      0.000337     0.012189     0.633268      1.191564    15.611933
std       0.020107     0.015994     0.374472      0.311003     4.074639
min      -0.173613     0.000000    -0.475585      0.329465     4.316813
5%       -0.027079     0.000653     0.062069      0.704306     9.228332
25%      -0.007352     0.003328     0.365585      0.950934    12.464097
50%       0.000448     0.007735     0.594965      1.197367    15.687239
75%       0.008048     0.015309     0.865041      1.408304    18.452312
95%       0.026318     0.037491     1.276725      1.764994    23.125502
max       0.313869     0.313869     1.546321      1.903007    24.934158
```

Breaking down the percentiles to show the 5% and 95% percentiles, we can see that the mean returns in those percentiles are much larger than those in the 25% to 75% percentiles. Although it's not something actionable from a trading perspective, it's good to know that your data has some significant down days and up days.

# Data Visualization

## Price chart

> First things, first. Let's plot our price data.

```
p = ggplot(aes(x='dt', y='adj_close'), data=df)
p + geom_line() + ggtitle("XLF adj_close")
```

![XLF Price Chart](https://raw.githubusercontent.com/Eurekoio/research/master/regression/000/xlf_adj_close.png)

A historical price chart give us a quick overview of our price data. When looking at a price chart I focus on any obvious regime changes to that a particular security and how security performed during known market regime changes.

For the XLF, it looks like the index went relatively unscathed during the 2001-2002 market downturn and rallied along with market until the 2008 market downturn. The XLF didn't fare as well during the market crash of 2008 as it did during the earlier downturn.  Fundamentally, this makes sense since the downturn 2001-2002 market downturn was largely driven by the bursting of the dotcom bubble while the 2008 market crash was driven by financial stocks.

Although, there are couple significant corrections, the index then rallied along with the general market for the past 10 years or so. An interesting observation is the sharp rally in the index after the 2016 elections. 

## Cumulative Return 

> The cumulative return is the simple cumulative sum of the returns

```
p = ggplot(aes(x='dt', y='cum_ret'), data=df)
p + geom_line() + ggtitle("XLF cumulative return")
```

![XLF Cumulative Return](https://raw.githubusercontent.com/Eurekoio/research/master/regression/000/xlf_cum_ret.png)

Off the bat, you should notice that the cumulative return chart looks a little different than the price chart. From 2009 on, it looks like the cumulative returns significantly outperforms the ETF. This is because the cumulative returns assumes you have an equal amount at risk on any given day.

Let's assume you start with $1,000 at the beginning of investing in the XLF ETF. The cumulative return would assumes that you keep you account at a $1,000 when beginning a new trading day. If you lose money you deposit enough to keep your balance at $1,000 and vice versa when you make money. In order to adjust for this, we have calculate the cumulative compounded return. 

## Compound Returns

> The compound return multiplies yesterday's cumulative compounded return by today's return

```
p = ggplot(aes(x='dt', y='compound_ret'), data=df)
p + geom_line() + ggtitle("XLF compound return")
```

![XLF Compound Return](https://raw.githubusercontent.com/Eurekoio/research/master/regression/000/xlf_compound_ret.png)

Compound returns better track the historical price chart and reflects the overall percentage gain based on a buy and hold strategy. Since we start with an index of 1.00, the most recent value of approximately 1.80 reflects an 80% gain that would have been realized if we had bought the ETF in 1999.


## Density and Histogram of Returns

### Density Distribution

```
df.ret.plot.kde()
plt.show()
```

![XLF Returns Density](https://raw.githubusercontent.com/Eurekoio/research/master/regression/000/xlf_ret_density.png)

The density of return reflects the distribution of the count of returns. The one big takeaway in this chart is how far out the outside values are from the center. There are many financial models that assume a normal distribution. The distribution of our data would break these models.

### Histogram

```
df.ret.hist(color='k', alpha=0.5, bins=50)
plt.show()
```

![XLF Returns Histogram](https://raw.githubusercontent.com/Eurekoio/research/master/regression/000/xlf_ret_histogram_bin_50.png)

The histogram of returns takes the count of the returns and separates them into bins. We're using 50 bins in this example. This chart is similar to the density of returns chart above. Not much additional information gain here. 


## Plot Returns and Absolute Returns

In order to compare apples to apples, you have to convert price data to a standard scale. This is obvious when comparing the price of one security to another but it also applies to a single stock as well. Imagine a security whose price ranges from $1 to $100 in any given year. If you used the difference in price in your analysis the magnitude of the differences would vary dramatically depending on whether the current price was a $1 or $100. 

Using returns addresses this problem. There are other way to address this problem but `n-period` returns are widely accepted way to standardize price differences. As you learn and read about different methodologies standardize `n-period` differences be extra careful with those formulas that use the maximum or minimum value for a given period. They have a tendency to data snoop and that could unintentionally propagate through a model.

### Returns
```
p = ggplot(aes(x='dt', y='ret'), data=df)
p + geom_line() + ggtitle("XLF returns")
```

![XLF returns](https://raw.githubusercontent.com/Eurekoio/research/master/regression/000/xlf_returns.png)

When looking at a plot of returns, I try to make a mental note of where the larger changes bunch. They usually coincide with regime changes in the price of the security. In the plot above we can see that the largest swings occurred during major market corrections. It's important to note that major market corrections aren't just characterized by down days but down days followed by up days. Another common observation is that you'll see volatility bunched up rather than randomly distributed across your timeline. 


### Absolute Returns
```
p = ggplot(aes(x='dt', y='abs_ret'), data=df)
p + geom_line() + ggtitle("XLF absolute returns")
```

![XLF Absolute Returns](https://raw.githubusercontent.com/Eurekoio/research/master/regression/000/xlf_absolute_returns.png)

Absolute returns are a simple measure of volatility. This plot confirms what we saw on the returns plot: volatility is concentrated during market downturns. In the next section, we'll take a look whether there is any auto correlation in the volatility.

## Autocorrelation 

> [Autocorrelation](http://www.investopedia.com/terms/a/autocorrelation.asp) is the correlation between the elements of a series and others from the same series separated from them by a given interval.

Basically, autocorrelation quantifies whether today's return can be used to help predict tomorrow's return. Autocorrelations scale from -1 to 1. A prices series with an autocorrelation of 1 for lat 1-day means its daily return perfectly correlates with tomorrow's return. So for a price series with a 1-day lag autocorrelation of 1, a 1% would be followed by another 1% day. The opposite would be trued if the 1-day lag autocorrelation was -1. 

Apparently, there was a day when autocorrelation ran rampant in different markets but, alas, those days are long gone.

### Returns
```
plt.figure()
autocorrelation_plot(df.ret.dropna())
plt.show()
```

![XLF Returns Auto Corr](https://raw.githubusercontent.com/Eurekoio/research/master/regression/000/xlf_ret_auto_corr.png)

We check autocorrelation to see if there are any tradeable patterns in the data or whether we're just looking at random data. The autocorrelation plot plots lines to delineate the the 95% and 99% confidence interval of correlations. If the bars for the different lag periods remain within the 99% confidence intervals then your data set is essentially random. 

The autocorrelation above shows that our data is essentially random but there some signals on the shorter term lag periods. Let's plot just only the first 20 lag periods.

```
plt.acorr(df.abs_ret.dropna(), maxlags=20)
plt.show()
```

![XLF Returns Auto Corr](https://raw.githubusercontent.com/Eurekoio/research/master/regression/000/xlf_ret_auto_corr_20.png)

So what looked like interesting signals are just small little blips here. Stat bros tends to dismiss a chart like since the signals are relatively week. However, crafting a trading system requires you to follow subtle hints. Although it's not significant, the more significant blips occur at the 5 and 10 day lag. The one day lag is more significant than the other and negatively correlate with the zero period. We can also see that all the lag periods up to 5 exhibit negative correlation. 

How can we use this information as we design our trading strategy? Given the negative autocorrelation on the shorter term lags, we should expect to employ a mean reversion trading strategy as opposed to momentum oriented one. We should also focus on shorter term indicators and trades. 

### Absolute Returns
```
plt.figure()
autocorrelation_plot(df.abs_ret.dropna())
plt.show()
```
![XLF Absolute Returns Auto Corr](https://raw.githubusercontent.com/Eurekoio/research/master/regression/000/xlf_abs_ret_auto_corr.png)

This is probably the most interesting chart of the bunch. It looks like there's significant autocorrelation in the absolute volatility to stocks during the short term. Again, let's plot the autocorrelation on just the first 20 lag periods.

```
plt.acorr(df.abs_ret.dropna(), maxlags=20)
plt.show()
```
![XLF 20 day Auto Correlation](https://raw.githubusercontent.com/Eurekoio/research/master/regression/000/xlf_abs_ret_auto_corr_20.png)

This is what I would consider significant autocorrelation in short term volatility. There are trading strategies to take advantage of this but it's outside of the scope of this walkthrough.

## Summary

Now that we've inspected our data and digested some of our findings let's move on to setting up an algorithm for backtesting.