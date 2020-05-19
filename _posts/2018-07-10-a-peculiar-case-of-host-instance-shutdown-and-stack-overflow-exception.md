---
ID: 60
post_title: >
  A peculiar case of Host Instance
  Shutdown and Stack Overflow Exception
author: Mandar Dharmadhikari
post_excerpt: ""
layout: post
permalink: >
  https://theabodeofcode.com/a-peculiar-case-of-host-instance-shutdown-and-stack-overflow-exception/
published: true
post_date: 2018-07-10 05:26:52
---
<h1>Scenario</h1>
Recently I was working on a complex BizTalk orchestration which calls multiple services and has to catch and parse the particular responses and exceptions received during the service call. The parsing is taken care of by a helper method to which we passed the unique response ids/ soap fault caught on service port and depending upon the business rules a particular error code was applied after reading the fault details.

When testing the code I found out that the BizTalk host instance was getting shut down and the orchestration halted all the processing. I tried restarting the host instance and again same thing would happen and it would shut down.

This helper call was after the response from an oracle stored procedure was received and based upon response Business rules for errors were applied and the response was to be sent to other orchestration.
<h1>Observations</h1>
Now checking the tracked service instances and the messages was of no use at all as BizTalk was not inserting the tracked instances as there was a send port ahead (after the response from oracle was received) and hence a persistence was at that point so no help from there.

I got following error at first in the  application event log.

<img class="alignnone size-full wp-image-64" src="https://theserverlessspirit.files.wordpress.com/2018/07/applicationerror.jpg" alt="ApplicationError" width="629" height="597" />

The error in itself is not very enlightening, it does not tell the exact reason for the application error and does not even tell about the faulting module.

But a few seconds later windows recorded a new information, yes information and not error in the event log. But this was the one which pointed me in the correct direction. The information is as follows

<img class="alignnone size-full wp-image-65" src="https://theserverlessspirit.files.wordpress.com/2018/07/windowserrorreporting.jpg" alt="WindowsErrorReporting" width="750" height="739" />

Now this is very enlightening as it told me the culprit project and the exception that occurred, yes it was the StackOverflowException.

With this information in hand it became easy for me to track the exception, I was aware of the code called from the helper class in the orchestration. So I attached the helper method to the Host Instance process running the orchestration and it gave me the exact idea why the error happened.

My helper method was like below
<div class="reCodeBlock" style="border:1px solid #7f9db9;overflow-y:auto;">
<div style="background-color:#ffffff;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">static</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:#000000;">ParseError(</code><code style="color:#006699;font-weight:bold;">string</code> <code style="color:#000000;">operation, </code><code style="color:#006699;font-weight:bold;">string</code> <code style="color:#000000;">correlationID, </code><code style="color:#006699;font-weight:bold;">string</code> <code style="color:#000000;">responseID, </code><code style="color:#006699;font-weight:bold;">string</code> <code style="color:#000000;">errorCode, </code><code style="color:#006699;font-weight:bold;">string</code> <code style="color:#000000;">identifier)</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:32px !important;"><code style="color:#000000;">{</code></span></div>
<div style="background-color:#ffffff;"><code>            </code><span style="margin-left:48px !important;"><code style="color:#006699;font-weight:bold;">return</code> <code style="color:#000000;"><code>ParseError</code>(operation, correlationID, responseID, errorCode, identifier);</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:32px !important;"><code style="color:#000000;">}</code></span></div>
</div>
&nbsp;

As clear from the method, I was calling the same method recursively which created an infinite loop call and the stack overflow exception was thrown. The solution became very easy after this, I just fixed the method and I was back in business.
<h1>Conclusion</h1>
The learning from this incidence was that Application event log is your friend and always explore the application event log to learn more about the errors and issues.