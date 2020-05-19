---
ID: 98
post_title: 'BizTalk DTA Purge And Archive Job Error: Unspecified error occurred on SQL Server.  Connection may have been terminated by the server. [SQLSTATE HY000] (Error 0)   Unable to find index entry in index ID 1, of table xyz, in database &#8216;BizTalkDTADb&#8217;.  The indicated index is corrupt or there is a problem with the current update plan. Run DBCC CHECKDB or DBCC CHECKTABLE.  If the problem persists, contact product support. [SQLSTATE HY000] (Error 8646).  The step failed.'
author: Mandar Dharmadhikari
post_excerpt: ""
layout: post
permalink: >
  https://theabodeofcode.com/biztalk-dta-purge-and-archive-job-error-unspecified-error-occurred-on-sql-server-connection-may-have-been-terminated-by-the-server-sqlstate-hy000-error-0-unable-to-find-index-entry-in-index/
published: true
post_date: 2018-08-21 04:07:47
---
<h1>Problem Statement</h1>
Today when I ran the BHM to check the health of my BizTalk development box, I found that the BizTalk job DTAPurgeAndArchive job was failing continuously.  When I opened the history for the job, I found following error logged in the history.

"Current Error: ABC\BiztalkService. Unspecified error occurred on SQL Server.
Connection may have been terminated by the server. [SQLSTATE HY000] (Error 0)
Unable to find index entry in index ID 1, of table xyz, in database 'BizTalkDTADb'.
The indicated index is corrupt or there is a problem with the current update plan. Run DBCC CHECKDB or DBCC CHECKTABLE.
If the problem persists, contact product support. [SQLSTATE HY000] (Error 8646). The step failed."

The event log gave me following

SQL Server Scheduled Job 'DTA Purge and Archive (BizTalkDTADb)' (0x1587D5C96B85D342BEC4D90B14B1EFA1) - Status: Failed - Invoked on: 2018-08-21 09:49:07 - Message: The job failed.  The Job was invoked by Schedule 12 (Archive and Purge Schedule).  The last step to run was step 1 (Archive and Purge).

Unable to find index entry in index ID 1, of table 965578478, in database 'BizTalkDTADb'. The indicated index is corrupt or there is a problem with the current update plan. Run DBCC CHECKDB or DBCC CHECKTABLE. If the problem persists, contact product support.
<h1>Observations and Solutions</h1>
I wanted to run the DBCC CHECKDB command on the DTA database. So I decided to take my environment down by
<ol>
	<li>Stopping IIS</li>
	<li>Stopping the applications</li>
	<li>Stopping the Host Instances</li>
	<li>ENT SSO service</li>
	<li>SQL Agent jobs</li>
</ol>
Once done, I ran the DBCC CHECKDB on the BizTalkDTADb to check for the integrity issues. Following are the two interesting error snippets from the DBCC CHECKDB.
<table>
<tbody>
<tr>
<td> DBCC results for 'Tracking_Parts1'.

Msg 8914, Level 16, State 1, Line 1

Incorrect PFS free space information for page (1:18447) in object ID 18099105, index ID 1, partition ID 72057594046185472, alloc unit ID 72057594047889408 (type LOB data). Expected value  95_PCT_FULL, actual value 100_PCT_FULL.

Msg 8965, Level 16, State 1, Line 1

Table error: Object ID 18099105, index ID 1, partition ID 72057594046185472, alloc unit ID 72057594047889408 (type LOB data). The off-row data node at page (1:18447), slot 13, text ID 45249724416 is referenced by page (1:9222), slot 32, but was not seen in the scan.

Msg 8965, Level 16, State 1, Line 1

Table error: Object ID 18099105, index ID 1, partition ID 72057594046185472, alloc unit ID 72057594047889408 (type LOB data). The off-row data node at page (1:18447), slot 15, text ID 45249789952 is referenced by page (1:8160), slot 33, but was not seen in the scan.

Msg 8929, Level 16, State 1, Line 1

Object ID 18099105, index ID 1, partition ID 72057594046185472, alloc unit ID 72057594048741376 (type In-row data): Errors found in off-row data with ID 45249789952 owned by data record identified by RID = (1:8160:33)

Msg 8929, Level 16, State 1, Line 1

Object ID 18099105, index ID 1, partition ID 72057594046185472, alloc unit ID 72057594048741376 (type In-row data): Errors found in off-row data with ID 45249724416 owned by data record identified by RID = (1:9222:32)</td>
</tr>
<tr>
<td><span style="font-size:12.1px;">DBCC results for 'dta_MessageInOutEvents'.</span>

<span style="font-size:12.1px;">Msg 8952, Level 16, State 1, Line 1</span>

<span style="font-size:12.1px;">Table error: table 'dta_MessageInOutEvents' (ID 965578478). Index row in index 'dta_MessageInOutEvents_index_ActivityId' (ID 2) does not match any data row. Possible extra or invalid keys for:</span>

<span style="font-size:12.1px;">Msg 8956, Level 16, State 1, Line 1</span>

<span style="font-size:12.1px;">Index row (1:7055:13) with values (uidServiceInstanceId = '4DA4DB68-178B-404E-BAFE-EC322255E809' and uidActivityId = '4DA4DB68-178B-404E-BAFE-EC322255E809' and uidMessageInstanceId = '581155AC-3471-4FF7-A1BC-BF640D840B62' and nEventId = 8087) pointing to the data row identified by (uidMessageInstanceId = '581155AC-3471-4FF7-A1BC-BF640D840B62' and nEventId = 8087).</span>

<span style="font-size:12.1px;">Msg 8952, Level 16, State 1, Line 1</span>

<span style="font-size:12.1px;">Table error: table 'dta_MessageInOutEvents' (ID 965578478). Index row in index 'dta_MessageInOutEvents_index_ActivityId' (ID 2) does not match any data row. Possible extra or invalid keys for:</span>

<span style="font-size:12.1px;">Msg 8956, Level 16, State 1, Line 1</span>

<span style="font-size:12.1px;">Index row (1:22581:37) with values (uidServiceInstanceId = 'C551885D-99B4-4B05-B056-70CCBE652EB6' and uidActivityId = 'C551885D-99B4-4B05-B056-70CCBE652EB6' and uidMessageInstanceId = '581155AC-3471-4FF7-A1BC-BF640D840B62' and nEventId = 8086) pointing to the data row identified by (uidMessageInstanceId = '581155AC-3471-4FF7-A1BC-BF640D840B62' and nEventId = 8086).</span>

<span style="font-size:12.1px;">Msg 8952, Level 16, State 1, Line 1</span>

<span style="font-size:12.1px;">Table error: table 'dta_MessageInOutEvents' (ID 965578478). Index row in index 'dta_MessageInOutEvents_index_dtTimestamp' (ID 3) does not match any data row. Possible extra or invalid keys for:</span>

<span style="font-size:12.1px;">Msg 8956, Level 16, State 1, Line 1</span>

<span style="font-size:12.1px;">Index row (1:39912:83) with values (dtTimestamp = '2018-01-15 08:52:32.137' and dtInsertionTimeStamp = '2018-01-15 08:52:33.413' and nEventId = 8086 and uidMessageInstanceId = '581155AC-3471-4FF7-A1BC-BF640D840B62') pointing to the data row identified by (nEventId = 8086 and uidMessageInstanceId = '581155AC-3471-4FF7-A1BC-BF640D840B62').</span>

<span style="font-size:12.1px;">Msg 8952, Level 16, State 1, Line 1</span>

<span style="font-size:12.1px;">Table error: table 'dta_MessageInOutEvents' (ID 965578478). Index row in index 'dta_MessageInOutEvents_index_dtTimestamp' (ID 3) does not match any data row. Possible extra or invalid keys for:</span>

<span style="font-size:12.1px;">Msg 8956, Level 16, State 1, Line 1</span>

<span style="font-size:12.1px;">Index row (1:39912:84) with values (dtTimestamp = '2018-01-15 08:52:32.357' and dtInsertionTimeStamp = '2018-01-15 08:52:33.470' and nEventId = 8087 and uidMessageInstanceId = '581155AC-3471-4FF7-A1BC-BF640D840B62') pointing to the data row identified by (nEventId = 8087 and uidMessageInstanceId = '581155AC-3471-4FF7-A1BC-BF640D840B62').</span>

<span style="font-size:12.1px;">Msg 8952, Level 16, State 1, Line 1</span>

<span style="font-size:12.1px;">Table error: table 'dta_MessageInOutEvents' (ID 965578478). Index row in index 'dta_MessageInOutEvents_index_nSchemaId' (ID 4) does not match any data row. Possible extra or invalid keys for:</span>

<span style="font-size:12.1px;">Msg 8956, Level 16, State 1, Line 1</span>

<span style="font-size:12.1px;">Index row (1:17813:72) with values (nSchemaId = -992533544 and uidMessageInstanceId = '581155AC-3471-4FF7-A1BC-BF640D840B62' and nEventId = 8086) pointing to the data row identified by (uidMessageInstanceId = '581155AC-3471-4FF7-A1BC-BF640D840B62' and nEventId = 8086).</span>

<span style="font-size:12.1px;">Msg 8952, Level 16, State 1, Line 1</span>

<span style="font-size:12.1px;">Table error: table 'dta_MessageInOutEvents' (ID 965578478). Index row in index 'dta_MessageInOutEvents_index_nSchemaId' (ID 4) does not match any data row. Possible extra or invalid keys for:</span>

<span style="font-size:12.1px;">Msg 8956, Level 16, State 1, Line 1</span>

<span style="font-size:12.1px;">Index row (1:17813:73) with values (nSchemaId = -992533544 and uidMessageInstanceId = '581155AC-3471-4FF7-A1BC-BF640D840B62' and nEventId = 8087) pointing to the data row identified by (uidMessageInstanceId = '581155AC-3471-4FF7-A1BC-BF640D840B62' and nEventId = 8087).</span></td>
</tr>
</tbody>
</table>
As evident from the message the next job that I did was try to rebuild the index of the BizTalkDTADb database. As already had an working copy of the BizTalkDTADb backed up using my BackUpBizTalk server job.

So I ran the stored procedure <strong>dtasp_RebuildIndexes</strong> on the BizTalkDTADb databse. I enabled the SQL server Agent Service and waited for the DTAPurgeANdArchivejob to run as scheduled, but ALAS, it failed with the same error as above, DBCC CHECKDB gave above result.

Next I tried the commands

<strong> ALTER INDEX ALL ON [dbo].[Tracking_Parts1] REBUILD</strong> and

<strong>ALTER INDEX ALL ON [dbo].[MessageInOutEvents] REBUILD</strong>

to rebuild the indexes on the faulty tables highlighted by the DBCC CHECKDB command, then again ran the DTAPurgeAndArchive job, but it gave the same error.

<strong>What to do next??</strong>

Next I thought of the terminator tool which helps in resolution of most of the BizTalk issues. I ran "<strong>PURGE dta_DebugTrace and dta_MessageInOutEvents</strong>" as shown in following screen shots

<img class=" size-full wp-image-99 alignleft" src="https://theserverlessspirit.files.wordpress.com/2018/08/deletedebugtraceamd-dtamessageinoutevents.jpg" alt="DeleteDebugTraceamd dtaMessageInOutEvents" width="1931" height="516" />

Once Done, I ran the DBCC CHECKDB command and it only listed following errors
<table>
<tbody>
<tr>
<td>DBCC results for 'Tracking_Parts1'.

Msg 8914, Level 16, State 1, Line 1

Incorrect PFS free space information for page (1:18447) in object ID 18099105, index ID 1, partition ID 72057594046185472, alloc unit ID 72057594047889408 (type LOB data). Expected value  95_PCT_FULL, actual value 100_PCT_FULL.

Msg 8965, Level 16, State 1, Line 1

Table error: Object ID 18099105, index ID 1, partition ID 72057594046185472, alloc unit ID 72057594047889408 (type LOB data). The off-row data node at page (1:18447), slot 13, text ID 45249724416 is referenced by page (1:9222), slot 32, but was not seen in the scan.

Msg 8965, Level 16, State 1, Line 1

Table error: Object ID 18099105, index ID 1, partition ID 72057594046185472, alloc unit ID 72057594047889408 (type LOB data). The off-row data node at page (1:18447), slot 15, text ID 45249789952 is referenced by page (1:8160), slot 33, but was not seen in the scan.

Msg 8929, Level 16, State 1, Line 1

Object ID 18099105, index ID 1, partition ID 72057594046185472, alloc unit ID 72057594048741376 (type In-row data): Errors found in off-row data with ID 45249789952 owned by data record identified by RID = (1:8160:33)

Msg 8929, Level 16, State 1, Line 1

Object ID 18099105, index ID 1, partition ID 72057594046185472, alloc unit ID 72057594048741376 (type In-row data): Errors found in off-row data with ID 45249724416 owned by data record identified by RID = (1:9222:32)
<div></div>
&nbsp;</td>
</tr>
</tbody>
</table>
I was still unable to overcome the issues with the TRacking_Parts1 table errors, so I had only two options
<ol>
	<li>Delete the data in the tracking database</li>
	<li>Restore and old copy of the database</li>
</ol>
The first option was the easiest way out, nuke all the data in the tracking database and it is as good as new. So I used the BHM Maintenance feature to nuke  the Tracking Database.

<img class="alignnone size-full wp-image-100" src="https://theserverlessspirit.files.wordpress.com/2018/08/purgedtadata.jpg" alt="purgeDTAData" width="1920" height="792" />

When I ran the DTAPurgeAndArchive job, it ran without any hassle. But this is not the correct way to get the job to work .

Another option that remains is to take a complete down time and restore the old backup of the tracking database and the job runs again marvelously.

These were the two approaches I could think of if anyone reading this blog knows another work around for the issue, I would love to hear it in the comments  section.

&nbsp;