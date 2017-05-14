# Intro

It should come as no surprise that data management is an integral part of analyzing financial data. Not only do you have to manage raw data such as daily stock quotes, you also have to deal with data inflation from extracting features from the data.

However, unlike other machine learning endeavors, we have the luxury of working with structured data sets that have been vetted by third parties.

The basic pattern for managing data goes like this:

1. Fetch data
2. Parse Data
3. Load Data

For this simple example we'll fetch End of Day ("EOD") data from Yahoo Finance, parse it and load it into our database. We'll be using Python as our programming language and Postgresql as our database so make sure you have those installed. Let's start fetching some data!