# Python Twitter SATrader
This is a just-for-fun project implementing a trading strategy on the QuantConnect platform, and my first experience with building a trading bot more broadly. It goes without saying that this bot is not to be used for live trading, unless you have more money than sense. 

I have chosen to use Python because of its many excellent data-related libraries, such as Pandas and Numpy, and the language's relative slowness is not a concern, given the strategy used. Additionally, Python is the language I am most comfortable with and there is an extensive support community for algorithmic trading using Python and its libraries.

The logging is mostly commented out because of log limits on QuantConnect, so do uncomment these at your discretion. 

## Alpha model


In QuantConnect, you have the ability to add custom data, i.e. social media data, weather data, SEC filings, news data or whatever data you can find (well, in tandem of iteration over similar data to train bot, it is technically ‘dataset’). Even though QuantConnect already offers a huge variety of different data, adding your own data gives you even more possibilities. To demonstrate this feature I created a skeleton trading bot that trades Tesla stock based
on simple sentiment analysis of Elon Musk's tweets.

addData() - takes three arguments.
the type of data that you want to add. Usually you would have to create a custom class for this and use this class as the type.
string which represents the ticker for that data.
resolution (spread of data contained within a period) which specifies how often the custom data should be polled. Ex: you specify a minutely resolution, the algorithm will check for the new data once a minute.

You might be able to see that the addData() is very similar to the addEquity().
In fact they both fundamentally do the same thing, the addData() is just a little
more general.
 
getSource() - used to get the source of that data
reader() - used to read your custom data.
(Since the data is custom and thus unstandardized, you need to specify the structure of that
data in reader())

Example With Custom Weather Data

Parameters of subscriptable data object we must create - (actual source of your data, subscription transport medium)

actual source of your data - [i.e., an URL that links to a raw .csv file]
subscription transport medium - Four options:
remote file which means that the algorithm expects one file with all the data in a remote location such as your Dropbox.
local file if the data is accessible locally.
options rest, which is used if each line of the custom data should be polled individually and comes from a rest call. This might be the case if you have a subscription to a data vendor that provides you with custom data in real time.
streaming

Let’s say we have custom weather data in a Dropbox.
The URL to this data is specified as the source and for the subscription transport medium
we choose a remote file.
In some cases you might want the getSource() to return different data depending
on whether you are backtesting your algorithm or actually live trading it.

If for example you have a real time data subscription, you can't use that for backtesting and might instead want to use a CSV file for the historical data when you are backtesting.

For this you can use the isLive flag which is true when you are live trading and false
if you are just backtesting.

Now the data specified in the CSV file will get sent to the reader() line by line. The goal of the reader() is now to standardize this data so that your algorithm understands what it looks like and can actually use it.

It would return, for example,  a chart containing x entries divided into x columns.

Let’s say it returns a template chart with 11 entries divided into 4 columns.
Column A specifies the data for each row.
The first 4 digits specify the year, the next 2 the month and the last 2 specify the day.
Column B represents the max temperature in degrees Celsius for that day.
Column C represents the daily average and column D is the minimum temperature.

Simple stuff.

Theoretically you could use weather data such as this one for trading in the agricultural
space since this is heavily weather dependent. Here it might also be useful to take precipitation, the weather forecast and other variables into Account. That said, you now hopefully have a good understanding of how this custom weather data is structured.

*Remember that this data now gets fed to the reader() line by line.*

The first thing we want to do in the reader() is check whether a meaningful line
data was passed to it. We do this by checking if the line of data is not empty and that the first value is a digit. This ensures that we don't try to use the data from the header index (where the ascription is to tracking data, not modeling data) and don't continue if there is no data left. So if this is not fulfilled, we just return an empty object. Otherwise, we split the line at its commas and save the generated list to the data variable.

*Note that we can split at commas since the file is a CSV file which stands for comma
separated values.*

Then we create an object of the weather class.

Since this class extends the Python data class there are three important properties that
we should set, namely the symbol, value and time property.

For the symbol property we should always use config.symbol.

For the time, we need to correctly format the values from the time column in the CSV file.
We can do so with DateTimeHelper() in which you can specify the format of the
date and time. Here it is very important to understand that data in QuantConnect is accessed by end time and usually the data and time in a custom file is the start time.

Hence, it's essential to add a time delta to the start time.

If you don't do this your algorithm might access this data before it actually would
be known aka look-ahead bias, which can lead to very unrealistic results.

Let’s say we add a time delta of 20 hours to the time of the weather data. If we would not do this, the algorithm could use the weather data for a given day at the beginning of a day in the future, a day null in resolution. 

Next up we set the value attribute of the weather object. For this we will take the daily average temperature which was saved into the second column of the weather data file. 

*Note that the value attribute always is a decimal object.* 

We also create two more properties for the max and min temperature that were saved in the first and third column. Last but not least we return this weather object.

Since custom data can sometimes lead to some unexpected errors it usually is a good idea
to add some exception handling to the reader().

Success! You can now access this custom weather data like you would access other price data.

Inside of the onData() you can index the data slice object with the symbol of the custom data and then access values either with the getProperty helper or the .value attribute.

Note that if this still seems a little overwhelming to you don't worry too much, we will go over another example step by step when we implement the twitter trading bot.

Besides adding custom data that you want to access on the go, QuantConnect
also allows you to add static data.

If, for instance, you just want to add a file containing a list of tradable symbols for
your universe or an AI training model file, you can accomplish this with the self.download. As an argument you can for example once again just pass a Dropbox download URL.

If the file is a CSV file, you could then use pandas to convert this file to a pandas dataframe.

Note that if you want to use custom data from a data vendor such as Quandl, Tingo or Intrineo,
QuantConnect does offer direct helper methods for this.  For the specifics on how to do this I recommend checking out the documentation page.

Twitter SATrader -

Strategy:

The idea behind this bot is that we will use the tweets from Elon Musk to perform sentiment
analysis and based on the results of this analysis we will then make a trade decision
for Tesla stock.

For this we will only consider those tweets that mention Tesla in some way.

For these tweets we then use a sentiment analyzer to find out whether this tweet has a positive
or negative connotation to it.

If it is positive we establish a long Tesla position for that day (hold).

If it's negative we short Tesla for that day (sell).

Neutral / Kinda Positive / Kinda Negative (NKPKN) are our controls.

For the sentiment analysis we will be using Python's natural language toolkit NLTK package.
Note that if you aren't familiar with this module, don't worry. We will just be using
a pre-trained sentiment intensity analyzer.

But if you are familiar with it, you could try using your own custom models for better
analysis.

Since QuantConnect does not directly offer tweets as data, we will have to find a dataset
containing Elon Musk's tweets and then import this custom dataset so that we can use it
for backtesting this strategy (I found my dataset containing Elon Musk's tweet from 2012 until 2017 for free on Kaggle).

This means we can use this dataset to backtest this strategy at least from 2012 until 2017.
However before we actually import this dataset, I firstly want to perform some pre-processing
on it since we otherwise might run into some unwanted problems.

For the pre-processing I will import this dataset into a Jupyter notebook where I will use Pandas to import it into a data frame. The unprocessed dataset contains 5 columns including values such as retweets, the row number, and a count. But we are actually only interested in two of these columns, namely the tweet itself and its date and time. So firstly we want to remove the three unwanted columns.

Next up, I change the order since we want the most recent tweets to be at the bottom and the least recent ones at the top. 

Besides that we will remove all URLs from the tweets since they can lead to some unexpected
formatting problems and they don't really add any value to our sentiment analysis.

After that I exported the pre-processed data to a new CSV file and uploaded it to Dropbox.
Through this Dropbox URL you can now import the tweet data into QuantConnect where we
will use it to test our strategy.

In QuantConnect we will head over to the lab tab where we will create a new algorithm.
We start with a blank template containing only the initialize and onData()s.

The first thing we do in the initialize() is set the backtesting time frame to the time
covered by the mosqtweets dataset.

So we set the start time to November 2012 and the end time to the beginning of 2017.

The starting balance I will just leave at $100,000.

Import Sentiment Intensity Analyzer from NLTK Sentiment. We will use this sentiment analyzer later to analyze the sentiment of the tweets in a second.

Thereafter we want to add the data that this algorithm will use.

First, we add Tesla data which we can do with the usual addEquityHelper().
Since the tweets can be sent pretty much at any time of the day and we have specific timestamps for them, we will set the resolution for the Tesla data to minutely data.

Then we want to add our custom data.

We can do so with the addData() which takes three arguments.

The first is the type that this data will have.
I will pass mosqtweet for this class that we will implement in a few seconds.

As the second argument you need to specify a ticker symbol that you can use to access
this data with.
For this you can choose whatever name you want to, just try to pick one that isn't
already occupied by some stock or other asset.

I will just go with mosqtweets.

Last but not least you need to specify a resolution.

This resolution will determine how often your algorithm polls your custom data source for
potential new data.

Here I will also go with minutely data.

Since the tweet data actually has timestamps with up to second accuracy, we could also go
with secondly data, but this would slow backtesting down and isn't that important
for now.

*Note that just like for equities and other assets, we can use .symbol to save the symbol
for this data to a variable.*

We will do this so that we can later use this symbol to request the data.

Implement the mosqtweet custom data class to extend the Python data class and implement the getSource and reader()s.

Let's start with the getSource(); but before we do so, save an instance of the NLTK sentiment intensity analyzer into a class variable named self.sia. In the reader(), we will use this for sentiment analysis.

Here we simply want to set a Dropbox URL as the data source
.
*Note that here it is very important to have dl==1 at the end of the URL. ‘dl’ stands for download and we want the algorithm to download this data. If you would want to view the CSV file online you could do so by setting dl equal to 0 and opening the URL in your browser.*

Then, all we have to do is return a subscription data source object(getSource()).
For this object, we will just use the saved URL as the source and specify this to be a remote file.

If you would want to live trade such a bot you would have to connect this bot with some kind of Twitter API that sends you real-time tweet updates. In that case you would use the ‘rest’ or ‘streaming’ subscription transport medium option. To differentiate between backtesting and live trading mode, you could then use the isLive parameter of this method.

But since we will just use this for backtesting we won't have to worry about this for now.

Next up let's move on to implementing the reader() which handles bringing the data
into the right format, so that your algorithm can actually use it in a meaningful way. Remember the reader() receives data from your remote file on a line by line basis. So the first thing we need to check in here is whether the line actually contains relevant data.

If, for instance, we receive an empty line or the first index row, we won't do anything,
which is why we return none. Otherwise we split the line by commas, since the data comes from a comma separated file.

The data variable now is a list containing two elements.

The first element is the date and time of the tweet and the second element is the tweet
content itself.

Next, we create an instance of the muskTweet class and save it to a new variable named
tweet.

The next step is to set the symbol, time and value of this muskTweet object.

For the symbol we can just use config.symbol. Config is one of the parameters of the reader().

As for the time we have to specify the format of the time column in our custom data. The format is year, month and day separated by a dash and hour, minute and second separated by a colon.

*Note that we add a time delta of one minute to this time since this will be the end time
of this data point. Since tweets should be available pretty much right away, we could also use a time delta of one second, but since we are using minutely data anyways, this likely won't make a big difference.*

*Be aware that when using price data or other data covering a certain period to always correctly
adjust the time to the end time of the data, otherwise the results can be very unrealistic.*

After setting the time we convert the tweet which is saved into the second place of the
data list to lowercase and save it to the content variable.

Now we want to perform some simple sentiment analysis on the content of each tweet. However, we are only interested in those tweets that reference Tesla.

Since the tweet was converted to lowercase, we can check whether it contains the ticker
symbol of Tesla or the written out version. If it does, we use our sentiment intensity analyzer to calculate a polarity score for the tweet. This polarity score will return a dictionary with scores for negative, positive and compounded sentiment(NKPKN).
Know that the compounded sentiment is not just an average of the others. An example of a tweet with a negative sentiment would be this one:
 
“Not all good news, Virginia DMV commissioner just denies Tesla a dealer license.”

This is interpreted to be a neutral to negative statement.

But, as you likely inferred, also know that often the sentiment analysis isn't as accurate as in these two cases. 

What we want to do is use the sentiment to either establish a long or short Tesla position. We will only use the compounded score for this though. That's why we save it as the value of the tweet data object. In the case that Tesla was not referenced in the tweet we just assign it a score of zero, which would be equivalent to a completely neutral tweet.

Next up, I also want the content of the tweet to be accessible in the data bar.

To accomplish this, we can simply create a property for the tweet object with whatever
name you want to and save the content to it.
We will use “tweet” as the name for this property.

Add some form of exception handling to reader() by putting a try statement around this code, and if a value error is thrown we simply
return none. 

Return the tweet object and then we are done with the reader().

Now we are almost done, we just have to quickly implement the onData() and the exit logic.

The first thing we do in onData is check whether the data slice object contains any data for
the musk tweets. We check this with the symbol object of that data.

If that's the case, we can save the score of the current tweet with data.value.
To access the content of the tweet, we can use .tweet - the name that we gave
this property in the reader() of the muskTweet class.

For this algorithm, we will establish a long position if the compounded sentiment scoreis above 0.5 and we establish a short position if the score
is below negative 0.5. We can check this with the simple if-elif condition and use set holdings to go long or short. Now all that's left to do is add a method that liquidates any open position before the market closes since we don't want to hold Tesla overnight based on one tweet. For this we can schedule an exitPositions() in the initialize(). We schedule it to be called 15 minutes before the close on every day that Tesla trades.
In the exitPositions() we then simply have to call self.liquidate to close all open positions.


## Dependencies

- Python 3
- Pandas
- Numpy
- NLTK


## To run

This is designed to run on the QuantConnect platform, built on top of their LEAN engine and using their method/class infrastructure. To run, create a QuantConnect account, navigate to the Algorithm Lab, select 'Create New Algorithm', and add the files in this repository. Make sure to backtest it!
