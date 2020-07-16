# Live Twitter Data Analysis and Visualization using Python and Plotly Dash

## Introduction
Twitter is a platform that embraces tons of information flow in every single second, which should be fully utilized if one wants to explore the real-time interaction between communities and real-life events. I think it would be cool to have an application that is capable of collecting, storing, analyzing, and finally, visualizing Twitter data in real-time so that we could know what is precisely happening by the time people run the application.

Having this exciting idea in mind, I implemented this app and deployed it to the cloud. The app itself can be accessed through this link (Figure 1). The app collects all the tweets related to the gaming community posted and visualizes some statistics. The first panel on the left-hand side is a line plot of the word-count trend, showing the changing pattern of the top-5 most frequently mentioned words. The top panel on the right-hand side is a bar chart showing the top-10 words for better comparison. The figure below is a time-shifting scatter plot of the averaged real-time sentiment score for all the tweets grouped by the top-5 mentioned words.

With the predefined tracking keywords, the user can get the corresponding real-time reaction from the Twitter community (for the gaming community in this example) about a specific topic. It can also be integrated into a recommendation system that requires short reacting time, or a Twitter surveillance system during an event, etc. In this post, I will talk about the framework and tools that I used to build this app.

<p align="center"><img width=95.5% src="https://github.com/jishubasak/Twitter-Data-Analysis-and-Streaming/blob/master/catalog/gif_1.gif" width=40%></p>

## Application Framework
I introduced how to stream tweets with Python using Tweepy package in one of the old posts. But previously, the process of data collection, data migration, storage, and processing are handled in individual steps, or even totally different timelines. For example, if the goal is to collect all the tweets that tweeted in the next two days, we will have to wait for 48 hours for the streaming to finish, then we can start to process the data. No matter how fast we finish the downstream work, there is still at least a two-day gap between the time when we collected the first record and the time we gain insight from the data.

For this live streaming app, however, the traditional workflow is divided into two independent pipelines working together (Figure 2). In detail, data processing starts right after the first line of record is received, followed by data analyzing and results-visualization, etc. While the visualization server (Plotly Dash) is handling data-processing, the streaming server (Tweepy), on the other hand, will bring in the next line of newly generated data. Each pipeline keeps looping at its own pace. Figure 2 shows the underlying framework of all the processes described: 

<p align="center"><img width=95.5% src="https://github.com/jishubasak/Twitter-Data-Analysis-and-Streaming/blob/master/catalog/app_framework-1.png" width=40%></p>

As shown in Figure 2, to make the application react promptly, we break the traditional data science pipeline into two modules that are taken care of by their corresponding local servers. Specifically, Tweepy is responsible for streaming data, i.e., control the flow of tweets, and we add extra functionalities to the trigger behavior, such as pushing data into a database or deleting old records from the database (optional). Note that the extra functionalities are triggered whenever there is a tweet falls under the predefined tracking conditions thus heard by the server. Therefore, for the Tweepy server,  the intervals between the last operation and the next are random, as shown in Figure 2.

Meanwhile, data processing and visualization are carried out by another platform, Dash by Plotly. Dash controls the rendering and refreshing of the visualization on the browser; hence, a constant time interval for this trigger is preferred. Depends on the data throughput, the time window can be ranging from 0.50.5 seconds to 55 seconds (22 seconds for this app). In a big picture, two running servers are executing their loops independently, and data is flowing in between them in the predefined time interval to update the application’s graphical interface. 

## Data Ingestion

For the data collecting module, we use Tweepy to set up a local server to hear tweets from the Internet. First, create a file with name streaming.py; which sets up the authentication and trigger conditions, the code for streaming.py; is shown below:

```python
# import packages
from tweepy import OAuthHandler
from tweepy import API
from tweepy import Stream
from sqlalchemy import create_engine
from sqlalchemy_utils import database_exists, create_database
from urllib3.exceptions import ProtocolError
from slistener import SListener
from key_secret import consumer_key, consumer_secret
from key_secret import access_token, access_token_secret
```

In the script above, we first import all the necessary packages, including Tweepy and Sqlalchemy. We also import ProtocalError from urllib3.exceptions so the app can automatically resume the streaming even if a connection error occurred later on. The following slistener and key_secrete object are just two other Python scripts in the same folder. Specifically, the key_secrete.py contains all the credential information such as keys and tokens. And slistener.py contains a child SListener class object inherited from Tweepy’s standard StreamListener class (more on this in a bit).

```python
# consumer key authentication
auth = OAuthHandler(consumer_key, consumer_secret)
# access key authentication
auth.set_access_token(access_token, access_token_secret)
# set up the API with the authentication handler
api = API(auth)
# instantiate the SListener object
listen = SListener(api)
# instantiate the stream object
stream = Stream(auth, listen)
# set up words to hear
keywords_to_hear = ['#Fortnite',
                    '#LeagueOfLegends',
                    '#ApexLegends']
 ```
Following importing packages, the next five lines of code are the standard routine to set up a streaming API using the authentication and predefined SListener object. keywords_to_hear is a list of keywords we want to catch, in other words, only if the tweets have one or more of the keywords in the list, would our streaming server capture it.

```python
# create a engine to the database
engine = create_engine("sqlite:///tweets.sqlite")
# if the database does not exist
if not database_exists(engine.url):
    # create a new database
    create_database(engine.url)
# begin collecting data
while True:
    # maintian connection unless interrupted
    try:
        stream.filter(track=keywords_to_hear)
    # reconnect automantically if error arise
    # due to unstable network connection
    except (ProtocolError, AttributeError):
        continue
```
Next, we use the create_engine function from Sqlalchemy to create a database to store our data if the database does not exist yet. We create the database here since we want a database ready to receive data before the stream is launched. Lastly, stream.filter(track=keywords_to_hear) launches the server and start to stream tweets that contain the keywords we specified in the list. We put this line of code into a while-loop since we do not want the streaming to be terminated if the network connection was randomly interrupted. The streaming will only be terminated if we stop the server manually.  

Once the streaming is launched, tweets will be flowing into our localhost machine. The way how the raw data should be handled is further specified by the SListener object. And the code below is a sample script for how to define the SListener object. Create a new file called slistener.py and put the following blocks of code into it. Begin with importing:

```python
# import packages
from tweepy.streaming import StreamListener
import json
import pandas as pd
from sqlalchemy import create_engine
```
First of all, we import the base StreamListener class object from tweepy so we can add some extra functionalities to it later on. Next, to process the raw Twitter statuses object, JSON package is needed to parse the statuses object into a JSON object to select the useful fields. Pandas is also necessary since it provides a convenient bridge connecting our Python working environment to the database.

```python
# inherit from StreamListener class
class SListener(StreamListener):
    # initialize the API and a counter for the number of tweets collected
    def __init__(self, api = None, fprefix = 'streamer'):
        self.api = api or API()
        # instantiate a counter
        self.cnt = 0
        # create a engine to the database
        self.engine = create_engine('sqlite:///tweets.sqlite')
    # for each tweet streamed
    def on_status(self, status): 
        # increment the counter
        self.cnt += 1
        # parse the status object into JSON
        status_json = json.dumps(status._json)
        # convert the JSON string into dictionary
        status_data = json.loads(status_json)

The rest of the script is for customizing our StreamListener class. First, in addition to the parent class provided by tweepy, we also initialize a database engine in the __init__ method, and the cnt variable is simply an indicator for the number of tweets streamed so far. The on_status method is the most essential part of the whole class. Whenever a new tweet is heard, this on_status method would be triggered. The first 3 lines of code parse the incoming raw data into a python-accessible data structure, JSON dictionary. Basically, the raw object is converted into a dictionary containing all the information of that specific tweet, such as the timestamp when it was created, the user, its text, etc. 

```python
# initialize a list of potential full-text
full_text_list = [status_data['text']]
# add full-text field from all sources into the list
if 'extended_tweet' in status_data:
    full_text_list.append(status_data['extended_tweet']['full_text'])
if 'retweeted_status' in status_data and 'extended_tweet' in status_data['retweeted_status']:
    full_text_list.append(status_data['retweeted_status']['extended_tweet']['full_text'])
if 'quoted_status' in status_data and 'extended_tweet' in status_data['quoted_status']:
    full_text_list.append(status_data['quoted_status']['extended_tweet']['full_text'])
# only retain the longest candidate
full_text = max(full_text_list, key=len)
```
The rest of the script is for customizing our StreamListener class. First, in addition to the parent class provided by tweepy, we also initialize a database engine in the __init__ method, and the cnt variable is simply an indicator for the number of tweets streamed so far. The on_status method is the most essential part of the whole class. Whenever a new tweet is heard, this on_status method would be triggered. The first 3 lines of code parse the incoming raw data into a python-accessible data structure, JSON dictionary. Basically, the raw object is converted into a dictionary containing all the information of that specific tweet, such as the timestamp when it was created, the user, its text, etc. 

```python
# extract time and user info
tweet = {
    'created_at': status_data['created_at'],
    'text':  full_text,
}
# uncomment the following to display tweets in the console
print("Writing tweet # {} to the database".format(self.cnt))
print("Tweet Created at: {}".format(tweet['created_at']))
print("Tweets Content:{}".format(tweet['text']))
#print("User Profile: {}".format(tweet['user']))
print()
# convert into dataframe
df = pd.DataFrame(tweet, index=[0])
# convert string of time into date time obejct
df['created_at'] = pd.to_datetime(df.created_at)
# push tweet into database
df.to_sql('tweet', con=self.engine, if_exists='append')
```
The rest of the two fields (the timestamp and the user’s profile) can be accessed more straightforwardly as we generate the dictionary tweet. We use print functions to help verify if the streaming is working fine. Then, we convert the dictionary into a pandas DataFrame. Since there is just one observation (row) to be converted, we tell pandas that we only have one index for the row by passing in a list with 0 in it as a parameter. Then we turn the timestamp from a string object into a datetime object for time-series-analysis, and then push the data into a database using the engine we initialized in the __init__ method.

```python
with self.engine.connect() as con:
    con.execute("""
                DELETE FROM tweet
                WHERE created_at IN(
                    SELECT created_at
                        FROM(
                            SELECT created_at, strftime('%s','now') - strftime('%s',created_at) AS time_passed
                                    FROM tweet
                                    WHERE time_passed >= 60))""")
```
For our case, since the application is hosted on the cloud 24/7, I certainly do not want all the tweets to stay in the remote machine forever since there is a storage limit on the host machine. So I choose to connect to the database again and remove the records that are older than 60 seconds from my database using SQL query. If you are running this application on your local machine or you do not care about the storage limit, you can remove the query. 

 ## Data Visualization
 
 The framework we use to visualize the data is Dash by Plotly, which is a Python framework written on top of Flask, Plotly.js, and React.js, for building analytical web applications. It allows us to implement complicated web functionalities with much fewer code compares to Flask or Django. The two most essential libraries for creating an interactive visualization app with Dash is the dash_core_components and dash_html_components, they come in automatically once Dash is installed in the current working environment. dash_core_components provides components that can be added to the interactive webpage, such as a graph, a slider, or a dropdown menu. On the other hand, the general layout of the webpage is specified using dash_html_components, which contains functions for generating all HTML tags so that we can create the web content in pure Python environment with a similar code structure to an HTML file.

To kick things off, create a new file app.py. The code inside app.py can usually be divided into two parts, the first part specifies the layout, and the second part specifies the interactive behavior (i.e., callbacks). Start with importing:

```python
# import basic packages
import os, datetime, re, nltk
import numpy as np
# import data ingestion packages
from collections import deque, Counter
from data_gathering.api import get_tweet_data
# import Dash packages
import dash
import dash_core_components as dcc
import dash_html_components as html
from dash.dependencies import Output, Input, State
# import Plotly Package
import plotly.graph_objs as go
# import NLTK packages
from nltk import word_tokenize
from nltk.corpus import stopwords
from nltk.sentiment.vader import SentimentIntensityAnalyzer
```
For this app specifically, we need to import some other packages to handle the extra functionalities, such as datetime for accessing current time, nltk for word counting and sentiment analysis, etc. We also need to import plotly.graph_objs, which contains all the graph objects, for instance, scatter plot, bar chart, and pie chart.

```python
# download nltk dependencies
nltk.download('punkt')
nltk.download('stopwords')
nltk.download('vader_lexicon')
# stop words for the word-counts
stops = stopwords.words('english')
stops.append('https')
# global refresh interval for the application, ms
GRAPH_INTERVAL = os.environ.get("GRAPH_INTERVAL", 2000)
# initialize a sentiment analyzer
sid = SentimentIntensityAnalyzer()
# keywords for the tweets
keywords_to_hear = ['#Fortnite',
                    '#LeagueOfLegends',
                    '#ApexLegends']
# X_universal is the x-axis with time stamps
X_universal = deque(maxlen=30)
# initialize dictionaries to store Y axis values for plots
scatter_dict = {}
sentiment_dict = {}
```
Next up are some pre-configurations. First, we download some NLTK dependencies, which are essential components for tokenization. For example, punkt is a package that contains pre-trained tokenizers that detects sentence boundaries. stopwords is a package with lists of stopwords in different languages. Here we specify that we only need the stopwords in English, and we also append https to the stopwords since it is a high-frequency word that does not contain much information. Then we get the application fresh-rate GRAPH_INTERVAL from the environment variables and set it to 2000 ms (2 seconds) if it is not found. Next, the x- and y-axis of the scatter plot is initialized by allocating deque objects with a maximal length of 30 since we only want to show 30 timestamps on the plot.

```python
# initialize the app and server
app = dash.Dash(__name__, meta_tags=[{"name": "viewport", "content": "width=device-width, initial-scale=1"}])
server = app.server
# add layout to the app
app.layout = html.Div(
    [
        # header
        html.Div(...
        ),
        # graph layout
        html.Div(
            [
                # left hand side, word count line plot
                html.Div(...
                ),
                # right hand side, bar plot and scatter chart
                html.Div(...
                ),
            ],
            className="app__content",
        ),
    ],
    className="app__container",
)
```
The code above shows how to initialize the application and its server with some default meta settings, followed by the layout object. Note that here I replace the detail layout setting with “…” to give you an overview of the webpage structure. The layout is straight-forward to specify and does not require a lot of coding skills. If you are curious about the details, you can refer to this page to see the raw code for the layout of this application. 

Next, we add some helper functions and callback functions to the application:
```python
def hashtag_counter(series):...
def bag_of_words(series):...
def preprocess_nltk(row):...
# define callback function for number_of_tweets scatter plot
@app.callback(
    Output('number_of_words', 'figure'),
    [Input('query_update', 'n_intervals')])
def update_graph_scatter(n):...
# define callback function for word-counts
@app.callback(
    Output('word_counts', 'figure'),
    [Input('query_update', 'n_intervals')])
def update_graph_bar(interval, slider_value):...
# define callback function for sentiment_score
@app.callback(
    Output('sentiment_score', 'figure'),
    [Input('query_update', 'n_intervals')])
def update_graph_sentiment(interval):...
# define callback function indicating number of tweets
@app.callback(
    Output("num_tweets", "children"),
    [Input("query_update", "n_intervals")])
def show_num_tweets(n_intervals):...
# run the app
if __name__ == '__main__':
    app.run_server(debug=True, host='0.0.0.0')
```
Again, I hide the details of the functions to give you a general idea of the structure and make it less overwhelming to follow through. Helper functions can always differ for different applications, refer to the link above if you want to see the details. Basically, hashtag_counter takes a pandas series and return the number of rows fall into each keyword category and return a dictionary with all the counts. bag_of_words and preprocess_nltk are text-cleaning tools.

For the callback functions, we use Python decorators to add extra functionalities to the original app.callback function, including different input/output, and different refresh behaviors. Here I use the callback function for the line plot as an example:

```python
@app.callback(
    Output('number_of_tweets', 'figure'),
    [Input('query_update', 'n_intervals')])
def update_graph_scatter(n):
    # query tweets from the database
    df = get_tweet_data()
    # get the word count
    cnt = bag_of_words(df['text'])
    # get the current time for x-axis
    time = datetime.datetime.now().strftime('%D, %H:%M:%S')
    X_universal.append(time)
    # initialize a list of items to pop
    to_pop = []
    # loop through the dictionary to filter out outdated keywords
    for keyword, cnt_queue in scatter_dict.items():
        # if the count queue for current keyword is not empty
        if cnt_queue:
            # pop all the outdated count values
            while cnt_queue and (cnt_queue[0][1] < X_universal[0]):
                cnt_queue.popleft()
        # if the queue for the keyword is empty
        else:
            # append it to the pop list
            to_pop.append(keyword)
    # pop all the outdated keywords
    for keyword in to_pop:
        scatter_dict.pop(keyword)
    # update the top_N keywords
    top_N = cnt.most_common(num_tags_scatter)
    # loop through the new top_N to update the dictionary
    for keyword, cnt in top_N:
        # if the current keyword is newly appeared
        if keyword not in scatter_dict:
            # initialize a new deque for it and append its count and time
            scatter_dict[keyword] = deque(maxlen=30)
            scatter_dict[keyword].append([cnt, time])
        # if it is not a new one, just append the count and time
        else:
            scatter_dict[keyword].append([cnt, time])
    # update the colors for the new dictionary
    new_colors = chart_colors[:len(scatter_dict)]
    # plot the scatter plot
    data=[go.Scatter(
        x=[time for cnt, time in cnt_queue],
        y=[cnt for cnt, time in cnt_queue],
        name=keyword,
        mode='lines+markers',
        opacity=0.7,
        marker=dict(color=color, size=15)
    ) for color, (keyword, cnt_queue) in list(zip(new_colors, scatter_dict.items()))]
    # specify the layout
    layout = go.Layout(
            xaxis={
                'automargin': False,
                'range': [min(X_universal), max(X_universal)],
                'title': 'Current Time',
                'nticks': 8},
            yaxis={
                'type': 'log',
                'autorange': True,
                'title': 'Number of Tweets'},
            height=700,
            plot_bgcolor=app_color["graph_bg"],
            paper_bgcolor=app_color["graph_bg"],
            font={"color": app_color["graph_font"]},
            autosize=False,
            legend={
                'orientation': 'h',
                'xanchor': 'center',
                'yanchor': 'top',
                'x': 0.5,
                'y': 1.025},
            margin=go.layout.Margin(
                l=75, r=25, b=45, t=25, pad=4))
    return go.Figure(
        data=data,
        layout=layout)
```

## Conclusion
The last line of code in the twitter.py sets the debug mode to True so that, after the server is launched, changes will be automatically applied without restarting the server. If the application is ready to deploy, you might want to set debug mode to False. Set the host IP address to 0.0.0.00.0.0.0 to allow access from outside the localhost.

Now if you open a terminal and execute python app.py, the server should be up and running on your local workstation at 0.0.0.0:80500.0.0.0:8050 (default port for Dash is 80508050).




To run the application locally with Docker:

    1. Install Docker, Docker Compose
    2. Clone this repository
    3. $ cd twitter_streaming_app/
    4. Set up your API keys and secret in ./data_gathering/keys_secret.py
    5. $ sudo docker-compose up
    6. Go to http://0.0.0.0:80
