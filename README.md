# Live Twitter Data Analysis and Visualization using Python and Plotly Dash

## Introduction
Twitter is a platform that embraces tons of information flow in every single second, which should be fully utilized if one wants to explore the real-time interaction between communities and real-life events. I think it would be cool to have an application that is capable of collecting, storing, analyzing, and finally, visualizing Twitter data in real-time so that we could know what is precisely happening by the time people run the application.

Having this exciting idea in mind, I implemented this app and deployed it to the cloud. The app itself can be accessed through this link (Figure 1). The app collects all the tweets related to the gaming community posted and visualizes some statistics. The first panel on the left-hand side is a line plot of the word-count trend, showing the changing pattern of the top-5 most frequently mentioned words. The top panel on the right-hand side is a bar chart showing the top-10 words for better comparison. The figure below is a time-shifting scatter plot of the averaged real-time sentiment score for all the tweets grouped by the top-5 mentioned words.

With the predefined tracking keywords, the user can get the corresponding real-time reaction from the Twitter community (for the gaming community in this example) about a specific topic. It can also be integrated into a recommendation system that requires short reacting time, or a Twitter surveillance system during an event, etc. In this post, I will talk about the framework and tools that I used to build this app.

To run the application locally with Docker:

    1. Install Docker, Docker Compose
    2. Clone this repository
    3. $ cd twitter_streaming_app/
    4. Set up your API keys and secret in ./data_gathering/keys_secret.py
    5. $ sudo docker-compose up
    6. Go to http://0.0.0.0:80
