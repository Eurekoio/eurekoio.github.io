# Introduction

This is a start to finish walkthrough for backtesting a complete algorithmic trading strategy using open source tools. The methodology and tools are derived from work I did a couple years for a "Big Data" start up. Although the start up was focused on text mining, we researched the financial markets for kicks. That start up blew up a couple years back but I recently went back to look at some of the research and open it up to a wider audience.

## Data Democratization

With the democratization of financial data and access to markets, the number of free resources available is mind boggling. I started in finance in 1993 and I painfully recall having to pay for financial data that arrived on floppy disks. Lot of floppy disks. I could never effectively analyze that data since you would have to build all of your analytical tools from scratch. The only language you would have probably used was C and I didn't know how to compile a `hello world` is C, let alone write my own statiscal analysis library.

Even within the 5 years since I researched the financial markets, the availability of free financial data and grown exponentially. Five years ago, we bought historical financial data and ping Yahoo Finance for stock data. Now services like Quandl a plethora of financial data available through nice APIs accessible through language specific libraries.

## 3 things

When deciding to trade a financial instrument you really only need three pieces of information.

1. Selection
2. Direction
3. Size

### Selection 

`Selection` refers to figuring out what security you're going to trade. Some professionals have this decision made for them by their bosses. You may be in the enviable situation where you want to trade a stock because your company went public and you want to manage your exposure to one stock. For the rest of us, we choose stocks based on our own experiences. 

For this walk through, I'm going to take a look at the XLF ETF. An ETF is a stock who's price reflects a basket of stocks and the XLF ETF consists of a basket of financial services stocks. Most ETFs provide you with the ability to gain diversified exposure to a sector by buying one stock instead having to buy an entire basket of stocks in the that sector. 

Some of the most liquid ETFS, the SPY and QQQQ, consist of the stocks in the S&P 500 and the Nasdaq 100.

### Direction

`Direction` refers whether you buy or sell a stock. I'm guessing 97.8% of the research or HOW TOs I've seen on the internet focuses on direction. This is a binary event: you can either buy (go long) or sell (go short). When you buy a security you make money when the price goes up. If you short a stock, you make money when the stock goes down. 

You would think that it would be easy to predict a binary event but security prices are essentially random. As a result, you're looking for really subtle tradeable signals that provide you with an edge that's hopefully better than monkey throwing darts at a wall of ticker symbols. 

This is really where algorithms come into play and fair amount of time will be spent on analyzing data to find an algorithm that "works". 

## Size

`Size` refers to how much you should allocate to a certain trade. If you've ever gambled at a casino, you've thought through this. If you walked into the casino willing to lose $1000, you have decide how much of it you bet at any given time. You could bet it all at one time but you might lose it all. Alternatively, you could bet as little as possible but that gets boring after a while. Wouldn't it be cool if there was a formula to calculate optimal bet size? Lucky you, there is and I'll show you how to use it and backtest it.

# A Road Map

In order to research the 3 things, we need to set up tools for the following: 

1. Data Management
2. Algorithms
3. Backtesting

## Data Management

Data management is the process of retrieving and managing data. Lucky for us, financial data is well structured so we don't have to deal with crazy parsing issues. Having said that data management is time consuming and tedious but is probably the most important part of this process. If you have bad data, you're algorithms will go nuts or worse will give you false signals.

For this walk through, I'll show you how to use Python to scrape data from Yahoo Finance and use Quandl's library to access its API. 

## Algorithms

This is really the fun part. Algorithms are basically a set rules that encapsulate a trading strategy. You can come up with your own algorithms based on your observations or theories about the financial markets. A tried a true algorithm is the moving average and might go something like this: 

* Buy the stock if its price is above the 10 day moving average of its price.
* Sell the stock if its price below the 10 day moving average of its price. 

For the most part, humans come up with these algorithms by analyzing data, coming up with a hypotheis, testing the hypothesis and iterating through the cycle again until they come up with tradeable signals. For this walkthrough, I'm going to do something a little different and see if we can use a neural network to help with coming up with our algorithms. 

With the release of open source neural network tools like TensorFlow, implementing a neural network has become trivial. Deep learning is the new hotness so let's take it out for a sping. It's also novel to me so I thought it would be interesting to see if it would yield anything interesting. The research cycle should be the same so you'll to able to apply the methodology when testing your own algorithms.

## Backtesting

So you have your data and an algorithm that seems to work on your training data but how do you know if it actually works? Much of the literature on testing algorithms will have you split your data between a set to train and a set to test. This is good practice but doesn't really reflect real trading conditions.

Let's look at a quick example. If you have 1000 days worth of financial data, you might use 700 days to train your algorithm and use the remaining 300 days to test your algorithm. By taking this approach you are assuming that you never retrain your algorithms. Retraining your algorithms may not be ideal but you should backtest it to see it what affect it might have on your final results. 

Historically, peforming this analysis was time consuming, network IO heavy, and error prone. I'll show you how to use open source databases to reduce potential errors and network IO. 
