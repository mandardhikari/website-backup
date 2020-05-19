---
ID: 73
post_title: >
  Understanding BizTalk Oracle Adapter
  Perfmon Counter and Removing Performance
  Bottlenecks by Creating Low Latency
  System
author: Mandar Dharmadhikari
post_excerpt: ""
layout: post
permalink: >
  https://theabodeofcode.com/understanding-biztalk-oracle-adapter-perfmon-counter-and-removing-performance-bottlenecks-by-creating-low-latency-system/
published: true
post_date: 2018-07-11 07:02:13
---
<h1>Problem</h1>
Recently I was working on a complex project where I was working on an orchestration which executes a stored procedure in an oracle database. The stored procedure has a simple select query that brings back data for a particular id. We executed the stored procedure from BizTalk using standard WCF-Custom adapter with oracle binding. The stored procedure when executed from a client like sqldeveloper, would take some where  around 600 ms to a second, but when the same was executed from BizTalk application it would take some where around 10 to 20 secs on average. Now as our application was supposed to be a fast application and the whole process of BizTalk was supposed to take 20 secs but due to the stored procedure issue the call was taking around 25 secs and was unacceptable according to business.

So I embarked upon the journey to check the performance issues with the stored procedure execution and to fix them using whatever means necessary viz code changes or performance tuning.

Following are my observations and findings which include
<ol>
	<li>How to trace performance counter of oracle adapter</li>
	<li>Understanding the actual meaning and behavior of the performance counter</li>
	<li>Identifying the bottleneck  and finally,</li>
	<li>Fixing the bottleneck by tweaking the Host properties.</li>
</ol>
<h1>How To Enable Performance Counters for Oracle adapter?</h1>
As per the MSDN documentation, the oracle adapter for BizTalk only has one performance counter "LOB Time" and this is defined as

"The <strong>BizTalk .NET Adapter for Oracle DB</strong> category has one performance counter called the “LOB Time (Cumulative).” This performance counter denotes the time, in milliseconds, that the LOB client library takes to complete an action that the adapter initiates"

source: <a href="https://docs.microsoft.com/en-us/biztalk/adapters-and-accelerators/adapter-oracle-database/use-performance-counters-with-the-oracle-database-adapter" target="_blank" rel="noopener noreferrer">MSDN Documentation: Use performance counters with the Oracle Database adapter </a>

Let us take a look at how to enable the performance counter and how to collect it in data collector set.

The performance counter collection should be enabled on the send port level. Following snap shot shows how to do that

<img class="alignnone size-full wp-image-74" src="https://theserverlessspirit.files.wordpress.com/2018/07/samplesendport1.jpg" alt="SampleSendport1" width="640" height="584" />

As we are using the oracle adapter with a BizTalk application the <strong>enableBizTalkCompatibilityMode</strong> should be set to <strong>True</strong>. Also <strong>enablePerformanceCounters</strong> when set to <strong>True</strong> will allow us to collect the performance counter <strong>LOB Time</strong> when the actual stored procedure is invoked by the adapter.

In order to track the LOB Time and save data collection, we need to set up a data collector set.

Following is a step by step process to set up the data collector set.
<ol>
	<li>Go to server Manger and click on Performance<img class="alignnone  wp-image-75" src="https://theserverlessspirit.files.wordpress.com/2018/07/datacollection1.jpg" alt="DataCollection1" width="910" height="479" /></li>
	<li>Click on the User Defined under the Data Collector Set to create a new data collector set.<img class="alignnone size-full wp-image-76" src="https://theserverlessspirit.files.wordpress.com/2018/07/datacollection2.jpg" alt="DataCollection2" width="678" height="375" /></li>
	<li>Give a meaningful name to the collector set and select the Create Manually(Advanced) option and click on Next<img class="alignnone size-full wp-image-77" src="https://theserverlessspirit.files.wordpress.com/2018/07/datacollection3.jpg" alt="DataCollection3" width="548" height="400" /></li>
	<li>Select the Create data logs option and check the Performance Counter check box<img class="alignnone size-full wp-image-78" src="https://theserverlessspirit.files.wordpress.com/2018/07/datacollection4.jpg" alt="DataCollection4" width="539" height="443" /></li>
	<li>Click on the highlighted Add option to see the list of available performance counters.<img class="alignnone size-full wp-image-79" src="https://theserverlessspirit.files.wordpress.com/2018/07/datacollection5.jpg" alt="DataCollection5" width="539" height="403" /></li>
	<li>Select the LOB Time(Cumulative) performance from under the BizTalk .NET Adapter for Oracle DB and click on Add. It gets added to the Added Counters section on left and your data collector will collect the data for this perfmon counter now.<img class="alignnone size-full wp-image-80" src="https://theserverlessspirit.files.wordpress.com/2018/07/datacollection6.jpg" alt="DataCollection6" width="708" height="530" /><img class="alignnone size-full wp-image-81" src="https://theserverlessspirit.files.wordpress.com/2018/07/datacollection7.jpg" alt="DataCollection7" width="708" height="525" /></li>
	<li>Once you click on Ok, you will be asked where the collected data needs to be saved. Save it as per your convenience.<img class="alignnone size-full wp-image-82" src="https://theserverlessspirit.files.wordpress.com/2018/07/datacollection8.jpg" alt="DataCollection8" width="541" height="402" /></li>
	<li>Click on Finish and it will ask how to run the data collector set, since I am already an admin on my machine, I allow it to run as default.<img class="alignnone size-full wp-image-83" src="https://theserverlessspirit.files.wordpress.com/2018/07/datacollection9.jpg" alt="DataCollection9" width="548" height="406" /></li>
	<li>Once clicked on Finish the collector will be available under the User Defined Data Collectors. Once you are ready to collect the data you can start the collector<img class="alignnone size-full wp-image-84" src="https://theserverlessspirit.files.wordpress.com/2018/07/datacollection10.jpg" alt="DataCollection10" width="1230" height="565" /></li>
</ol>
&nbsp;
<h1>Understanding the actual meaning and behavior of the performance counter</h1>
As mentioned on the MSDN documentation the Oracle adapter has only one counter called as LOB Time which measures the time taken by the LOB client library to perform the action initiated by the adapter and it is tracked in milliseconds. But notice the <strong>Cumulative </strong>added to the performance counter name. The reason is that this counter tracks the cumulative time for the next iteration.  Let us see what this means

Let us say that a call to an oracle stored procedure is initiated at 12:00:00 PM so at start the counter value will be o ms. Once the call gets complete. the  counter value gets updated say it takes 200 ms to complete the action. so the counter reaches the value of 200 ms now. So when the next call happens at that point the counter will not start again from zero, it will be constant on 200 ms and the next call duration gets added to 200 ms. So say second call takes another 200 ms then in that case counter becomes stable at 400 ms.

So as opposition to a spiked behavior of the counter what we see is a step behavior, hence the counter is a cumulative counter as at any current point of time, it denotes the cumulative time from all the previous time. Following snap shot which tracks one cumulative LOB time will make the concept clear.

<img class="alignnone size-full wp-image-86" src="https://theserverlessspirit.files.wordpress.com/2018/07/lobtimeout.jpg" alt="LOBTimeOut" width="819" height="460" />
So after this I started the collection of the data on the box and fired up some requests to the BizTalk application.

Now from tracked data , I calculated the time duration between the send and receive activity of the oracle send port and the time captured using the performance counter. Following is sample data(LOB time is the difference between to steps in performance counter)
<table width="487">
<tbody>
<tr>
<td width="296">Port Execution Duration as noted from DTA Db(ms)</td>
<td width="191">LOB Time(ms)</td>
</tr>
<tr>
<td width="296"> 20500</td>
<td width="191">1529</td>
</tr>
<tr>
<td width="296"> 7000</td>
<td width="191">1607</td>
</tr>
<tr>
<td width="296"> 5800</td>
<td width="191">1451</td>
</tr>
<tr>
<td width="296"> 19300</td>
<td width="191">1497</td>
</tr>
</tbody>
</table>
As seen there is vast difference between the execution times tracked from DTA and the LOB time, it became clear to me that there is nothing wrong with the performance of the oracle adapter whatsoever. <strong>so next task was to fins out what was wrong and the big question was where??</strong>

After doing a bit of reading around the BizTalk performance counters, I came across following counters available under the <strong>BizTalk:Messaging Latency</strong> object.

<strong>Outbound Adapter Latency</strong> : This performance counter is available under the <strong>BizTalk:Messaging Latency</strong> object and is described in MSDN documentation as

"Average latency in milliseconds from when the adapter gets a document from the Messaging Engine until the time it is sent by the adapter."

<strong>Inbound Latency: </strong> This performance counter is available under the <strong>BizTalk:Messaging Latency</strong> object and is described in MSDN documentation as

"Average latency in milliseconds from when the Messaging Engine receives a document from the adapter until the time it is published to Message Box"

These performance counters were enabled(Only for the send handler attached to the port executing oracle stored procedure under question) and then again data was collected using the collector sets. Following is a snap shot of all the three performance counters collected together.

<img class="alignnone size-full wp-image-87" src="https://theserverlessspirit.files.wordpress.com/2018/07/completeperfmontracking.jpg" alt="CompletePerfmonTracking.JPG" width="711" height="310" />

As it is clear each time there is a call to the stored procedure there will simultaneous peaks for the Outbound Adapter Latency and the Inbound Latency. Let us take a look at the data collected.
<table width="587">
<tbody>
<tr>
<td width="233">Port Execution Duration  from DTA Db(ms)</td>
<td width="64">LOB Timeout(ms)</td>
<td width="100"> Outbound Adapter Latency(ms)</td>
<td width="95">Inbound Latency(ms)</td>
<td width="95">Total Duration(ms)</td>
</tr>
<tr>
<td width="233">  20500</td>
<td width="64"> 1529</td>
<td width="100"> 16941</td>
<td width="95">2106</td>
<td width="95">20576</td>
</tr>
<tr>
<td width="233"> 7000</td>
<td width="64"> 1607</td>
<td width="100"> 4196</td>
<td width="95">1202</td>
<td width="95">7005</td>
</tr>
<tr>
<td width="233">  5800</td>
<td width="64"> 1451</td>
<td width="100"> 3947</td>
<td width="95"> 437</td>
<td width="95"> 5835</td>
</tr>
<tr>
<td width="233">  19300</td>
<td width="64"> 1497</td>
<td width="100">16302</td>
<td width="95">1529</td>
<td width="95">19328</td>
</tr>
</tbody>
</table>
Now when we add the LOB Time, Outbound Adapter Latency and the Inbound Latency for a particular stored procedure execution they match up with the data recorded in the DTA table.

<strong>So Finally !!! Success , well partly because I found out that the issue was due to the Latency in BizTalk messaging. Bottleneck identified and half the battle is already won</strong>.
<h1>Fixing the BottleNeck</h1>
Armed with observations made from the performance counters, I took on the task of the fixing the bottle neck. Theoretical solution to the latency problem is that you create a low latency system, in BizTalk that would translate to "Create low latency Hosts". Now after reading up a bit on internet about how to create low latency hosts, I came across following posts
<ol>
	<li><a href="https://docs.microsoft.com/en-us/biztalk/core/using-settings-dashboard-for-biztalk-server-performance-tuning" target="_blank" rel="noopener noreferrer">Use Settings Dashboard for BizTalk Server Performance Tuning</a></li>
	<li><a href="https://docs.microsoft.com/en-us/biztalk/core/how-to-modify-host-settings" target="_blank" rel="noopener noreferrer">Update BizTalk host settings</a></li>
</ol>
<strong>First of all the most important thing we did was that we downgraded the tracking in the BizTalk application. We only kept the minimum shape start and end times tracking enabled other than that everything was shut off</strong>

Next after reading through the links above I started playing around with following values
<h3><strong>Under the General Settings</strong></h3>
Refer <a href="https://docs.microsoft.com/en-us/biztalk/core/how-to-modify-general-settings" target="_blank" rel="noopener noreferrer">How to Modify General Settings</a>

I toyed around with the polling interval values as they are in direct correlation to how the BizTalk engine polls the data from the message box. There are two polling intervals available under general settings
a) Message Polling Interval
b)Orchestration polling interval, both these values are in ms.
Let us take a look at the description on MSDN site
<img class="alignnone size-full wp-image-89" src="https://theserverlessspirit.files.wordpress.com/2018/07/generalcounterssettings.jpg" alt="GeneralCountersSettings.JPG" width="939" height="264" />
This presents following understanding
<ol>
	<li>For a Handler(Host Used on Send Ports)- Keep the messaging polling interval as little as possible and keep the Orchestrations polling interval to a high value.</li>
	<li>For an Application Host(Host Tied to Orchestrations): Reverse the above settings</li>
</ol>
<h3>Under Resource Based Throttling</h3>
Refer <a href="https://docs.microsoft.com/en-us/biztalk/core/how-to-modify-resource-based-throttling-settings" target="_blank" rel="noopener noreferrer">How to Modify Resource Based Throttling Settings</a>

The main settings that play an important role in the latency scenario are

a)<strong class="">In-process messages</strong>

b)<strong>Internal message queue size </strong>

Let us take a look at what these settings are .

<img class="alignnone size-full wp-image-90" src="https://theserverlessspirit.files.wordpress.com/2018/07/resourcebasedthrottlingsettings.jpg" alt="ResourceBasedThrottlingSettings.JPG" width="853" height="383" />

So we need an acceptable value higher that the default values specified for these settings.

Now we need to understand that there is no right or wrong or a fixed value that we can use for the settings discussed above. We need to play around with these values and see what works for us

This is what I ended up doing in my environment

For  A Handler ( Host attached to Send Ports)
<ol>
	<li>Messaging Polling interval changed from 500 to 50 ms</li>
	<li>Orchestration polling interval changed from 500 to 150000 ms</li>
</ol>
For an Application Host (Host attached to Orchestrations)
<ol>
	<li>Messaging Polling interval changed from 500 to 150000 ms</li>
	<li>Orchestration Polling Interval changed from 500 to 50 ms</li>
</ol>
And for all the hosts related to this application, following changes were made in the resource based throttling.
<ol>
	<li>In-Process messages : Changed from 1000 to 10000</li>
	<li>Internal Message Queue changed from 100 to 1000</li>
</ol>
After these changes following were the observations
<table width="584">
<tbody>
<tr>
<td width="178">Parameter(ms)- Avg</td>
<td width="176">Without Low Latency Hosts</td>
<td width="155">With Low Latency Hosts</td>
<td width="75">%  Gain</td>
</tr>
<tr>
<td>SP execution Time</td>
<td>13150</td>
<td>1100</td>
<td>94</td>
</tr>
<tr>
<td>Outbound Adapter Latency</td>
<td>10347</td>
<td>100</td>
<td>99</td>
</tr>
<tr>
<td>Inbound Latency</td>
<td>1318</td>
<td>70</td>
<td>94.68</td>
</tr>
<tr>
<td>LOB Time</td>
<td>1521</td>
<td>500</td>
<td>67.12</td>
</tr>
</tbody>
</table>
<h1>Learnings</h1>
<ol>
	<li>Enable tracking only when required. But to track the basic flow, keep the bare minimum tracking enabled</li>
	<li>Creating low latency hosts when fast responses are expected.</li>
	<li>How the LOB Time counter works</li>
</ol>