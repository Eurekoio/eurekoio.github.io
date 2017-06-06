# Linear Regression Models

A linear regression is seldom used as a first example for algorithmic trading. Usually, variations of trading signals derived from moving averages are the de facto starting point for learning about algorithms. Moving averages are easy to calculate and relatively easy to get your head around. However, linear regression models aren't that much harder to understand and it easily lends itself to more complicated trading strategies.

I won't go deep into what linear regression modeling is but you can check out these [sources](https://en.wikipedia.org/wiki/Linear_regression) [for](https://onlinecourses.science.psu.edu/stat501/node/250) a [quick](http://www.stat.yale.edu/Courses/1997-98/101/linreg.htm) [refresher](https://www.youtube.com/watch?v=ZkjP5RJLQF4). For our purposes, we'll just focus on a simple linear regression model with two variables. 

## Our first model

We're going to use the an `n-period` days worth of returns of a stock to predict returns for the following day. If we were running a linear regression on a stock today, our independent variable will be the previous `n-period` returns while our dependent value, the value we're trying to predict, will be tomorrow's return. It's important to get your head around because it's easy to get tripped up when you're writing code and expose yourself to [data snooping.](https://www.ma.utexas.edu/users/mks/statmistakes/datasnooping.html)

## Code Init

> If you haven't done it yet, make sure you've initiated your XLF repo.
> The code for this section is available [here](https://github.com/Eurekoio/research/tree/master/regression/001)

Let's apply our regression model and plot a our predictions and the actual results to see if there any visible non-random patterns. Here's the code to build our regression model to make our predictions. There's a bit going on here so digest the code before the walkthrough.

```python
def query(periods):
    sql = """
    SELECT
    ' {periods}_day_regression' as model_name
    , dt
    , (x * slope) + intercept AS pvalue
    , next_day_ret
    FROM
    (
        SELECT * 
        , regr_slope(next_day_ret, x) OVER deriv_wdw AS slope
        , regr_intercept(next_day_ret, x) OVER deriv_wdw AS intercept
        FROM
        (
            SELECT 
            feature_0.dt
            , feature_0.ticker
            , feature_0.ret AS x
            , dep.ret AS next_day_ret
            FROM xlf_returns feature_0, xlf_next_day_returns dep
            WHERE feature_0.dt = dep.dt
        ) AS a
        WINDOW deriv_wdw AS 
            (PARTITION BY ticker ORDER BY dt ROWS BETWEEN {periods} PRECEDING AND 1 PRECEDING)
    ) b;
    """
    return sql.format(periods=periods) # Don't do this in production

day_5_df = pd.read_sql_query(query(5), con=conn)
```

The first thing you may notice is that most of code is SQL. In fact, most of the work is done in PostgreSQL with three passes of our data. Doing backtests in databases saves you network round trips and provides you with better data inspection. As you start performing more complicated backtests, you'll deal with exponential data inflation and saving network round trips will pay off.

# Code walkthrough


## First Pass // Calculate Returns

```sql
SELECT 
feature_0.dt
, feature_0.ticker
, feature_0.ret AS x
, dep.ret AS next_day_ret
FROM xlf_returns feature_0, xlf_next_day_returns dep
WHERE feature_0.dt = dep.dt
```

This the first pass we join the current day returns with next day returns. This gives us something that looks like this: 

```text
     dt     | ticker |            x            |      next_day_ret       
------------+--------+-------------------------+-------------------------
 1998-12-22 | xlf    |                         |  0.01474535247145115037
 1998-12-23 | xlf    |  0.01474535247145115037 |  0.00660497782213908891
 1998-12-24 | xlf    |  0.00660497782213908891 | -0.01312336068227701268
 1998-12-28 | xlf    | -0.01312336068227701268 |  0.01063829877772755555
 1998-12-29 | xlf    |  0.01063829877772755555 | -0.00394732664819592827
 1998-12-30 | xlf    | -0.00394732664819592827 | -0.00924707040410337243
```

This should be straightforward but keep in ming that your next day returns, the value your trying to predict, will be off by one day.

## Second Pass // Apply algorithm

```sql
SELECT * 
, regr_slope(next_day_ret, x) OVER deriv_wdw AS slope
, regr_intercept(next_day_ret, x) OVER deriv_wdw AS intercept
FROM
(
    ...
) AS a
WINDOW deriv_wdw AS 
    (PARTITION BY ticker ORDER BY dt ROWS BETWEEN {periods} PRECEDING AND 1 PRECEDING)
```

This is when the algorithm gets applied. We create a window function with a periods variable that will inserted via string formatting in Python (don't do this in production). This allows us to easily test different time periods without duplicating code. The aggregate functions `regr_slope` and `regr_intercept` are native to Postgres and takes the next day's return as our dependent value and the returns for `n-period` days as the independent values. 

The window function allows us to apply the regression on a rolling basis. The tricky bit in the setting the windows is `ROWS BETWEEN {periods} PRECEDING AND 1 PRECEDING`. If `periods` was 5, Postgres would use the returns between the 5 rows before the current row and 1 row preceding the current row. 


```text

     dt     | ticker |             x              |        next_day_ret        |         slope         |       intercept       
------------+--------+----------------------------+----------------------------+-----------------------+-----------------------
 1998-12-22 | xlf    |                            |     0.01474535247145115037 |                       |                      
 1998-12-23 | xlf    |     0.01474535247145115037 |     0.00660497782213908891 |                       |                      
 1998-12-24 | xlf    |     0.00660497782213908891 |    -0.01312336068227701268 |                       |                      
 1998-12-28 | xlf    |    -0.01312336068227701268 |     0.01063829877772755555 |      2.42351726478379 |   -0.0291306384677451
 1998-12-29 | xlf    |     0.01063829877772755555 |    -0.00394732664819592827 |    -0.341346007410265 |   0.00230938638249899
 1998-12-30 | xlf    |    -0.00394732664819592827 |    -0.00924707040410337243 |    -0.375331447256747 |   0.00181332943918483
 1998-12-31 | xlf    |    -0.00924707040410337243 |     0.00000000000000000000 |    -0.207362468032042 |  -0.00119621198347397
 1999-01-04 | xlf    |     0.00000000000000000000 |     0.00933337679644815330 |    -0.629137865809961 |  -0.00427771173025455
```

### Window Function Intermission

It's important that you get your head around how window functions calculate rolling aggregates so let's take a look at a trivial example.

```sql
SELECT *
, SUM (value) OVER (ORDER BY row_number ROWS BETWEEN 3 PRECEDING AND 1 PRECEDING) AS rolling_sum
FROM
(
select 
  generate_series(0,19) as row_number, 
  generate_series(1,20) as value
) a

 row_number | value | rolling_sum 
------------+-------+-------------
          0 |     1 |            
          1 |     2 |           1
          2 |     3 |           3
          3 |     4 |           6
          4 |     5 |           9
          5 |     6 |          12
          6 |     7 |          15
          7 |     8 |          18
          8 |     9 |          21
          9 |    10 |          24
         10 |    11 |          27
```

The rolling sum at row_number 4 is 9 which is the sum of the values at row numbers 1, 2, 3. Let's take a look at what happens when we use `CURRENT ROW` rather than `1 PRECEDING`. 

```sql
SELECT *
, SUM (value) OVER (ORDER BY row_number ROWS BETWEEN 3 PRECEDING AND CURRENT ROW) AS rolling_sum
FROM
(
select 
  generate_series(0, 19) as row_number, 
  generate_series(1, 20) as value
) a

 row_number | value | rolling_sum 
------------+-------+-------------
          0 |     1 |           1
          1 |     2 |           3
          2 |     3 |           6
          3 |     4 |          10
          4 |     5 |          14
          5 |     6 |          18
          6 |     7 |          22
          7 |     8 |          26
          8 |     9 |          30
          9 |    10 |          34
         10 |    11 |          38
```

Here the rolling sum at row number 4 is 14 which is the sum the values at row numbers 1, 2, 3 **and** 4. The important thing to remember is that when you use `n precediing`, it starts aggregating values from `n periods` ago rather than the last `n` values.

!> Use `1 PRECEDING` rather than `CURRENT ROW` to avoid data snooping. If you see results that look too good to be true this is the first place to look when debugging. 


## Third Pass // Make predictions

```sql
SELECT
' {periods}_day_regression' as model_name
, dt
, (x * slope) + intercept AS pvalue
, next_day_ret
FROM
(
    ...
) b;
```

Here we calculate our predictions with the regression slope and intercept calculated in the second pass and name it `pvalue`. Now that we have our predictions for the next day returns and the actual next day returns, let's take a look at a scatter plot of the values.

#### Predictions vs Next Day Returns plot

![pvalue vs next day returns](https://raw.githubusercontent.com/Eurekoio/research/master/regression/001/pvalue_v_next_day_ret.png)

It's hard to see any predictive relationships in these charts because of a couple outliers. Let's generate scatterplots without the outliers to get better resolution.


#### Filtered Predictions vs Next Day Returns

![pvalue vs next day returns filtered](https://raw.githubusercontent.com/Eurekoio/research/master/regression/001/pvalue_v_next_day_ret_filtered.png)

That's better. The increased resolution doesn't provide us with obvious predictive relationships but it allows us to make a couple observations. The volatility of predictions decrease as we add more periods in the training data set. Intuitively, this should make sense since 5 data points to train a model would result in a brand new data set every 5 days and the volatility of that period would be fully reflected in the model. Another observation we could make is that if a trading strategy does work, it will probably work better with the 5 day regression since there's more 'spread' in the relationship of predictions to actual returns.


# Backtesting

## Backtesting vs Modeling

Most algorithmic trading tutorials would have you split your data into a training set and a test set. You use the training set to train your model then you apply that trained model to your test data. As an example, assume that you have two years worth of daily closing prices for Google. Let's say 2015 and 2016. These examples would have you train on 2015 data and test on 2016 data. 

Although it's not a terrible methodology, it doesn't reflect real life. Why assume your only going to retrain your model once a year in 2017? Most modern computers have enough computing power to easily retrain models on a daily basis. In order to reflect real life trading situation, you have to perform a rolling backtest where you retrain your model for the next period. At the end of the next period, you retrain your model again for the following period and so on. 

Historically, there was been no easy to perform a rolling backtest. You would have needed to write your own software application and you would probably have to build out new 'features' for every new idea you wanted to test. This is a lot of work just to see if a little regression model works in real life. Luckily, we have [window functions](https://robots.thoughtbot.com/postgres-window-functions) in PostgreSQL.


## Backtesting Code

```python
def query(periods):
    sql = """
    SELECT *
    , SUM(pnl) OVER (ORDER BY dt) AS cum_pnl
    FROM
    (
        SELECT
        ' {periods}_day_regression' as model_name
        , dt
        , CASE 
            WHEN (x * slope) + intercept > 0 
                THEN  next_day_ret
            WHEN (x * slope) + intercept < 0 
                THEN -1 * next_day_ret
            ELSE NULL
            END AS pnl
        FROM
        (
            SELECT * 
            , regr_slope(next_day_ret, x) OVER deriv_wdw AS slope
            , regr_intercept(next_day_ret, x) OVER deriv_wdw AS intercept
            FROM
            (
                SELECT 
                feature_0.dt
                , feature_0.ticker
                , feature_0.ret AS x
                , dep.ret AS next_day_ret
                FROM xlf_returns feature_0, xlf_next_day_returns dep
                WHERE feature_0.dt = dep.dt
            ) AS a
            WINDOW deriv_wdw AS 
                (PARTITION BY ticker ORDER BY dt ROWS BETWEEN {periods} PRECEDING AND 1 PRECEDING)
        ) b
    ) c;
    """
    return sql.format(periods=periods)


day_5_df = pd.read_sql_query(query(5), con=conn)
day_20_df = pd.read_sql_query(query(20), con=conn)
day_50_df = pd.read_sql_query(query(50), con=conn)
day_250_df = pd.read_sql_query(query(250), con=conn)

frames = [day_5_df, day_20_df, day_50_df, day_250_df]
big_frame = pd.concat(frames)


plot_title = "XLF Cum PNL"
p = ggplot(aes(x='dt', y='cum_pnl'), data=big_frame)
g = p + geom_line() + \
    facet_wrap("model_name") + \
    ggtitle(plot_title)

t = theme_gray()
t._rcParams['font.size'] = 10

g + t
```

## Backtesting Results

![xlf backtesting results](https://raw.githubusercontent.com/Eurekoio/research/master/regression/002/xlf_simple_trade_cum_pnl.png)


## Performance Metrics




