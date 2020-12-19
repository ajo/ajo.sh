---
layout: post
title:  "Creating a AWS Lex Chatbot With MeiliSearch & Lambda"
date:   2020-12-19 02:22:49 -0500
categories: AWS Lex Lambda
---
# Creating an AWS powered chatbot with search.

Using [Amazon Lex](https://aws.amazon.com/lex/) and [AWS Lambda](https://aws.amazon.com/lambda/) together with the REST API provided by [MeiliSearch](https://www.meilisearch.com/) allows you to quickly and cheaply add search functionality to a chat bot. This could be used to search helpdesk articles, movies, recipes or anything else you could create text to describe. This guide assumes some familiarity with Linux (Running on DigitalOcean), AWS, Python, and Rest APIs. 

*Please note that this is an example of how you can use Amazon Lambda Functions, Amazon Lex, and MeiliSearch together to search content. Its focus is on quickness and simplicity. If you want to do something like this in production, please take care to [set proper security](https://docs.meilisearch.com/running-production/) in place for AWS resources and MeiliSearch.*

## Setting Up MeiliSearch

1. Create a DigitalOcean droplet using the MeiliSearch template from the marketplace. Once created, connect to your droplet via SSH. The MeiliSearch setup wizard should automatically start.

2. Proceed through the setup wizard. Select yes to setting up a master key, and no to the next two prompts.

***Take note of the MEILI_MASTER_KEY you receive**. In my case it was YWEwMDYyYjcxZjk5YmNhZGE2MjQyZWUw.*

3. Add your data – for this example I will add the [movies.json](https://github.com/meilisearch/MeiliSearch/blob/master/datasets/movies/movies.json) data from the  [MeiliSearch Getting stated documentation](https://docs.meilisearch.com/guides/introduction/quick_start_guide.html#create-your-index)

From the server:

{% highlight bash %}
wget https://raw.githubusercontent.com/meilisearch/MeiliSearch/master/datasets/movies/movies.json
curl -H "X-Meili-API-Key: YWEwMDYyYjcxZjk5YmNhZGE2MjQyZWUw" -X POST "http://127.0.0.1:7700/indexes/movies/documents" --data @movies.json
{% endhighlight %}

**Replace YWEwMDYyYjcxZjk5YmNhZGE2MjQyZWUw with your MEILI_MASTER_KEY that you received during the setup wizard.**

4. Test your installation & import with the following command. 

This queries the API as if you had searched “computer” from the web interface of Meilisearch (which can bee seen at your droplets IP address)

{% highlight bash %}
curl -H "X-Meili-API-Key: YWEwMDYyYjcxZjk5YmNhZGE2MjQyZWUw"  -X POST 'http://localhost:7700/indexes/movies/search' --data '{ "q": "computer" }'
{% endhighlight %}

If you only wanted the top 3 results you could add the argument limit=2 to the URL

{% highlight bash %}
curl -H "X-Meili-API-Key: YWEwMDYyYjcxZjk5YmNhZGE2MjQyZWUw"  -X POST 'http://localhost:7700/indexes/movies/search' --data '{ "q": "computer", "limit": 3 }'
{% endhighlight %}

If you only wanted the names & descriptions for the top 3 results you could add the argument limit

{% highlight bash %}
curl -H "X-Meili-API-Key: YWEwMDYyYjcxZjk5YmNhZGE2MjQyZWUw"  -X POST 'http://localhost:7700/indexes/movies/search' --data '{ "q": "computer", "limit": 3, "attributesToRetrieve": ["title","overview"] }'
{% endhighlight %}

5. Before we can use MeiliSearchs API from outside of localhost, we need to let it through the firewall. Assuming you are using the DigitalOcean template, you can do so with the following:

{% highlight bash %}
sudo ufw allow 7700
{% endhighlight %}

You can view active firewall rules with

{% highlight bash %}
sudo ufw status verbose
{% endhighlight %}

6. By default the MeiliSearch API only listens on localhost. We need it to listen to public requests for our lambda function to work.  

We can open this up by adding --http-addr=0.0.0.0:7700 to the ExecStart configuration for MeiliSearch in system.

{% highlight bash %}
nano /etc/systemd/system/meilisearch.service
{% endhighlight %}

Enable the MeiliSearch service on startup and restart the service.

{% highlight bash %}
systemctl enable meilisearch
service meilisearch restart
{% endhighlight %}

You can test that your Meilisearch API is responding to public request by running

{% highlight bash %}
curl -H "X-Meili-API-Key: YWEwMDYyYjcxZjk5YmNhZGE2MjQyZWUw"  -X POST 'http://your.droplet.ip.address:7700/indexes/movies/search' --data '{ "q": "computer", "limit": 3, "attributesToRetrieve": ["title","overview"] }'
{% endhighlight %}

## Setting up the Lambda Function

1. Create a new Project in your favorite Python IDE. 
2. Create file named lambda_function.py
3. Run `pip install requests -t .` from within the project directory. This will allow us to import and use the requests module. 

4. Paste the following into lambda_function.py - **make sure to add your IP address & key**.  This code is based off of the lambda lex-order-flowers-python example

{% highlight python %}
import requests
import logging
logger = logging.getLogger()
logger.setLevel(logging.DEBUG)


def get_slots(intent_request):
    return intent_request['currentIntent']['slots']


def lambda_handler(event, context):
    logger.debug('event.bot.name={}'.format(event['bot']['name']))
    return dispatch(event)


def close(session_attributes, fulfillment_state, message):
    response = {
        'sessionAttributes': session_attributes,
        'dialogAction': {
            'type': 'Close',
            'fulfillmentState': fulfillment_state,
            'message': message
        }
    }

    return response


def search_movies(intent_request):
    query_phrase = get_slots(intent_request)["QueryPhrase"]
    url = "http://159.65.232.219:7700/indexes/movies/search"
    headers = {"X-Meili-API-Key": "YWEwMDYyYjcxZjk5YmNhZGE2MjQyZWUw"}
    title = ""
    overview = ""

    response = requests.post(url, headers=headers,
                             json={"q": query_phrase, "limit": 1,
                                   "attributesToRetrieve": ["title", "overview"]})

    if response.json() and 'hits' in response.json():
        for hit in response.json()['hits']:
            title = hit.get('title')
            overview = hit.get('overview')

    if title and overview:
        return close(intent_request['sessionAttributes'],
                     'Fulfilled',
                     {'contentType': 'PlainText',
                      'content': 'I think you might enjoy ' + title + " - " + overview})
    else:
        return close(intent_request['sessionAttributes'],
                     'Fulfilled',
                     {'contentType': 'PlainText',
                      'content': 'Sorry, we don\'t have anything for ' + query_phrase})


def dispatch(intent_request):
    """
    Called when the user specifies an intent for this bot.
    """

    logger.debug(
        'dispatch userId={}, intentName={}'.format(intent_request['userId'], intent_request['currentIntent']['name']))

    intent_name = intent_request['currentIntent']['name']

    if intent_name == 'SearchMovies':
        return search_movies(intent_request)

    raise Exception('no such intent - ' + intent_name)'
{% endhighlight %}

5. You should have a project that contains the requests module, it's dependencies, and your python script. Zip the contents of this project (not the project itself, just the contents)

6. Create a new Lambda function, select Author from scratch, and Python 3.x (3.8 as of writing) as the runtime. I named my function getSearchResults. 

7. From the Function code section in configuration, select Actions -> Upload a .zip file. Select, and upload the previously created zip file. 

8. From the configuration area, select Test. Give it a name, enter the following as the JSON, and click create. 

{% highlight json %}
{
  "messageVersion": "1.0",
  "invocationSource": "DialogCodeHook",
  "userId": "MovieChatUser",
  "sessionAttributes": {},
  "bot": {
    "name": "CheckMovies",
    "alias": "$LATEST",
    "version": "$LATEST"
  },
  "outputDialogMode": "Text",
  "currentIntent": {
    "name": "SearchMovies",
    "slots": {
      "QueryPhrase": "missiles"
    },
    "confirmationStatus": "None"
  }
}
{% endhighlight %}

## Building the Lex Chatbot

 1. Create a new custom bot.
 2. Create a new intent.
 3. Add sample Utterances some examples are: 

- i need something to watch	
- what movie should i watch
 - im looking for a movie

4. Add a Slot named queryPhrase with the prompt What do you want to see in your movie?
5. Select your search results lambda function as the fulfillment. 
6. Add How about {movie_title}? it is about {movie_description}. as a re