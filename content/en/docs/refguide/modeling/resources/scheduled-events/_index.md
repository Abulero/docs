---
title: "Scheduled Events"
url: /refguide/scheduled-events/
parent: "resources"
weight: 80
description: "Options for configuring scheduled events"
tags: ["Scheduled Event", "Execution properties", "Timing", "intervals", "scheduling issues", "time zones", "daylight saving"]
#If moving or renaming this doc file, implement a temporary redirect and let the respective team know they should update the URL in the product. See Mapping to Products for more details.
---

## 1 Introduction

With scheduled events you can let the runtime execute a microflow at specific moments in time.

A scheduled event is added to your module as a document (right-click your module and you will find it listed under *Add other*).

{{% alert color="warning" %}}
Scheduled events can be tested locally, but they will not be run if your app is deployed as a Free App. See the Free App section of [Mendix Cloud](/developerportal/deploy/mendix-cloud-deploy/#free-app) for more information on Free App limitations.
{{% /alert %}}

## 2 Variants

Until Mendix 9.12.0 scheduled events only execute on the cluster leader node, with an at-most-once guarantee, which means that they
will not be executed if something happens to the cluster leader node. Also, these scheduled events do not support proper monthly and
yearly events and potentially suffer from a shift of one hour due to daylight saving. These 'legacy' scheduled events are
deprecated and will no longer be supported from Mendix 10.

As of Mendix 9.12.0 scheduled events have been improved and are being executed using the [task queue](/refguide/task-queue/), providing an
at-least-once guarantee. They will be executed by an arbitrary node in the cluster and support monthly and yearly events properly.
Most importantly, the task queue-based scheduled events no longer have a start-time, but allow specifying a particular time of day,
ensuring that they work as expected in the face of daylight saving.

Mendix versions 9.12.0 and above supports the following schedule types:

* **Legacy** – the type that existing before Mendix 9.12.0 — see [Legacy scheduled events](/refguide/scheduled-events-legacy/)
* **Yearly** – an event that occurs once a year on a particular day or date and at a specified time — see [Yearly scheduled events](/refguide/scheduled-events-task-queue/#yearly)
* **Monthly** – an event that occurs every so many months on a particular day and at a specified time — see [Monthly scheduled events](/refguide/scheduled-events-task-queue/#monthly)
* **Weekly** – an event that occurs weekly on particular days and at a specified time — see [Weekly scheduled events](/refguide/scheduled-events-task-queue/#weekly)
* **Daily** – an event that occurs daily at a specified time — see [Daily scheduled events](/refguide/scheduled-events-task-queue/#daily)
* **Hourly** – an event that occurs every so many hours — see [Hourly scheduled events](/refguide/scheduled-events-task-queue/#hourly)
* **Minutes** – an event that occurs every so many minutes — see [Minute scheduled events](/refguide/scheduled-events-task-queue/#minutes)

## 3 Migration{#migration}

When migrating to Mendix version 9.12.0 or above, Studio Pro will attempt to convert legacy scheduled events into task queue-based events,
when possible.

The following cases cannot be converted automatically when the model is upgraded to Mendix version 9.12.0 or above:

* The event is not repeating — remove the scheduled event or use the [Java API](/refguide/task-queue/#queuing) to schedule a one-time action — we no-longer support non-repeating scheduled events.
* The event has a start-time in the future, which we'll stop supporting — change the start-time to a date in the past or switch to a task queue based scheduled event.
* The event has interval type Month or Year, which is translated to 31 and 365 days respectively — use the Monthly or Yearly type instead.
* The event has interval type Seconds — use a schedule event with a 1-minute interval instead — we no-longer support scheduled events which repeat in less than a minute.
* The event has an interval that does not divide precisely into its next biggest interval type, for example an event that executes every 7 minutes; this will execute 8 times per hour, with 4 minutes left, causing it to 'drift' 4 minutes every hour.

    If it is absolutely critical that a non-supported interval is used, you should schedule the event with interval value of 1 (every minute) and then start your microflow with a decision that checks whether it should continue executing at that particular time.

{{% alert color="warning" %}}
Mendix 9.12.0 and above allow legacy scheduled events to be edited, but do not allow new legacy scheduled events to be created.
{{% /alert %}}
