# DQL Queries

## Basic

### Use Case 1 (provided by Kale) - maybe this is also an intermediate use case:
- Use Case Description: 
Dear Product Specialist. 
We are a big Travel Company from Reykjavík in Iceland.
There is an important request from our EasyTravelBackendDev Team where we need your help.
We have a process called "com.dynatrace.easytravel.business.backend.jar easytravel-*-x*". 
On May, 24th a few customer complained that the transaction does not work after they input the credit number. This potentially made us lose lots of money.
We heard about the awesome capabilities of Dynatrace, but we are not sure how to leverage them efficiently.
Is there some dql query that may help us to find error logs linked to the process that has some information related to credit card payments?
Can you help us write a query to find the errors?

```
fetch logs, timeframe:"2023-05-24T00:00:00Z/2023-05-24T23:59:59Z"
| filter contains(dt.process.name, "com.dynatrace.easytravel.business.backend.jar easytravel-")
| filter matchesValue(loglevel, "ERROR")
```

This is awesome! Our manager needs a report about the impact of this issue. May you please count the number of error logs that are associated with the credit card issue that day? Please exclude the events without an associated credit card issue.

```
fetch logs, timeframe:"2023-05-24T00:00:00Z/2023-05-24T23:59:59Z"
| filter contains(dt.process.name, "com.dynatrace.easytravel.business.backend.jar easytravel-")
| filter matchesValue(loglevel, "ERROR")
| parse content, "DATA 'was not valid: ' LONG:cardNumber ','"
| fields cardNumber, content, timestamp
| sort cardNumber asc
| filter isNotNull(cardNumber)
| summarize count(), alias: total
```

Hints:
- use the right timeframe (think about the place the customer is from and see if there is an offset to your timezone, UTC timezone)
- be careful with the format of the credit card number (using INT in the DPL cannot extract the number correctly, use LONG or DOUBLE)

### Use Case 2 (provided by Danilo - timeseries):
Furthermore, there is an important request from our infrastructure team who wants to make sure this doesn't happen again in the future. The underlying host for a process "com.dynatrace.easytravel.business.backend.jar easytravel--x" is "ls-ub-lb4ac00v.dev.lab.dynatrace.org". We need you to make sure the infra team is always aware of the current CPU usage. Please make sure you come up with a dql query that will always contain the current CPU usage for this host.

```
timeseries avg(dt.host.cpu.load), by:{dt.entity.host}, interval: 1h
| filter dt.entity.host == "HOST-6476B9B0E0F2A993"
```

That's great, however, the team wants to know the current memory usage as well for this host. Could you write a query for that as well, please?

```
timeseries avg(dt.host.memory.usage), by:{dt.entity.host}, interval: 1h
| filter dt.entity.host == "HOST-6476B9B0E0F2A993"
```

### Use Case 3 (provided by Dylan - bizevents):
Hello Dynatrace, there are a few key business metrics our application team is looking to track. The team wants to know how many unique users are accesing the Easy Trade application over the last week. We currently are tracking the accounts using a "com.easytrade.get-account-by-id" business event. 

```
fetch bizevents, from:now()-7d
| filter event.type == "com.easytrade.get-account-by-id"
| summarize unique_users=countDistinct(email)
```

We also are very interested in knowing the total number of buys that have been completed in the last week. This is something we started to set up in our "com.easytrade.buy-assets" business event. 

```
fetch bizevents, from:now()-7d
| filter event.type == "com.easytrade.buy-assets"
| summarize count=count()
```
Along with the total count, it would be great insight for our team if we knew the average volume per buy as well. These two metrics over the last week would be helpful but also if possible, being able to break it down per day on a chart would be best.

```
fetch bizevents, from:now()-7d
| filter event.type == "com.easytrade.buy-assets"
| summarize count=count(), avg=avg(trading_volume), by:{bin(timestamp,1d)}
```

### Use Case 4 (Provided by Marcelo):
Hi team! We have a node JS process running on EKS and have been experiencing some issues. I've been noticing some errors in the logs, and I have been trying to make a DQL query to look at the different types of 4xx errors coming in. I haven't quite figured out how to pull the code out from the content of the logs, the name of the process is "element-cli unguard-user-simulator-*". Thanks!

```
fetch logs
| fields timestamp, content
| filter(dt.process.name == "element-cli unguard-user-simulator-*") and contains(content, "Failed")
| parse content, "LD INT:code "
```
Great! Would we be able to see how many of each type there are here? It looks like I see 403, 404, and 409 codes.

```
fetch logs
| fields timestamp, content
| filter(dt.process.name == "element-cli unguard-user-simulator-*") and contains(content, "Failed")
| parse content, "LD INT:code "
| summarize err_one = countIf(code  ==  403),
            err_two = countIf(code == 404),
            err_three = countIf(code == 409)
```

## Intermediate
### Use Case 1 (provided by Kale):
A customer, from Inteligencia AG, who has a focus in the Data Science domain started using Dynatrace for Log Analysis. They want to parse the content and extract 2 words from it. As an example they provided a sample content string that you can use for testing and providing an answer. The string "Test Alarm", representing the example, should be extracted here:
```
4/25/2023 10:00:04 AM Test Alarm CMD 59\r\nValue True \r\nSite Name ntapth7186m00\r\nPrevious Status Normal\r\n
```
Hints:
- If you are already familiar with Notebooks, try the new UI for fetching the logs. Furthermore, the DPL Architect will help you write the DPL parsers.
- as we do not have the data within our environment, first try to add the data to a sample log that you fetch - e.g. like so:
```
fetch logs
| limit 1
| fieldsAdd content2 = """<CONTENT>"""
| fields content2
```
- After showing the data correctly, continue with parsing the content example and help the customer achieve their goal.
 
```
fetch logs
| limit 1
| fieldsAdd content2 = """4/25/2023 10:00:04 AM Test Alarm CMD 59\r\nValue True \r\nSite Name ntapth7186m00\r\nPrevious Status Normal\r\n"""
| fields content2
| PARSE content2, "TIMESTAMP('M/dd/yyyy HH:mm:ss a'):parsed_timestamp BLANK? (LD SPACE LD):extractedWords SPACE"
```


### Use Case 2 (provided by Dylan):
We at ABC company are looking to idenfity response times from their application logs. However, the response time is nested inside of the content of their logs (in the "easytrade" k8s namespace). Can you help write a DQL query to analyze these logs to find the average response time?

```
fetch logs
|filter k8s.namespace.name=="easytrade" AND matchesPhrase(content, "Response time")
|parse content, "LD 'Response time [' INT:responsetime"
|summarize average = round(avg(responsetime),decimals: 2)
```

Now that we have the average response times we would like to get a little bit more information. We are trying to better understand how our actual performance is vs. our goal performance of 75ms. Please help write a DQL query to show us what percentage of calls met our performance goal. 

```
fetch logs
| filter k8s.namespace.name=="easytrade" AND matchesPhrase(content, "Response time")
| parse content, "LD 'Response time [' INT:responsetime"
| summarize success=countIf(responsetime < 75), total=count()
| fields success=toDouble(success), total=toDouble(total)
| fieldsAdd successRate= (100 * success / total)
| fields successRate
```

### Use Case 3 (provided by Kale):
A customer asks you to help him collect cpu usage information using the Grail-based metric (dt.host.cpu.usage) with the timeseries command. 
He is not sure how to use the new lookup command. The basic idea is to fetch the metric data, grouped by individual hosts. Furthermore, they would like to filter by the os type of the host. They want to focus on Linux hosts. However, this is not part of the metric/timeseries data. Therefore, a lookup is necessary to extend the timeseries/metric data with the data from dt.entity.host. First, try to connect the metric data with additional host data from dt.entity.host. Next, filter for the osType from dt.entity.host. 

```
timeseries avg(dt.host.cpu.usage), by:{dt.entity.host}
| lookup [fetch dt.entity.host], sourceField:dt.entity.host, lookupField:id
| fields `avg(dt.host.cpu.usage)`, dt.entity.host, lookup.id, lookup.osType
| filter lookup.osType=="LINUX"
```
### Use Case 4 (provided by Danilo - topology):
A customer is working on a new deployment and they need to make sure they have a summary of all hosts that belong to certain host groups. They are not sure how to leverage our newly developed 'lookup' command but would like to work with our smartscape topology as well as join the data with their respective host group to get the result. Furthermore, they would like to add 'tags' to each entity and see which tags have been applied. 
In order for them to be able to fetch hosts as well as their respective host groups, they have to make sure the host_group attribute is added to the query set and then use lookup to join them. Lastly, they can add the tags field as well. 
Note: Please make sure we only get the data set with host groups present. We do not want to see hosts where we have missing host groups. 

```
fetch dt.entity.host
|fieldsAdd host_group=instance_of[dt.entity.host_group]
|lookup [fetch dt.entity.host_group],sourceField:host_group,lookupField:id
|fieldsAdd tags
|filter isNotNull(host_group)
```



## Advanced
### Use Case 1 (provided by Kale):
A customer, from Inteligencia AG, who has a focus in the Data Science domain started using Dynatrace for Log Analysis. They want to parse the content and extract 2 words from it. As an example they provided a sample content string that you can use for testing and providing an answer. The string "NAID-andr::NAID-andr-2f11a1c7af82f0e5b99fd319f97046f0-1675142707719", representing the example, should be extracted here:
```
SYSLOG_HOST=aiops-splunkent-prd-50100-002-syslogng-journald02-1 CONTAINER_NAME=\"cncservice\" _COMM=\"dockerd-current\" SYSLOG_IDENTIFIER=\"\" MESSAGE=\"[ACCESS]	10.185.57.113	127.0.0.1	-	2023-01-30 23:25:08.190	POST	\"/v1/cart \"	500	67	108	\"kohls/8.0.6 (Android 13; es_US; SM-A716U; Build/23_1)\"	\"34.138.41.243\"	\"71.209.216.126\"	
[correlation-id::NAID-andr::NAID-andr-2f11a1c7af82f0e5b99fd319f97046f0-1675142707719::1675142708082::us-central1-b::cprodn-blue-cncservice-prod-xf9b::null::CnC::cart::46.0.1::************7812::null::50951014884130001::NONE::null::lazarom0315]	[ includeheaderfooter::-]\"
```
Hints:
- If you are already familiar with Notebooks, try the new UI for fetching the logs. Furthermore, the DPL Architect will help you write the DPL parsers.
- as we do not have the data within our environment, first try to add the data to a sample log that you fetch - e.g. like so:
```
fetch logs
| limit 1
| fieldsAdd content2 = """<CONTENT>"""
| fields content2
```
- After showing the data correctly, continue with parsing the content example and help the customer achieve their goal.
 
```
fetch logs //, scanLimitGBytes: 500, samplingRatio: 1000
| limit 1
| fieldsAdd content2 = "SYSLOG_HOST=aiops-splunkent-prd-50100-002-syslogng-journald02-1 CONTAINER_NAME=\"cncservice\" _COMM=\"dockerd-current\" SYSLOG_IDENTIFIER=\"\" MESSAGE=\"[ACCESS]	10.185.57.113	127.0.0.1	-	2023-01-30 23:25:08.190	POST	\"/v1/cart \"	500	67	108	\"kohls/8.0.6 (Android 13; es_US; SM-A716U; Build/23_1)\"	\"34.138.41.243\"	\"71.209.216.126\"	
[correlation-id::NAID-andr::NAID-andr-2f11a1c7af82f0e5b99fd319f97046f0-1675142707719::1675142708082::us-central1-b::cprodn-blue-cncservice-prod-xf9b::null::CnC::cart::46.0.1::************7812::null::50951014884130001::NONE::null::lazarom0315]	[ includeheaderfooter::-]\""
| fields content2
| parse content2, "DATA '::' (DATA '::' DATA):extracted2Parts '::'// modify this DPL pattern to extract fields"
```

### Use Case 2 (provided by Dylan):
A customer has been tasked with creating a capacity planning report for their linux hosts. They are looking to utilize DQL to look at the average CPU usage of their linux hosts for the day, focusd on the top 10 highest. The customer is interested in a quick way to see if any of their top hosts have breached their threshold of 30%. Please help provide a DQL query to create this report.

```
timeseries usageCPU = avg(dt.host.cpu.usage), by:{dt.entity.host}
| lookup[fetch dt.entity.host], sourceField:dt.entity.host, lookupField:id
| fields host=lookup.entity.name, avgCpuUsage=arrayAvg(usageCPU), os=lookup.osType
| filter os== "LINUX"
| fieldsAdd status=if(avgCpuUsage > 30,"🔴",else: "🟢")
| fieldsRemove os
| sort avgCpuUsage desc
| limit 10
```

### Use Case 3 (provided by Danilo - bizevents):
One of our biggest customers is currently working with bizevents. They are struggling with finding out a way to do a few things:
1. Fetch all events from a particular even provider which is "www.easytrade.com"
2. Filter on parcticular event types such as buying assets and depositing money (hint:event.type)
3. They need to make sure they can calculate the difference in time between depositing money and buying first assets (hint:summarize)
4. Make sure we are only filtering for fields that are above 0 (hint:summarized field > 0)
5. Select only fields we are interested in and remove the ones we don't want to see (fields command to filter out all noise)

```
fetch bizevents, from:now()-30d, to:now()
| filter event.provider == "www.easytrade.com"
| sort timestamp, direction:"descending"
| filter event.type == "com.easytrade.buy-assets" OR event.type == "com.easytrade.deposit-money"
| fieldsAdd deposit_ts = if(event.type == "com.easytrade.deposit-money", toLong(timestamp))
| fieldsAdd buy_asset_ts = if(event.type == "com.easytrade.buy-assets", toLong(timestamp))
| summarize {first_deposit_ts = takeFirst(deposit_ts), first_buy_asset_ts = takeFirst(buy_asset_ts)}, by:{accountId}
| fieldsAdd time_from_deposit_to_first_buy = (first_buy_asset_ts - first_deposit_ts)/(10000000000.0)
| filter time_from_deposit_to_first_buy > 0
| fields accountId, time_from_deposit_to_first_buy
```

# Workflow Example
## Use Case 1 (provided by Kale)
Our customer Nina, an ultra senior software engineer in her company, came back from her vacation and explained how life is beautiful since her company successfully rolled out and configured Dynatrace.
Now they have observability in all their mission-critical IT resources and when some incident happens, different teams are able to handle the issue, also when Nina, who used to be called in from vacation before when something happened to one of their systems, is not there.

She is intrigued by the beauty of DQL and having all data within Grail just one query away.
After she mastered the basic of DQL, thanks to your help, she wants to try out the workflow capability with a small example using Slack.
In order to help her out and provide the amazing customer experience as you have done before, create a demo workflow that does the following:

### On the demo live environment open the Workflow app, create a new workflow called "YOURTEAMNAME_Slack_Demo" and do the following:
- trigger: on-demand
- fetch 5 lines of logs
- send the logs to the Slack Sandbox - channel: welcome-test

The hard part is sending the data successfully to slack - try making use of the expression reference (https://www.dynatrace.com/support/help/shortlink/automations-reference#for-loop)
   
Solution:
   
 dql query:
```
fetch logs
| fields content
| limit 5
```

Slack Action Message (only use 3 backticks (`) before and after the message, i.e. ignore the first 2 backticks in the first and last line:
```
`` ```   
*Logs from YOUR_TEAM_NAME*
   
{% for log in result("execute_dql_query_1").records %}
    - {{ log.content }}
{% endfor %}
`` ```
```
