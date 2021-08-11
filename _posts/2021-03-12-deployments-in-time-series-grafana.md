---
title: Tracking deployment and maintenance activity in time series
date: 2021-03-12
tags:
  - Tech
  - Grafana
---

We use a popular time-series stack (Telegraf / InfluxDB / Grafana) to collect and visualize performance data across our infrastructure. When we began collecting system stats, an early blind spot emerged: we did not have a simple way to associate developer activity with system performance.

The problem with infrastructure performance data, is that it only shows you the *effects* of application activity- leaving you to speculate about the causes. In an effort to help narrow this visibility gap, we devised an elegant way to plot deployment history alongside our system performance graphs.

One of the nice things about the InfluxDB time-series database, is that there are [many ways](https://docs.influxdata.com/influxdb/v2.0/write-data/developer-tools/) to feed data into it- including a [web API](https://docs.influxdata.com/influxdb/v2.0/write-data/developer-tools/api/). This means you can push your own metrics into it directly, using a simple `curl` statement:

{% raw %}
```shell
curl --request POST "http://YOUR_INFLUX_SERVER:8086/api/v2/write?org=YOUR_ORG&bucket=YOUR_BUCKET&precision=s" \
  --header "Authorization: Token YOURAUTHTOKEN" \
  --data-raw "
deployments,host=host1,environment=test activity="deployment-app1-1.2.34",status="success",failed=False
"
```
{% endraw %}

There are examples online for doing the same with python, powershell, etc.

We created a deployment status script that accepts basic arguments for the host, environment, application, activity, and status. We worked with development teams to add this script to the end of their CI/CD pipelines and deployment playbooks.

The result is a record of deployment activity that we can plot alongside a system's performance graphs in Grafana.

![2021-03-12-grafana-2.png](/assets/images/2021-03-12-grafana-2.png)

## Making the deployment data useful

Data tables are a start, but they aren't very useful on their own. How about combining them with our performance graphs?

![2021-03-12-grafana-1.png](/assets/images/2021-03-12-grafana-1.png)

The blue and green vertical bars are deployments that ran against this host, and the moving line represents database connections opened by applications on the server.

Notice anything odd about the graph? Immediately after the "blue" deployment on the right, the database connections shot to 100 and stayed there. In this case, some manual configuration changes were made to troubleshoot connectivity issues on the host... but the developer forgot to add those changes back into their code. As a result, the next deployment bulldozed their manual config changes, and the connection issues reappeared.

This kind of overlay technique has been very useful in flushing out mysterious bugs, configuration drift, etc. With just a little bit of scripting knowledge, you can apply the same layer of visibility to other processes as well:

* Job schedulers
* Patching or maintenance scripts
* Ansible playbook runs, either as a task at the end of the playbook or as a callback plugin
* CI/CD pipelines

## Obligatory "How did you do that in Grafana?"
If you run Grafana at your workplace and you are wondering how we achieved the above overlay effect, see below for an example:
![2021-03-12-grafana-3.png](/assets/images/2021-03-12-grafana-3.png)

In our graph panel's settings, we use the following series overrides and customizations for the deployment data:

* Use the right-hand y-axis
* Enable bars and disable lines
* Make the bar color semi-transparent
* For the right-hand Y axis, the upper limit is forced to 1- which makes any deployment activity appear as a vertical bar that cuts through the graph- giving us a clear and consistent marker for deployments.
* The deployment metrics are grouped into 1-hour "chunks" in the query, which makes the resulting deployment bars thicker and easier to see when the timeline is zoomed out.
* If your query for deployments gives you data points that are aliased by app name or environment, you can use series overrides with pattern matching to color-code your deployments.

