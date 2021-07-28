---
layout: post
title:  "Setting Up Graylog To Send Slack Messages"
date:   2020-12-07 17:41:50 +0530
---
# Setting up Graylog to send Slack messages

In the [previous post](https://mlndshh.github.io/2020/12/02/fluent-graylog.html), we successfully set up fluentd to send logs to Graylog. This time we will set up Graylog to send messages on Slack on any channel of our choice. Let's get started!

## Slack Setup
Make sure you have a Slack workspace that you can use for this.

We will get started by creating a Slack app, so that we can create an [Incoming Webhook](https://api.slack.com/messaging/webhooks). An incoming webhook is used to send messages to Slack channels from your apps. You get an URL that you send a JSON payload too.

### Create a Slack app
[Click here](https://api.slack.com/apps?new_app=1), and create a new app. Enter a name for it and then choose the workspace you'll use for this.

![Create your slack app](/images/graylog-slack/createapp.png)

### Enable Incoming Webhooks
After creating the application, you will be redirected to its settings page where you can enable Incoming Webhooks

![Enable Incoming Webhooks](/images/graylog-slack/enablewebhooks.png)

### Create an Incoming Webhook 
Once you have enabled Incoming Webhooks, on the same page you can create a new webhook. Add a new webhook and it will ask you to choose a channel to post to as an app. Choose one and click on `Allow`. 

![Create an Incoming Webhook](/images/graylog-slack/webhook1.png)

You will now be redirected back to the webhooks page with the newly created hook now present. You can test it out with the sample curl request they have given. We'll be using the URL it creates in Graylog.

![Create an Incoming Webhook](/images/graylog-slack/webhook2.png)

We can now start setting up Graylog from here.

## Graylog - Installing the plugin and setting up notifications

### Installing the Slack plugin
There are plugins out there that allow us to set up Graylog notifications that send messages to Slack using the webhook url.

**This is important here, if you are on Graylog version <3.1, [use this plugin](https://github.com/graylog-labs/graylog-plugin-slack)**

**If you are on Graylog version >=3.1, [use this plugin](https://github.com/sportalliance/graylog-plugin-slack-notification)**

Assuming you've made your chocie from above, let's continue. To install the plugin, we will create a volume for the .jar file of the plugin. Add this in your `docker-compose.yml` for Graylog.

```
    volumes:
      - ./graylog-plugin-slack-notification-1.0.5.jar:/usr/share/graylog/plugins/graylog-plugin-slack-notification-1.0.5.jar
```

The above .jar file name will change based on which plugin you are using. I am using [the one meant for versions>3.1](https://github.com/sportalliance/graylog-plugin-slack-notification)

Entire file for reference

```
version: "3"
services:
  mongo:
    image: mongo:3
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch-oss:6.8.10
    environment: 
      - http.host=0.0.0.0
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
  graylog:
    image: graylog/graylog:3.3
    environment: 
      - GRAYLOG_HTTP_EXTERNAL_URI=http://127.0.0.1:9000/
    links:
      - elasticsearch
      - mongo
    ports:
      - 9000:9000
      - 12201:12201
      - 12201:12201/udp
      - 1514:1514
    volumes:
      - ./graylog-plugin-slack-notification-1.0.5.jar:/usr/share/graylog/plugins/graylog-plugin-slack-notification-1.0.5.jar
```

That's it! Now, start (or restart) your Graylog server, and let's set up a notification and an event. The notification is basically used as an endpoint for the event (Everytime said event is triggered , send said notification).

### Graylog Notification
In the Menu, click on `Alerts` and choose `Notifications`. Click on `Create Notification`. Give it a title, description, and choose `Slack Notification` as the Notification Type. 

![New Notification](/images/graylog-slack/notif1.png)

Enter your created Webhook URL here and change the message formats if needed.

Enter the other optional information as needed, and cilck on `Execute Test Notification` to send a test message to your Slack channel. If everything has been done right so far, You will receive the message.

### Graylog Event
Let's set up an event now that will look for a condition at constant intervals and sends notifications (like the one we just created) if it finds logs matching to the condition.

1. In the Menu, click on `Alerts` and choose `Event Definitions`. Click on `Create Event Definition`. 

2. Give a Title, Description and Priority and move onto the Condition.

3. Here, select `Filter & Aggregation` as the `Condition Type`. In the search query for my example, I'm using `message:"this is a test"`. Select `Streams` if needed, and change `Search within the last` and `Execute search every` as per your liking. `Search within the last` specifies how to only search within the mentioned duration and `Execute search every` specifies how often to search for matching logs. For my case here, I've set both to `30 seconds`
 
![New Notification](/images/graylog-slack/event.png)

4. In `Fields`, you can add custom fields that you might need in the event. In notifications, add the notification you just created.

5. Check everything once in the `Summary`, and then click on `Done`. Let's test this out now!


## Tests
If you have followed my posts for fluentd and Graylog, we should have similar setups. Let's send messages from the python file we were using which sends `message:"this is a test"` (which is also what we are looking for).

```
$ python test_fluent.py
$ python test_fluent.py
$ python test_fluent.py
``` 

**test_fluent.py:**
```
from fluent import sender
logger = sender.FluentSender('app', host='localhost', port=24224)
logger.emit('follow', {'from': 'userA', 'to': 'userB', 'message': 'this is a test'})%   
```

**Fluentd setup:**

```
clientfluent_1  | 2020-12-07 12:02:45.000000000 +0000 app.follow: {"from":"userA","to":"userB","message":"this is a test"}
clientfluent_1  | 2020-12-07 12:02:46.000000000 +0000 app.follow: {"from":"userA","to":"userB","message":"this is a test"}
clientfluent_1  | 2020-12-07 12:02:47.000000000 +0000 app.follow: {"from":"userA","to":"userB","message":"this is a test"}
serverfluent_1  | 2020-12-07 12:02:45.000000000 +0000 app.follow: {"from":"userA","to":"userB","message":"this is a test"}
serverfluent_1  | 2020-12-07 12:02:46.000000000 +0000 app.follow: {"from":"userA","to":"userB","message":"this is a test"}
serverfluent_1  | 2020-12-07 12:02:47.000000000 +0000 app.follow: {"from":"userA","to":"userB","message":"this is a test"}
```

**Graylog Receiving the logs:**

![Graylog](/images/graylog-slack/graylognotif.png)

**Slack Receiving the messages soon after:**

![Slack](/images/graylog-slack/slack.png)

And there we have it! We've successfully setup Graylog to send Slack notifications.