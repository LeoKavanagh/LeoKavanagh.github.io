# Telegram Bot on AWS Lambda

An even simpler implementation of the simple implementation of a serverless chatbot in
[this blog post](https://www.developintelligence.com/blog/2017/08/building-serverless-chatbot-aws-zappa-telegram-api-ai/).

I use a bot like this to send myself little notifications in an easy-to-access way. Eg
```
./really-long-job.sh && curl $telegram_bot --headers "msg: The really long job finished. Get back to your desk."
```
I also have a cron job that scrapes [met.ie](https://met.ie) for a summary of the day's
weather forecast where I live (generally only one or two sentences) and then sends it to me on Telegram every morning.

There are other, better, ways of doing this sort of stuff, but using this simple little bot amuses me.
I get quite a lot of use out of it.

To get going with this, you'll need to have accounts with both the Telegram messaging app, and with AWS.

## 1. Register a bot

Telegram has good documentation for its bot API at https://core.telegram.org/bots.
You will use one of their own bots to register yours. The process is very straightforward once you get started

 1. Visit [BotFather](https://telegram.me/botfather) on the Telegram app or web interface.
 2. Enter /newbot
 3. Give it a name
 4. Give it a username

You'll receive a HTTP API token that you'll need later.

### 1a. Set environmental variables

This little program needs your own Telegram Chat ID and the Bot Token that you got from Step 1.
I've set these as environmental variables `TELEGRAM_CHAT_ID` and `THINGDONE_BOT_TOKEN`.
These are referenced in `server.py` and `bot.py` respectively.
There might be better or more secure ways of doing this,
but setting environmental variables is enough to get off the ground.

## 2. Deploy with Zappa

[Zappa](https://github.com/Miserlou/Zappa) will take care of getting all our code to into AWS:
```
zappa init
```

You should now have a file called `zappa_settings.json`. I needed to explictly set the field `aws_region`
before it would deploy for me, even though it looked like an optional parameter at first glance
(a good couple of years before writing this post, in fairness).

I chose `eu-west-1` because that's nearest to me, but it shouldn't matter much.

Now deploy the code to AWS Lambda in one line
```
zappa deploy
```
If all goes to plan, you should see a message telling you that the deploy was successful, followed by the URL to invoke it:
```
Your Zappa deployment is live!: https://...
```
You now need to set a webhook with Telegram using this url.

## 3. Set the Webhook

Once the code has been deployed to AWS Lambda (or anywhere else), you need to set a webhook with
the Telegram bot API.

For this, you'll need the bot token that BotFather gave you, and the URL where the lambda is deployed.

Then run
```
python set_webhook.py
```
You won't need to rerun this script unless you decide to move the bot off AWS Lambda and onto some other service.

# Updating the Bot

It's fairly easy to update the business logic of the bot once it's up and running. If you want to add a new endpoint to
the Flask server, say, you can test this on your localhost, then just run
```
zappa update
```
to deploy these changes to AWS Lambda.

## Test locally by running the Flask server on localhost

`cd` to the root directory of the project and start the Flask server on your localhost.
```sh
export FLASK_APP=server.py && flask run
```

You can test that it's up and running in your browser or by using `curl`:
```
curl 127.0.0.1:5000/
curl 127.0.0.1:5000/msg --header "msg: Hello from curl"
```

or by using `requests` in Python:
```python
import requests

x = requests.post('http://127.0.0.1:5000/msg', headers={'msg': 'Hello from requests'})
print(x.content)
```

In production (quote unquote), this Flask server will run on AWS Lambda instead of on your localhost.

