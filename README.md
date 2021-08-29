# Logging and notifying about rare events with Slack Webhooks and Log4j

Have you ever logged something like this in your code?

```java
  logger.warn("This shouldn't happen: state={}", state)
```

Indeed `WARN` logging level is often used to indicate something odd and unexpected about your application logic, but
may be not too critical that you can't recover from (in that case `ERROR` level would be appropriate).
Almost by definition this is about rare events that are hard to reproduce to investigate.
That's why quite often developers have nothing better than to log as much details as possible about the oddity, so
when it does happen they would have at least some data in the logs to help.

But how do you know that some rare event happened in your application?
Regularly checking the logs is not an option for something that happens once in a blue moon.
Especially when logs are huge, numerous and are stored in restricted locations.
[Advanced logs management solutions](https://www.elastic.co/what-is/elk-stack) can definetely help, but
such tool are often not avaialble to developer, and when they are, their learning curve is quite steep.

Is there existing technology at the intersection of logging and notificaton concerns that could provide a simpler
solution to our problem?

Recently I learned about great Slack feature called [Workflows](https://api.slack.com/workflows):

```
Workflows are automated multi-step tasks or processes that can run right in Slack,
or connect with other tools and services. Workflows in Slack can be as simple or
as complex as you’d like, and typically don’t require writing any code.

If you’re a developer or have some experience with code,
you may want to create a workflow triggered by an event in an external
service (like an internal tool your company uses) with a webhook.
```

[Webhook](https://slack.com/intl/en-ca/help/articles/360041352714-Create-more-advanced-workflows-using-webhooks) is an
auto-generated, unique URL that Slack provides for your Workflow, that you can trigger by POST'ing a JSON payload
containing Worflow's variables:

```
Slack will generate a unique request URL for your workflow once you publish it,
and you can configure your webhook to pass information to Slack via the HTTP request body.
Any data your webhook sends to Slack can be referenced in subsequent workflow steps by creating variables.
```

These are steps to create a workflow with a webhook trigger that sends a message to specific channel:

1. select existing channel or create new
2. go to channel's details (righ click on channel name, select "Open channel details")
3. in channel details dialog open "Integrations" tab
4. in "Workflow" section click on "Add a workflow"
5. in the "Workflow Builder" dialog click on "Create"
6. Give your workflow a name (e.g. "PROD Monitor") and click on "Next"
7. Select "Webhook" in the "Choose a way to start your workflow" dialog
8. In the "Webhook" dialog click on "Add variable"
9. Enter key named "text" of the "Text" Data type and click on "done"
10. In the "Webhook" dialog click on "Next"
11. In the newly opened dialog click on "Add Step"
12. In the "Add a workflow step" dialog select "Send a message" step by clicking on "Add"
13. In the "Send a message" dialog select channel from step #1 in the "Send this message to" drop-down list
14. Click on "Insert a viariable" link right below "Message Text" editor and select "text" variable from step #9
15. In the editor click on a "Code block" icon to embed "text" variable inside it
16. Click on "Save"
17. Now that you're back to your Worlflow Builder dialog - click on "Publish"

Finally, you will be presented with "Your workflow is ready to use" dialog with unique Webhook URL. Copy it.

You can now post messages to your workflow's channel with this simple `curl` command: 

```
% curl -X POST -H 'Content-type: application/json' --data '{"text":"Hello Workflow"}' https://hooks.slack.com/workflows/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
```

Next, we will see how *Log4j* can be configured to make certain loggers and/or certain logging levels (e.g. `WARN`) to
"HTTP POST" log messages to a configured Webhook URL.

For that we will use *Log4j*'s standard `Http` appender together with customized `PatternLayout`:
```
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{ISO8601} [%t] %-5level %logger - %msg%n"/>
        </Console>
        <Http name="SlackProdMonitorWebhook"
              url="https://hooks.slack.com/workflows/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"
              connectTimeoutMillis="2000"
              readTimeoutMillis="1000">
            <PatternLayout pattern="{&quot;text&quot;:&quot;[%t] %logger - %enc{ %m }{JSON}&quot;}"/>
        </Http>
    </Appenders>
    <Loggers>
        <Logger name="com.starsgroup.ffs">
            <AppenderRef ref="SlackProdMonitorWebhook" level="warn"/>
        </Logger>

        <Root level="info">
            <AppenderRef ref="Console"/>
        </Root>
    </Loggers>
</Configuration>
```

That's all!
