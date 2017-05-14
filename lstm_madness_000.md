# 000 / Init

## LSTM + XLF

For this example we're going to set up a recurrent neural network on XLF price data. The XLF is an ETF that mirrors a basket of financial stocks. We could use the SP500 or an ETF that tracks the SP500 but it's always easier to work an ETF with less components.

For our neural network, we'll be using Long Short-Term Memory ("LSTM"). I'll let [other](http://colah.github.io/posts/2015-08-Understanding-LSTMs/) [sites](https://deeplearning4j.org/lstm) walk you through the wonders of LSTM. For our purposes the important thing you need to know is that they not only allow you to train on features but also allows you to train on the time series of those features as well. Sounds like a handy tool for financial data. 

## Initializing Data

Look at the previous posts about how to get data from Yahoo and use that to create a script to get the historical data from the XLF and each of its components. Don’t know the XLF components by heart? You can find them right here.

If that’s too much for you, you can just run `python main.py` in `/get_yahoo_data` of this [repo](https://github.com/Eurekoio/research/tree/master/lstm-madness). You’re going to have to run it twice, once for the components and once for just the XLF price series. Save them to different buffers because we’re going to copy them to different tables.

### In the beginning

So that we’re on the same page, let’s create a new user and database.
```sql
CREATE USER ventris_admin WITH PASSWORD 'X4NAdu';
ALTER ROLE ventris_admin WITH SUPERUSER;
CREATE DATABASE ventris WITH OWNER ventris_admin;
```
Now you should be able to login into your database like so:
```
psql -d ventris -U ventris_admin -h 127.0.0.1
```
If you followed the instructions above, you should have two files, one with just the XLF price data and the other with the price data for all of its components. Let’s create two tables to copy the data into.
```sql
DROP TABLE IF EXISTS xlf_components_data;
CREATE TABLE xlf_components_data (
  ticker text
, dt date
, open numeric
, high numeric
, low numeric
, close numeric
, volume numeric
, adj_close numeric
);

COPY xlf_components_data FROM '<YOUR FILEPATH HERE>' WITH DELIMITER ',';

DROP TABLE IF EXISTS xlf_etf_data;
CREATE TABLE xlf_etf_data (
  ticker text
, dt date
, open numeric
, high numeric
, low numeric
, close numeric
, volume numeric
, adj_close numeric
);

COPY xlf_etf_data FROM '<YOUR XLF FILEPATH HERE>' WITH DELIMITER ',';
Calculate Returns
```
Now that we have our price data loaded let’s derive a table of returns for each of these tables. You calculate the return by taking today's close - yesterday's close and dividing the result by yesterday's close. Calculating the returns kills two birds with one stone. It normalizes our data and provides us with a table to help us calculate how much money we made.
```sql
-- Create returns based on the days's close
DROP TABLE IF EXISTS xlf_components_returns;
CREATE TABLE xlf_components_returns AS
SELECT
  dt
, ticker
, CASE
    WHEN LAG(close, 1) OVER (PARTITION BY ticker ORDER BY dt) > 0
      THEN (close - LAG(close, 1) OVER (PARTITION BY ticker ORDER BY dt)) / LAG(close, 1) OVER (PARTITION BY ticker ORDER BY dt)
    ELSE NULL END AS ret
FROM xlf_components_data;

-- Create returns of xlf
DROP TABLE IF EXISTS xlf_returns;
CREATE TABLE xlf_returns AS
SELECT
  dt
, ticker
, CASE
    WHEN LAG(close, 1) OVER (PARTITION BY ticker ORDER BY dt) > 0
      THEN (close - LAG(close, 1) OVER (PARTITION BY ticker ORDER BY dt)) / LAG(close, 1) OVER (PARTITION BY ticker ORDER BY dt)
    ELSE NULL END AS ret
FROM xlf_etf_data;
```
This should be pretty straightforward. If Postgresql’s window functions are new to you, you should stop and read up on them. Window functions are indispensable for backtests and is really the most under appreciated feature of Postgresql.

The last thing we have to do is create a table with XLF’s next day’s returns. This will be the Y value for fitting our models and is just like our returns table except the returns are offset by one day. For example, our XLF returns table looks like this:
```sql
dt     | ticker |           ret           
------------+--------+-------------------------
1998-12-22 | xlf    |                        
1998-12-23 | xlf    |  0.01474535247145115037
1998-12-24 | xlf    |  0.00660497782213908891
1998-12-28 | xlf    | -0.01312336068227701268
```
While our xlf_next_day_returns table looks like this:
```sql
dt     | ticker |           ret           
------------+--------+-------------------------
1998-12-22 | xlf    |  0.01474535247145115037
1998-12-23 | xlf    |  0.00660497782213908891
1998-12-24 | xlf    | -0.01312336068227701268
1998-12-28 | xlf    |  0.01063829877772755555
```
We could create these returns when we do our SQL calls but I find that it’s cleaner to just create a table with those returns:
```sql
DROP TABLE IF EXISTS xlf_next_day_returns;
CREATE TABLE xlf_next_day_returns AS
SELECT
  dt
, ticker
, CASE
    WHEN LEAD(close, 1) OVER (PARTITION BY ticker ORDER BY dt) > 0
      THEN (LEAD(close, 1) OVER (PARTITION BY ticker ORDER BY dt) - close) / close
    ELSE NULL END AS ret
FROM xlf_etf_data;
```
Now that you can follow along at home, let’s build our first neural network!


### Tomorrow never knows

For our first neural network, we'll try to answer the most obvious question:

> Can today's return predict tomorrow's return?

Let's break this down to actionable tasks:

1. Get today's returns for our x values.
2. Get the next returns for our y values.
3. Shape the data for Keras
4. Setup, compile and fit the model.
5. Make predictions on our existing data.
6. Plot the predicted returns to the actual returns.

We're going to ignore applying our model to test data while we try to figure out what all the levers do in the neural network cockpit.

## Data Shaping

> You can skip to the next post if you want to jump right into the LSTM model.

Many machine learning examples completely ignore the process of shaping data which is the process of take raw data and shaping it so it's in a format that can be read by our machine learning tools. Often times, this takes much longer than actually setting up and running the machine learning process. Since every data source is slightly different this process includes a lot of trial and error.

After you've imported your dependencies and connected to your database,
let's query our database and get the XLF returns. We're going use `pandas` for this so our results will be returned in a `pandas dataframe`.

```python
x_query = """
SELECT
  dt
, ticker
, ret
FROM xlf_returns
WHERE dt > '1998-12-22'
AND dt < '2017-02-28'
ORDER BY dt
"""

x_df = pd.read_sql_query(x_query, con=conn)
```

If you type `x_df` on the command line you should have something that looks like this:

```python
>>> x_df
              dt ticker       ret
0     1998-12-23    xlf  0.014745
1     1998-12-24    xlf  0.006605
2     1998-12-28    xlf -0.013123
3     1998-12-29    xlf  0.010638
4     1998-12-30    xlf -0.003947
...          ...    ...       ...
4568  2017-02-21    xlf  0.004904
4569  2017-02-22    xlf  0.000813
4570  2017-02-23    xlf  0.000000
4571  2017-02-24    xlf -0.007720
4572  2017-02-27    xlf  0.005323

[4573 rows x 3 columns]
```

Before we go on, it's helpful to see what our dataset ultimately needs to look like.

```python
>>> dataset_x
array([[ 0.01474535],
       [ 0.00660498],
       [-0.01312336],
       ...,
       [ 0.        ],
       [-0.00772048],
       [ 0.00532346]], dtype=float32)
```

We need to take our dataframe and shape it into a `numpy.array` of datatype `float32`. This seems pretty straightforward, right? Nope. My first attempt at converting this was to just take `x_df.ret` and convert it to `floast32`. I ended up with this:

```python
test = x_df["ret"].astype('float32')

>>> test
0       0.014745
1       0.006605
2      -0.013123
3       0.010638
4      -0.003947
5      -0.009247
6       0.000000
          ...   
4569    0.000813
4570    0.000000
4571   -0.007720
4572    0.005323
Name: ret, dtype: float32
```

It got the data type right, `float32`, but it's just a column of numbers. It's not even an array. We should be able to easily convert that to an array.

```python
test_array = numpy.array(test)

>>> test_array
array([ 0.01474535,  0.00660498, -0.01312336, ...,  0.,
       -0.00772048,  0.00532346], dtype=float32)
```

Now we have an array with the right datatype but if you go back and look at what the final dataset needs to look like we actually need an array of arrays. This makes sense when you consider that we would need to be able to use array of past returns as X values to feed into our model. At this point, I would create a `for loop` to create an array for each value in the array but I inadvertantly ran into another solution. 

```python
x_df_wide = x_df.pivot(index="dt", columns="ticker", values="ret")


>>> x_df_wide
ticker           xlf
dt                  
1998-12-23  0.014745
1998-12-24  0.006605
1998-12-28 -0.013123
1998-12-29  0.010638

dataset_x = x_df_wide.values.astype('float32')

>>> dataset_x
array([[ 0.01474535],
       [ 0.00660498],
       [-0.01312336],
       ..., 
       [ 0.        ],
       [-0.00772048],
       [ 0.00532346]], dtype=float32)
```

Here we take the tall dataframe and make it wide by pivoting the dataframe. With only one ticker, we end up with a dataframe that has the dates as the index and XLF as the column with its returns as row values. If we had more than one ticker, we would have a separate column for each header. For some reason, when you convert wide dataframe's values to a numpy array you end up with the shape we need for our model. We also get the added bonus of having figured out how we use additional pull in and shape data with more than one feature. Here's a quick summary of where we're at.

```python
# We're using date ranges to avoid `null` values.
# Get x data

x_query = """
SELECT
  dt
, ticker
, ret
FROM xlf_returns
WHERE dt > '1998-12-22'
AND dt < '2017-02-28'
ORDER BY dt
"""

x_df = pd.read_sql_query(x_query, con=conn)
x_df_wide = x_df.pivot(index="dt", columns="ticker", values="ret")

# Get y data

y_query = """
SELECT
  dt
, ticker
, ret
FROM xlf_next_day_returns
WHERE dt > '1998-12-22'
AND dt < '2017-02-28'
ORDER BY dt;
"""

y_df = pd.read_sql_query(y_query, con=conn)
y_df_wide = y_df.pivot(index="dt", columns="ticker", values="ret")

dataset_x = x_df_wide.values.astype('float32')
dataset_y = y_df_wide.values.astype('float32')
```

The next bit of code splits our dataset so that 67% will be used to train our network while the remainder will be used to test its predictions. Confusingly, you also need to predict against your training data to see how well the model fits your training data. I'll cover more on this later. 

```python
train_size = int(len(dataset_x) * 0.67)
test_size = len(dataset_x) - train_size
stage_train_x, stage_test_x = dataset_x[0:train_size], dataset_x[train_size:len(dataset_x)]
train_y,  test_y = dataset_y[0:train_size], dataset_y[train_size:len(dataset_y)]
```

The last thing we have to do is add the time step to the shape of the X value datasets. A numpy array is an object with the property `shape`. `stage_train_x.shape` returns `(3063, 1)` which reflects the number of objects in our array, 3063, and the number of items in each object, 1. That's great but the LSTM model also needs a value for the number of time steps or the lookback period of our data. Since we're only using one day's worth of data, our time step value is 1 and so we'll add that to the `shape` property.

```python
train_x = numpy.reshape(stage_train_x, (stage_train_x.shape[0], 1, stage_train_x.shape[1]))
test_x = numpy.reshape(stage_test_x, (stage_test_x.shape[0], 1, stage_test_x.shape[1]))
```

We only need to do this for the X values.

Now that we have the right shape, we can finally start building our model.


## Model time

Let's build our model.

```python
number_of_features = train_x.shape[2]

model = Sequential()
model.add(LSTM(1, input_dim=number_of_features))
model.add(Dense(1))
model.compile(loss='mean_squared_error', optimizer='adam')
model.fit(train_x, train_y, nb_epoch=100, batch_size=1, verbose=2)
```

A neural network requires at least three layers. You always need an input layer and an output layer. The layers in between are the hidden layers where the neurons do their work. Read through [this](http://stats.stackexchange.com/questions/181/how-to-choose-the-number-of-hidden-layers-and-nodes-in-a-feedforward-neural-netw) answer on Stack Overflow for a great walkthrough of the layers. 

In our example, as far I can tell, after we instantiate our model with `model = Sequential()` we're adding the input layer and a hidden layer by calling `model.add(LSTM(1, input_dim=number_of_features))`. The first parameter in `LSTM` is the number of neurons for that layer while `input_dim` tells the model how many features we have. Our output layer is called by the `Dense` function which has a output of 1 dimension. 

After our model is set up, we run `fit`, with our X values and Y values. `nb_epoch` is the number of times we we pass over the the data while `batch_size` is the number of samples (X values) the model uses at one time. For the sake of argument, let's say we have 2,000 X values. Using a `batch_size` of 1 means the model will iterate over the X values 2,000 times for each epoch. If our `batch_size` was 200, it would only iterate over the X values 10 times for each epoch.

All of these parameters are referred to as hyperparameters in neural network speak. In machine learning, when you see hyper-anything means it's going to be like looking at the McDonald's menu after the they started serving breakfast all day. More parameters are great but they add the potential for cognitive overload which may ultimately lead to overfitting your data and a bad headache.

### The Fit Bit

```python
# make predictions
train_predict = model.predict(train_x)
test_predict = model.predict(test_x)

# Add prediction to y_df_dataframe
y_df_train = y_df[0:train_x.shape[0]]
y_df_train["prediction"] = train_predict
y_df_train.to_sql('lstm_madness_xlf_train_01', engine) 
```

Here we take the trained model and predict Y values with our training data. We then take those results, line them up with the our actual Y values and save that dataframe as a table in our database. We'll use these tables to see how well our trained models can predict the values it was trained with. 

### Plots

For each iteration, I'm going to generate two initial plots. 

1. A scatter plot comparing the predicted return to the actual return.
2. A line plot comparing the cumulative actual returns and the cumulative predicted returns.

My preference is to iterate quickly so I usually take a quick look at these two plots and just jump into another iteration. I'll only start running other reports and plotting those as well if something interesting comes up. 

Let's take a look at what we got. 

![Scatterplot](https://raw.githubusercontent.com/ventrisco/lstm-madness/master/01/scatter.png)

In an ideal world, what you want to see in this scatterplot is a linear line that goes from the lower left quadrant to the upper right quadrant. This would mean that our trained model perfectly predicted the actual return. We don't live in an ideal world and this scatter plot shows that our model doesn't make very good predictions. It looks like the majority of its predictions are negative. 

Let's take a look at our cumulative return plot. 

![Cumulative Returns](https://raw.githubusercontent.com/ventrisco/lstm-madness/master/01/cum_ret.png)


Yup, the majority of those predictions are negative. The red line represents the sum of the XLF returns that results in a chart that looks like the actual price chart for the XLF. The financial markets have had some rough spots over the past 18 years and the XLF's actual cumulative returns reflects that. Our predictions on the other hand are consistently negative and looks more like a random negative drift than a series of hard predictions. The spike of activity in the 2009 may be related to better "edges" that the model was able to detect due to increased volatility during that year.

Our first model was a bust. The predictions of our trained model against the trained data is so lame, it doesn't make any sense to use our trained model on out of sample data. The trained model, however, shows that increased volatility results in more prediction activity. We'll explore how we can use this in our next iteration. 


## Hyper Time

So our initial model was a bust. You should embrace that because in research knowing what doesn't work is just as important as figuring out what does. Let's start messing with some hyperparameters.

The first thing that I can think of tweaking is the number of epochs. I'm going to increase epochs from 100 to 1000. We run the risk of overfitting the model but at this point it's better than a model that doesn't fit all. Besides, we'll test for overfitting later.

Since we're going to increase our epochs to 1000, I'm going to increase our batch size to 32 in order to decrease the run time. Our initial model ran 100 epochs with a batch size of 1. It took about 14 seconds to run each epoch on my machine. That's because it's only fitting 1 data point for each epoch. Why 32? It's an arbitrary number based on some examples on Keras' site, don't read anything into it.

### That's Interesting

Here's the cumulative return of our first model.

![Cumulative Return](https://raw.githubusercontent.com/ventrisco/lstm-madness/master/01/01/cum_ret.png)

Here's the scatterplot of our predicted returns vs the actual returns for our train data. 

![Scatter](https://raw.githubusercontent.com/ventrisco/lstm-madness/master/01/01/scatter.png)

This is much better than our first model since it actually "fits" a linear line to some trends in the price. However, it's odd that it catches those trend changes pretty late even on the train data. The scatterplot is interesting in that it seems as if the larger predictions are downside predictions. If you look along the X-axis you can see that the actual returns for those downside predictions are pretty random. Given that these plots are of the model trying to fit the training data, I would have thought the predictions would be random as well.

Before we move one, let's take a look at the profit and loss.

![Cumulative Profit](https://raw.githubusercontent.com/ventrisco/lstm-madness/master/01/01/cum_profit.png)

Given that this ignores trading costs, it would be like Chinese Water torture trading this model from 2003 to 2008. Since the profits are based on our training data we shouldn't get too excited but a 250% return shows that the model is moving in a positive direction.

