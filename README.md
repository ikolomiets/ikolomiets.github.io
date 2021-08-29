# Logging and notifying about rare events with Slack Webhooks and Log4j

### Table of Contents

* [Introduction](#introduction)
* [Creating and configuring Slack Workflow with Webhook trigger](#creating-and-configuring-slack-workflow-with-webhook-trigger)
* [Configuring *Log4j*](#configuring-log4j)
* [Limitations and other consideration](#limitations-and-other-consideration)

<a name="introduction"></a>
## Introduction

Have you ever logged something like this in your code?

```java
  logger.warn("This shouldn't happen: state={}", state)
```

Indeed `WARN` logging level is often used to indicate that something odd has happened in your application logic, but
not too critical that you can't recover from (in that case `ERROR` level would be appropriate).
Almost by definition this is about rare events that are hard to reproduce.
Developers have nothing better than to log as much detail as possible about the oddity, so when it happens they would
have at least some data in the logs to help.

But how do you know that specific rare event occured in your application?
Regularly checking the logs is not an option for something that happens once in a blue moon.
Especially when logs are huge, numerous and are stored in restricted locations.
[Advanced logs management solutions](https://www.elastic.co/what-is/elk-stack) can definetely help, but
such tools are often not avaialble to developer, and when they are, their learning curve is quite steep.

Are there alternatives at the intersection of logging and notificaton concerns that could provide a simpler  solution
to this problem?

Recently, I learned about great Slack's feature called [Workflows](https://api.slack.com/workflows) that when used in
combination with properly configured logging library can offer such alternative.

> Workflows are automated multi-step tasks or processes that can run right in Slack,
> or connect with other tools and services.
> Workflows in Slack can be as simple or as complex as you’d like, and typically don’t require writing any code.
>
> If you’re a developer or have some experience with code,
> you may want to create a workflow triggered by an event in an external
> service (like an internal tool your company uses) with a webhook.

[Webhook](https://slack.com/intl/en-ca/help/articles/360041352714-Create-more-advanced-workflows-using-webhooks) is an
auto-generated, unique URL that Slack provides for your Workflow to be triggered by POST'ing a JSON payload
containing Worflow's variables to it:

```
Slack will generate a unique request URL for your workflow once you publish it,
and you can configure your webhook to pass information to Slack via the HTTP request body.
Any data your webhook sends to Slack can be referenced in subsequent workflow steps by creating variables.
```

<a name="creating-and-configuring-slack-workflow-with-webhook-trigger"></a>
## Creating and configuring Slack Workflow with Webhook trigger

Next, I will guide you through the steps required to create and configure a Workflow with Webhook trigger in Slack,
so that when triggered it will post a given message to specific Slack channel:

1. select existing channel or create new (e.g. "#prod-issues")
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
14. Click on "Insert a viariable" link right below "Message Text" editor and select "text" variable added in step #9
15. In the editor click on a "Code block" icon to embed "text" variable inside it ![Send a message](Send_a_message_step.png)
16. Click on "Save"
17. Now that you're back to your Worlflow Builder dialog - click on "Publish"

You will be presented with "Your workflow is ready to use" dialog with unique Webhook URL. Copy it.

![Your workflow is ready to use](your_workflow_is_ready_to_use.png)

You can now post messages to your workflow's channel with this simple `curl` command:

```
% curl -X POST -H 'Content-type: application/json' --data '{"text":"Hello Workflow"}' https://hooks.slack.com/workflows/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
```

![Hello Workflow](Hello_Workflow.png)

<a name="configuring-log4j"></a>
## Configuring *Log4j*

To make your Java application able to send notifications to Slack, we can configure *Log4j*'s standard [`Http`](https://logging.apache.org/log4j/2.x/manual/appenders.html#HttpAppender) appender with customized `PatternLayout`, and make certain loggers use such appender.

The following *Log4j* configuration makes any child logger of the "com.starsgroup.ffs" category and `WARN` logging level
to send messages with Slack Webhook (in addition to Console appender):
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

If, instead, you want a dedicated Slack logger to be used with any logging level, you can replace `<Loggers>` section
with this: 

```
<Loggers>
    <Logger name="SlackProdMonitorWebhook" level="debug">
        <AppenderRef ref="SlackProdMonitorWebhook"/>
    </Logger>

    <Root level="info">
        <AppenderRef ref="Console"/>
    </Root>
</Loggers>
```

This is how you access *SlackProdMonitorWebhook* logger in your code: 

```kotlin
val logger = LogManager.getLogger("SlackProdMonitorWebhook")
logger.info("something terrible has happened")
```

<a name="limitations-and-other-consideration"></a>
## Limitations and other consideration

Webhook workflows are limited to [one request per second](https://api.slack.com/docs/rate-limits#overview).
Obviously, loggers that make use of Slack Webhook should only be used to notify about rare events.

Also be mindful of the fact that *Log4j* *Http* appender blocks execution of application code for duration of
network call. It's important to enusre that appender's configured connection and read timeouts are appropriate for your
application. If latecny is critical, consider using *Log4j* [`<Async>`](https://logging.apache.org/log4j/2.x/manual/appenders.html#AsyncAppender) appender to wrap `Http` one.

Another important consideration is privacy. Do not log data that may be considered as private or otherwise senistive
information.

Lastly, due to limitation of `PatternLayout` to define Webhook's JSON request body, logger's methods that take
instance of `Throwable` as last argument are not supported (stacktrace comes after JSON and causes parsing error on
Slack side).
