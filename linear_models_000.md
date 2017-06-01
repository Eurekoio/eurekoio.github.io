# Linear Models

A linear regression is seldom used as a first example for algorithmic trading. Usually, variations of trading signals derived from moving averages are the de facto starting point for learning about algorithms. Moving averages are easy to calculate and relatively easy to get your head around. However, Linear regression isn't that much harder to understand and it easily lends itself to more complicated trading strategies.

I won't go deep into what linear regression modeling is but you can check out these [sources](https://en.wikipedia.org/wiki/Linear_regression) [for](https://onlinecourses.science.psu.edu/stat501/node/250) a [quick](http://www.stat.yale.edu/Courses/1997-98/101/linreg.htm) [refresher](https://www.youtube.com/watch?v=ZkjP5RJLQF4). For our purposes, we'll just focus on a simple linear regression model with two variables. A classic example is one where you observe the weight and height of, let's say, 30 people. Intuitively, you may guess that taller people are generally heavier with a couple outliers here an there and your observations look like they reflect that. But how do you quantify the relationship.

That's where linear regression comes in. When you run a linear regression you're basically drawing a line through your observations that best fits those observations. The concept is simple but there are a couple gotchas to be aware of. 

First, outliers can change the slope of your model. In our height and weight example think of what would happened if you met somebody who was 6'5" but only weighed 80 pounds. This dovetails into a second gotcha which is that when you run a regression, it's going to try to fit a line whether your data supports that or not. 

## Our first model

There are a variety of ways to apply a linear regression model to financial data but we'll keep it simple. We're going to use the an `n-period` days worth of returns of a stock to predict returns for the following day. If we were running a linear regression on a stock today,our independent variable will be the previous `n-period` returns while our dependent value, the value we're trying to predict, will be tomorrow's return. It's important to get this right in your head because it's easy to get tripped up when you're writing code.



## Code Init

> If you haven't done it yet, make sure you've initiated your XLF repo.
> The code for this section is available [here](https://github.com/Eurekoio/research/tree/master/regression/001)

The first thing I like to do when approaching a new security with a new algorithm is to generate a scatterplot to see how our prediction copmares to the actual return. You should also take a look at the price chart to get an idea of what you're dealing and run an autocorrelation to see if there's anything interesting. 

Here's the code to build our regression model to make our predictions. There's a bit going on here so I'll walk through the code after you digest it for a bit. 

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

The first thing you may notice is that most of code is SQL. In fact, most of the work is done in PostgreSQL three passes of our data. Peforming a backtest in the database saves us a network trip, which may take a while once your dataset gets larger, while keeping our data organized. I find that it's easy to get tripped up with dates resulting in data snooping. We'll cover this a bit later but data snooping is when you use variable that could not have existed at the point in time that it was used in your analysis. It's bringing the future into forward. 

## Code walkthrough


![pvalue vs next day returns](https://raw.githubusercontent.com/Eurekoio/research/master/regression/001/pvalue_v_next_day_ret.png)




