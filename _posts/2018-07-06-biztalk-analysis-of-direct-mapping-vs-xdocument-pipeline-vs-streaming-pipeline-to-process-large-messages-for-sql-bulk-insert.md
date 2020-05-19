---
ID: 33
post_title: 'BizTalk : Analysis of Direct Mapping vs XDocument Pipeline vs Streaming Pipeline To Process Large Messages for SQL Bulk Insert'
author: Mandar Dharmadhikari
post_excerpt: ""
layout: post
permalink: >
  https://theabodeofcode.com/biztalk-analysis-of-direct-mapping-vs-xdocument-pipeline-vs-streaming-pipeline-to-process-large-messages-for-sql-bulk-insert/
published: true
post_date: 2018-07-06 07:08:16
---
<div dir="ltr" style="text-align:left;"><a href="https://www.blogger.com/null" name="Top"></a>
<h1>Introduction</h1>
Sometimes while developing BizTalk applications, there are requirements where BizTalk developers have to write custom code that will help them achieve the functionality they want. e.g, functionality to process data from excel sheet or to process pdf documents . While in some requirements it becomes necessary that BizTalk developers write custom components which help them, make the BizTalk application perform in optimal condition. e.g. while dealing with large messages that are to be processed by BizTalk. In such case if the application is not designed properly then there are chances that the solution will put lot of strain on the BizTalk environment which can in turn cause the degradation of performance for  other applications. This article aims to discuss one such scenario where BizTalk handles large messages and what BizTalk developers do to handle this load better by writing better code.

<hr />

<h1>Fictional Scenario</h1>
EmployeeManager is a application which is used by the OnBorading team to onboard the employees in to the company ABC Pvtr ltd. EmployeeManager uses the BizTalk as a middle ware to communicate with multiple systems in organization. One such process in the EmployeeManager is where they upload the Employees On boarded in bulk . Another process is uploading the necessary on boarding documents of the employee to the data store. In such cases EmployeeManager relies on BizTalk to communicate with the SQL data store where the System stores documents from the employees.


As there can be multiple employees that can be on boarded in a day, BizTalk needs an efficient solution to manage the large messages. In this article the example of saving the Employee Information like user ID, Location and their avatar(profile picture) to the database is discussed. Each call to BizTalk can contain multiple employee profiles to be inserted/updated in the datastore. The messages that BizTalk receives can go up to 30 to 40 MB and at times even more than that. This BizTalk application is hosted on a BizTalk Group where there are other applications which also process high loads of messages. Hence it becomes of paramount importance to have a graceful code which is efficient itself not to cause issues for other applications.

<hr />

<h1>Conventional Approach</h1>
The basic approach to achieve this is to use the BizTalk development approach as is. Meaning, The BizTalk exposes a schema to accept the details like the userId, Location and Avatar and then map them to the schema generated for inserting the data into the SQL store. This approach has apparent disadvantages. Since the BizTalk publishes messages to the message box and then the subscriber pricks it up, in such case the entire message which can be up to 30 to 40 MB is published in the message box and then it is picked up by the orchestration which then works on the business logic and again publishes the transformed message for the SQL port to pick up for the insertion. In such case the large message gets published twice in the message box ( once when the BizTalk Receive location acts as an publisher and a second time when Orchestration acts as a Publisher). If the number of calls are more for such large messages, it is bound to produce issues down the line for other applications.

<hr />

<h1>Conventional Solution</h1>
The conventional solution that can be thought of to this Direct Publish-Map -Publish scenario is
<ol>
	<li>Identify the parts of the message which are big in size (in this case the avatars of the employees)</li>
	<li>Create a temporary data store on the  File System of BizTalk application server</li>
	<li>On the receive location, use a custom pipeline component which removes the large data from the incoming message and save it as a temporary file in the data store.</li>
	<li>Replace the large chunk in the message with the path to the file store in the file store.</li>
	<li>Publish the message to the message box ( Size of the message is substantially reduced here as the large parts are stripped and replaced by the path of file)</li>
	<li>Let the orchestration subscribe and do the business logic and direct mapping into the SQL insert message.</li>
	<li>On the SQL send port, create a custom pipeline component and in the last stage of pipeline(encode stage) , replace the path to the file with the actual data after reading the file from data store.</li>
	<li>Delete the file from the store and send the message to sql for insert.</li>
</ol>
The advantage the solution gives is that it removes the large data from the message and stores it if to file system temporarily. Thus this ensures that the message sent to the "Actual BizTalk To process" has a very small size and can ensure efficient processing.

<hr />

<h1>Problem With The Conventional Solution?</h1>
The conventional solution discussed above is the correct way to go about designing the solution. But there can be problems if this is not implemented properly. Most of the time the developers use the XDocument class to do the work for them. They just load the message incoming into the pipeline and then perform the necessary operations on them. While this a perfectly valid way of doing the stuff, the problem lies with the way XDocument makes the changes to the message. The XDocument class loads the entire BizTalk message in to the process memory and then does the necessary work. The disadvantage with this process is that the XDocument creates a memory foot print which is way larger than the actual size of the message coming into the BizTalk. This defeats the purpose of implementation of the conventional solution.
<h2>How To Overcome This Problem?</h2>
BizTalk server product provides an Microsoft.BizTalk.Streaming.dll assembly which has various streaming classes present in it which can be used in lieu of the classical XDocument class. These classes are inherited from the System.IO classes and they write the message components to the file instead of loading the message components into the memory as done by the XDocument class. Thus using the classes in Streaming namespace will give the added benefit of having a code which goes easier on the in process memory. The only problem is the lack of documentation and knowledge about the classes in the streaming classes. This article will try to help the reader understand few classes of the Streaming namespace.

<hr />

<h1>Implementation</h1>
The implementation for the demo for this article can be divided into following categories.
<ol>
	<li>Design SQL Tables and Stored Procedure to Save Employee Data</li>
	<li>Design the BizTalk Solution to Do a Bul Insert of Employee Profiles to SQL Table</li>
	<li>Create the Custom Receive Pipeline Component Using XDocument <span style="font-size:12.1px;">To Load Large Message</span></li>
	<li>Create the Custom Send Pipeline Component Using XDocument To Load Large Message</li>
	<li>Create the Custom Receive Pipeline Component Using BizTalk Streaming Classes To Load Large Message</li>
	<li>Create the Custom Send Pipeline Component Using BizTalk Streaming Classes To Load Large Message</li>
</ol>
<h2>SQL Design</h2>
Following code can be used to create the table that stores the employee profile details.
<div class="reCodeBlock" style="border:1px solid #7f9db9;overflow-y:auto;">
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:black;">USE [POCDatabase]</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">GO</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">/****** Object:  </code><code style="color:#006699;font-weight:bold;">Table</code> <code style="color:black;">[dbo].[Employee_Profile]    Script </code><code style="color:#006699;font-weight:bold;">Date</code><code style="color:black;">: 2/24/2018 7:49:24 PM ******/</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">SET</code> <code style="color:black;">ANSI_NULLS </code><code style="color:#006699;font-weight:bold;">ON</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">GO</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">SET</code> <code style="color:black;">QUOTED_IDENTIFIER </code><code style="color:#006699;font-weight:bold;">ON</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:black;">GO</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">CREATE</code> <code style="color:#006699;font-weight:bold;">TABLE</code> <code style="color:black;">[dbo].[Employee_Profile](</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[UserId] [</code><code style="color:#006699;font-weight:bold;">int</code><code style="color:black;">] </code><code style="color:grey;">NOT</code> <code style="color:grey;">NULL</code><code style="color:black;">,</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[LOCATION] [</code><code style="color:#006699;font-weight:bold;">varchar</code><code style="color:black;">](10) </code><code style="color:grey;">NOT</code> <code style="color:grey;">NULL</code><code style="color:black;">,</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[AVATAR] [image] </code><code style="color:grey;">NOT</code> <code style="color:grey;">NULL</code><code style="color:black;">,</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">UNIQUE</code> <code style="color:black;">NONCLUSTERED </code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">(</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[UserId] </code><code style="color:#006699;font-weight:bold;">ASC</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">)</code><code style="color:#006699;font-weight:bold;">WITH</code> <code style="color:black;">(PAD_INDEX = </code><code style="color:#006699;font-weight:bold;">OFF</code><code style="color:black;">, STATISTICS_NORECOMPUTE = </code><code style="color:#006699;font-weight:bold;">OFF</code><code style="color:black;">, IGNORE_DUP_KEY = </code><code style="color:#006699;font-weight:bold;">OFF</code><code style="color:black;">, ALLOW_ROW_LOCKS = </code><code style="color:#006699;font-weight:bold;">ON</code><code style="color:black;">, ALLOW_PAGE_LOCKS = </code><code style="color:#006699;font-weight:bold;">ON</code><code style="color:black;">) </code><code style="color:#006699;font-weight:bold;">ON</code> <code style="color:black;">[</code><code style="color:#006699;font-weight:bold;">PRIMARY</code><code style="color:black;">]</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:black;">) </code><code style="color:#006699;font-weight:bold;">ON</code> <code style="color:black;">[</code><code style="color:#006699;font-weight:bold;">PRIMARY</code><code style="color:black;">] TEXTIMAGE_ON [</code><code style="color:#006699;font-weight:bold;">PRIMARY</code><code style="color:black;">]</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">GO</code></span></div>
</div>
Following is the Stored Procedure that is used to insert the data into the table created above.
<div class="reCodeBlock" style="border:1px solid #7f9db9;overflow-y:auto;">
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:black;">USE [POCDatabase]</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">GO</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">/****** Object:  StoredProcedure [dbo].[usp_OnboardEmployee]    Script </code><code style="color:#006699;font-weight:bold;">Date</code><code style="color:black;">: 2/24/2018 7:55:03 PM ******/</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">SET</code> <code style="color:black;">ANSI_NULLS </code><code style="color:#006699;font-weight:bold;">ON</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">GO</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">SET</code> <code style="color:black;">QUOTED_IDENTIFIER </code><code style="color:#006699;font-weight:bold;">ON</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:black;">GO</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">CREATE</code> <code style="color:#006699;font-weight:bold;">PROCEDURE</code> <code style="color:black;">[dbo].[usp_OnboardEmployee]</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">@employeeId </code><code style="color:#006699;font-weight:bold;">INT</code><code style="color:black;">, @employeeName </code><code style="color:#006699;font-weight:bold;">VARCHAR</code> <code style="color:black;">(100), @salary </code><code style="color:#006699;font-weight:bold;">DECIMAL</code> <code style="color:black;">(10, 2), @managerid </code><code style="color:#006699;font-weight:bold;">INT</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">AS</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">BEGIN</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:#006699;font-weight:bold;">INSERT</code>  <code style="color:#006699;font-weight:bold;">INTO</code> <code style="color:black;">[EMPLOYEE_MASTER]</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:#006699;font-weight:bold;">VALUES</code> <code style="color:black;">(@employeeId, @employeeName, @salary, @managerid);</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">END</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:black;">GO</code></span></div>
</div>
<h2>BizTalk Design</h2>
Following is the schema that is used for the BizTalk service ( BizTalk schemas exposed as WCF Service)
<div class="reCodeBlock" style="border:1px solid #7f9db9;overflow-y:auto;">
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:black;">&lt;?</code><code style="color:#006699;font-weight:bold;">xml</code> <code style="color:grey;">version</code><code style="color:black;">=</code><code style="color:blue;">"1.0"</code> <code style="color:grey;">encoding</code><code style="color:black;">=</code><code style="color:blue;">"utf-16"</code><code style="color:black;">?&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:schema</code> <code style="color:grey;">xmlns</code><code style="color:black;">=</code><code style="color:blue;">"<a href="http://processlargefilesinbiztalk.biztalkservicetypes/">http://ProcessLargeFilesInBizTalk.BizTalkServiceTypes</a>"</code> <code style="color:grey;">xmlns:b</code><code style="color:black;">=</code><code style="color:blue;">"<a href="http://schemas.microsoft.com/BizTalk/2003">http://schemas.microsoft.com/BizTalk/2003</a>"</code> <code style="color:grey;">targetNamespace</code><code style="color:black;">=</code><code style="color:blue;">"<a href="http://processlargefilesinbiztalk.biztalkservicetypes/">http://ProcessLargeFilesInBizTalk.BizTalkServiceTypes</a>"</code> <code style="color:grey;">xmlns:xs</code><code style="color:black;">=</code><code style="color:blue;">"<a href="http://www.w3.org/2001/XMLSchema">http://www.w3.org/2001/XMLSchema</a>"</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>  </code><span style="margin-left:6px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:element</code> <code style="color:grey;">name</code><code style="color:black;">=</code><code style="color:blue;">"Request"</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:complexType</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>      </code><span style="margin-left:18px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:sequence</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:element</code> <code style="color:grey;">minOccurs</code><code style="color:black;">=</code><code style="color:blue;">"1"</code> <code style="color:grey;">maxOccurs</code><code style="color:black;">=</code><code style="color:blue;">"unbounded"</code> <code style="color:grey;">name</code><code style="color:black;">=</code><code style="color:blue;">"Profile"</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>          </code><span style="margin-left:30px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:complexType</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:sequence</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>              </code><span style="margin-left:42px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:element</code> <code style="color:grey;">name</code><code style="color:black;">=</code><code style="color:blue;">"UserId"</code> <code style="color:grey;">type</code><code style="color:black;">=</code><code style="color:blue;">"xs:string"</code> <code style="color:black;">/&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>              </code><span style="margin-left:42px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:element</code> <code style="color:grey;">name</code><code style="color:black;">=</code><code style="color:blue;">"Location"</code> <code style="color:grey;">type</code><code style="color:black;">=</code><code style="color:blue;">"xs:string"</code> <code style="color:black;">/&gt;</code></span></div>
<div style="background-color:white;"><code>              </code><span style="margin-left:42px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:element</code> <code style="color:grey;">name</code><code style="color:black;">=</code><code style="color:blue;">"Avatar"</code> <code style="color:grey;">type</code><code style="color:black;">=</code><code style="color:blue;">"xs:base64Binary"</code> <code style="color:black;">/&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:sequence</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>          </code><span style="margin-left:30px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:complexType</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:element</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>      </code><span style="margin-left:18px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:sequence</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:complexType</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>  </code><span style="margin-left:6px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:element</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>  </code><span style="margin-left:6px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:element</code> <code style="color:grey;">name</code><code style="color:black;">=</code><code style="color:blue;">"Response"</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:complexType</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>      </code><span style="margin-left:18px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:sequence</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:element</code> <code style="color:grey;">minOccurs</code><code style="color:black;">=</code><code style="color:blue;">"1"</code> <code style="color:grey;">maxOccurs</code><code style="color:black;">=</code><code style="color:blue;">"unbounded"</code> <code style="color:grey;">name</code><code style="color:black;">=</code><code style="color:blue;">"Profile"</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>          </code><span style="margin-left:30px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:complexType</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:sequence</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>              </code><span style="margin-left:42px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:element</code> <code style="color:grey;">name</code><code style="color:black;">=</code><code style="color:blue;">"UserId"</code> <code style="color:grey;">type</code><code style="color:black;">=</code><code style="color:blue;">"xs:string"</code> <code style="color:black;">/&gt;</code></span></div>
<div style="background-color:white;"><code>              </code><span style="margin-left:42px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:element</code> <code style="color:grey;">name</code><code style="color:black;">=</code><code style="color:blue;">"TransactionId"</code> <code style="color:grey;">type</code><code style="color:black;">=</code><code style="color:blue;">"xs:string"</code> <code style="color:black;">/&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:sequence</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>          </code><span style="margin-left:30px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:complexType</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:element</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>      </code><span style="margin-left:18px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:sequence</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:complexType</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>  </code><span style="margin-left:6px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:element</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:schema</code><code style="color:black;">&gt;</code></span></div>
</div>
The schemas generated after consuming the SQL stored procedure are as follows ( Refer See Also Section to understand how to consume the Stored Procedure in Visual Studio BizTalk Application)
<div class="reCodeBlock" style="border:1px solid #7f9db9;overflow-y:auto;">
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:black;">&lt;?</code><code style="color:#006699;font-weight:bold;">xml</code> <code style="color:grey;">version</code><code style="color:black;">=</code><code style="color:blue;">"1.0"</code> <code style="color:grey;">encoding</code><code style="color:black;">=</code><code style="color:blue;">"utf-8"</code><code style="color:black;">?&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:schema</code> <code style="color:grey;">xmlns:tns</code><code style="color:black;">=</code><code style="color:blue;">"<a href="http://schemas.microsoft.com/Sql/2008/05/TypedProcedures/dbo">http://schemas.microsoft.com/Sql/2008/05/TypedProcedures/dbo</a>"</code> <code style="color:grey;">elementFormDefault</code><code style="color:black;">=</code><code style="color:blue;">"qualified"</code> <code style="color:grey;">targetNamespace</code><code style="color:black;">=</code><code style="color:blue;">"<a href="http://schemas.microsoft.com/Sql/2008/05/TypedProcedures/dbo">http://schemas.microsoft.com/Sql/2008/05/TypedProcedures/dbo</a>"</code> <code style="color:grey;">version</code><code style="color:black;">=</code><code style="color:blue;">"1.0"</code> <code style="color:grey;">xmlns:xs</code><code style="color:black;">=</code><code style="color:blue;">"<a href="http://www.w3.org/2001/XMLSchema">http://www.w3.org/2001/XMLSchema</a>"</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>  </code><span style="margin-left:6px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:annotation</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:appinfo</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>      </code><span style="margin-left:18px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">fileNameHint</code> <code style="color:grey;">xmlns</code><code style="color:black;">=</code><code style="color:blue;">"<a href="http://schemas.microsoft.com/servicemodel/adapters/metadata/xsd">http://schemas.microsoft.com/servicemodel/adapters/metadata/xsd</a>"</code><code style="color:black;">&gt;TypedProcedure.dbo&lt;/</code><code style="color:#006699;font-weight:bold;">fileNameHint</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:appinfo</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>  </code><span style="margin-left:6px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:annotation</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>  </code><span style="margin-left:6px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:element</code> <code style="color:grey;">name</code><code style="color:black;">=</code><code style="color:blue;">"usp_UpdateEmployeeProfile"</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:annotation</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>      </code><span style="margin-left:18px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:documentation</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">doc:action</code> <code style="color:grey;">xmlns:doc</code><code style="color:black;">=</code><code style="color:blue;">"<a href="http://schemas.microsoft.com/servicemodel/adapters/metadata/documentation">http://schemas.microsoft.com/servicemodel/adapters/metadata/documentation</a>"</code><code style="color:black;">&gt;TypedProcedure/dbo/usp_UpdateEmployeeProfile&lt;/</code><code style="color:#006699;font-weight:bold;">doc:action</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>      </code><span style="margin-left:18px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:documentation</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:annotation</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:complexType</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>      </code><span style="margin-left:18px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:sequence</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:element</code> <code style="color:grey;">minOccurs</code><code style="color:black;">=</code><code style="color:blue;">"0"</code> <code style="color:grey;">maxOccurs</code><code style="color:black;">=</code><code style="color:blue;">"1"</code> <code style="color:grey;">name</code><code style="color:black;">=</code><code style="color:blue;">"USERId"</code> <code style="color:grey;">nillable</code><code style="color:black;">=</code><code style="color:blue;">"true"</code> <code style="color:grey;">type</code><code style="color:black;">=</code><code style="color:blue;">"xs:int"</code> <code style="color:black;">/&gt;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:element</code> <code style="color:grey;">minOccurs</code><code style="color:black;">=</code><code style="color:blue;">"0"</code> <code style="color:grey;">maxOccurs</code><code style="color:black;">=</code><code style="color:blue;">"1"</code> <code style="color:grey;">name</code><code style="color:black;">=</code><code style="color:blue;">"location"</code> <code style="color:grey;">nillable</code><code style="color:black;">=</code><code style="color:blue;">"true"</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>          </code><span style="margin-left:30px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:simpleType</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:restriction</code> <code style="color:grey;">base</code><code style="color:black;">=</code><code style="color:blue;">"xs:string"</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>              </code><span style="margin-left:42px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:maxLength</code> <code style="color:grey;">value</code><code style="color:black;">=</code><code style="color:blue;">"10"</code> <code style="color:black;">/&gt;</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:restriction</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>          </code><span style="margin-left:30px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:simpleType</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:element</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:element</code> <code style="color:grey;">minOccurs</code><code style="color:black;">=</code><code style="color:blue;">"0"</code> <code style="color:grey;">maxOccurs</code><code style="color:black;">=</code><code style="color:blue;">"1"</code> <code style="color:grey;">name</code><code style="color:black;">=</code><code style="color:blue;">"avatar"</code> <code style="color:grey;">nillable</code><code style="color:black;">=</code><code style="color:blue;">"true"</code> <code style="color:grey;">type</code><code style="color:black;">=</code><code style="color:blue;">"xs:base64Binary"</code> <code style="color:black;">/&gt;</code></span></div>
<div style="background-color:white;"><code>      </code><span style="margin-left:18px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:sequence</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:complexType</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>  </code><span style="margin-left:6px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:element</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>  </code><span style="margin-left:6px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:element</code> <code style="color:grey;">name</code><code style="color:black;">=</code><code style="color:blue;">"usp_UpdateEmployeeProfileResponse"</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:annotation</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>      </code><span style="margin-left:18px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:documentation</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">doc:action</code> <code style="color:grey;">xmlns:doc</code><code style="color:black;">=</code><code style="color:blue;">"<a href="http://schemas.microsoft.com/servicemodel/adapters/metadata/documentation">http://schemas.microsoft.com/servicemodel/adapters/metadata/documentation</a>"</code><code style="color:black;">&gt;TypedProcedure/dbo/usp_UpdateEmployeeProfile/response&lt;/</code><code style="color:#006699;font-weight:bold;">doc:action</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>      </code><span style="margin-left:18px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:documentation</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:annotation</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:complexType</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>      </code><span style="margin-left:18px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:sequence</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:element</code> <code style="color:grey;">minOccurs</code><code style="color:black;">=</code><code style="color:blue;">"1"</code> <code style="color:grey;">maxOccurs</code><code style="color:black;">=</code><code style="color:blue;">"1"</code> <code style="color:grey;">name</code><code style="color:black;">=</code><code style="color:blue;">"ReturnValue"</code> <code style="color:grey;">type</code><code style="color:black;">=</code><code style="color:blue;">"xs:int"</code> <code style="color:black;">/&gt;</code></span></div>
<div style="background-color:white;"><code>      </code><span style="margin-left:18px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:sequence</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:complexType</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>  </code><span style="margin-left:6px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:element</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:schema</code><code style="color:black;">&gt;</code></span></div>
</div>
In Odrer To do a Bulk Insert using the Composite Operation facility of the WCf-Custom/WCF-SQL adapter, following custom schema is created and the request/response types created above are imported as child records in the schema. The Min and max occurs properties are set to ) and unbounded respectively. Following is the schema created for the bulk insert.
<div class="reCodeBlock" style="border:1px solid #7f9db9;overflow-y:auto;">
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:black;">&lt;?</code><code style="color:#006699;font-weight:bold;">xml</code> <code style="color:grey;">version</code><code style="color:black;">=</code><code style="color:blue;">"1.0"</code> <code style="color:grey;">encoding</code><code style="color:black;">=</code><code style="color:blue;">"utf-16"</code><code style="color:black;">?&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:schema</code> <code style="color:grey;">xmlns</code><code style="color:black;">=</code><code style="color:blue;">"<a href="http://processlargefilesinbiztalk.sqlschemas.bulkinserttypes/">http://ProcessLargeFilesInBizTalk.SQLSchemas.BulkInsertTypes</a>"</code> <code style="color:grey;">xmlns:b</code><code style="color:black;">=</code><code style="color:blue;">"<a href="http://schemas.microsoft.com/BizTalk/2003">http://schemas.microsoft.com/BizTalk/2003</a>"</code> <code style="color:grey;">xmlns:ns0</code><code style="color:black;">=</code><code style="color:blue;">"<a href="http://schemas.microsoft.com/Sql/2008/05/TypedProcedures/dbo">http://schemas.microsoft.com/Sql/2008/05/TypedProcedures/dbo</a>"</code> <code style="color:grey;">targetNamespace</code><code style="color:black;">=</code><code style="color:blue;">"<a href="http://processlargefilesinbiztalk.sqlschemas.bulkinserttypes/">http://ProcessLargeFilesInBizTalk.SQLSchemas.BulkInsertTypes</a>"</code> <code style="color:grey;">xmlns:xs</code><code style="color:black;">=</code><code style="color:blue;">"<a href="http://www.w3.org/2001/XMLSchema">http://www.w3.org/2001/XMLSchema</a>"</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>  </code><span style="margin-left:6px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:import</code> <code style="color:grey;">schemaLocation</code><code style="color:black;">=</code><code style="color:blue;">".\TypedProcedure.dbo.xsd"</code> <code style="color:grey;">namespace</code><code style="color:black;">=</code><code style="color:blue;">"<a href="http://schemas.microsoft.com/Sql/2008/05/TypedProcedures/dbo">http://schemas.microsoft.com/Sql/2008/05/TypedProcedures/dbo</a>"</code> <code style="color:black;">/&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>  </code><span style="margin-left:6px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:annotation</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:appinfo</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>      </code><span style="margin-left:18px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">b:references</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">b:reference</code> <code style="color:grey;">targetNamespace</code><code style="color:black;">=</code><code style="color:blue;">"<a href="http://schemas.microsoft.com/Sql/2008/05/TypedProcedures/dbo">http://schemas.microsoft.com/Sql/2008/05/TypedProcedures/dbo</a>"</code> <code style="color:black;">/&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>      </code><span style="margin-left:18px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">b:references</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:appinfo</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>  </code><span style="margin-left:6px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:annotation</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>  </code><span style="margin-left:6px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:element</code> <code style="color:grey;">name</code><code style="color:black;">=</code><code style="color:blue;">"UpdateEmployeeProfiles"</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:complexType</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>      </code><span style="margin-left:18px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:sequence</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:element</code> <code style="color:grey;">minOccurs</code><code style="color:black;">=</code><code style="color:blue;">"1"</code> <code style="color:grey;">maxOccurs</code><code style="color:black;">=</code><code style="color:blue;">"unbounded"</code> <code style="color:grey;">ref</code><code style="color:black;">=</code><code style="color:blue;">"ns0:usp_UpdateEmployeeProfile"</code> <code style="color:black;">/&gt;</code></span></div>
<div style="background-color:white;"><code>      </code><span style="margin-left:18px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:sequence</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:complexType</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>  </code><span style="margin-left:6px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:element</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>  </code><span style="margin-left:6px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:element</code> <code style="color:grey;">name</code><code style="color:black;">=</code><code style="color:blue;">"UpdateEmployeeProfilesResponse"</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:complexType</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>      </code><span style="margin-left:18px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:sequence</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">&lt;</code><code style="color:#006699;font-weight:bold;">xs:element</code> <code style="color:grey;">minOccurs</code><code style="color:black;">=</code><code style="color:blue;">"1"</code> <code style="color:grey;">maxOccurs</code><code style="color:black;">=</code><code style="color:blue;">"unbounded"</code> <code style="color:grey;">ref</code><code style="color:black;">=</code><code style="color:blue;">"ns0:usp_UpdateEmployeeProfileResponse"</code> <code style="color:black;">/&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>      </code><span style="margin-left:18px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:sequence</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:complexType</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>  </code><span style="margin-left:6px !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:element</code><code style="color:black;">&gt;</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:black;">&lt;/</code><code style="color:#006699;font-weight:bold;">xs:schema</code><code style="color:black;">&gt;</code></span></div>
</div>
Following screen shot shows the schema as visible in the BizTalk schema editor.

<a href="http://social.technet.microsoft.com/wiki/cfs-file.ashx/__key/communityserver-wikis-components-files/00-00-00-00-05/0310.BulkInsert-Operationxsd.JPG"> <img style="border-style:solid;border-width:0;" src="https://social.technet.microsoft.com/wiki/resized-image.ashx/__size/550x0/__key/communityserver-wikis-components-files/00-00-00-00-05/0310.BulkInsert-Operationxsd.JPG" alt="" /></a>

Following are the maps used to create the SQL insert message and the Response Returned by BizTalk WCF Service( The scripting functoid just returns the GUID as transaction if).

<span style="white-space:pre;"> <a href="http://social.technet.microsoft.com/wiki/cfs-file.ashx/__key/communityserver-wikis-components-files/00-00-00-00-05/4150.Map1.JPG"><img style="border-style:solid;border-width:0;" src="https://social.technet.microsoft.com/wiki/resized-image.ashx/__size/550x0/__key/communityserver-wikis-components-files/00-00-00-00-05/4150.Map1.JPG" alt="" /></a> </span>
<span style="white-space:pre;"> <a href="http://social.technet.microsoft.com/wiki/cfs-file.ashx/__key/communityserver-wikis-components-files/00-00-00-00-05/2553.Map2.JPG"><img style="border-style:solid;border-width:0;" src="https://social.technet.microsoft.com/wiki/resized-image.ashx/__size/550x0/__key/communityserver-wikis-components-files/00-00-00-00-05/2553.Map2.JPG" alt="" /></a> </span>

The Orchestration Flow is as Follows.

<span style="white-space:pre;"> <a href="http://social.technet.microsoft.com/wiki/cfs-file.ashx/__key/communityserver-wikis-components-files/00-00-00-00-05/8507.BizTalkOrchestration.jpg"><img style="border-style:solid;border-width:0;" src="https://social.technet.microsoft.com/wiki/resized-image.ashx/__size/550x0/__key/communityserver-wikis-components-files/00-00-00-00-05/8507.BizTalkOrchestration.jpg" alt="" /></a> </span>
<h2>Custom Receive Pipeline Using XDocument</h2>
The code for the receive pipeline using the XDocument is as follows. The Component loads the entire message and memory which causes a high memory utilization.
<div class="reCodeBlock" style="border:1px solid #7f9db9;overflow-y:auto;">
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.Collections.Generic;</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.Linq;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.Text;</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.Threading.Tasks;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">Microsoft.BizTalk.Message.Interop;</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">Microsoft.BizTalk.Component.Interop;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.IO;</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.Xml.Linq;</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">namespace</code> <code style="color:black;">ProcessLargeFilesInBizTalk.Components</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[ComponentCategory(CategoryTypes.CATID_PipelineComponent)]</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[ComponentCategory(CategoryTypes.CATID_Decoder)]</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[System.Runtime.InteropServices.Guid(</code><code style="color:blue;">"2B62CE58-5C73-483D-8CB2-76E2E3014057"</code><code style="color:black;">)]</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">class</code> <code style="color:black;">ExtractorClassesUsingXDocument : IBaseComponent, IComponentUI, IComponent, IPersistPropertyBag</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#region IBaseComponent</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">const</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">_Description = </code><code style="color:blue;">"Decode Pipeline Component To Extract the Data from Node and save To File"</code><code style="color:black;">;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">const</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">_Name = </code><code style="color:blue;">"ImageExtractorXDocument"</code><code style="color:black;">;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">const</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">_Version = </code><code style="color:blue;">"1.0.0.0"</code><code style="color:black;">;</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">Description</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_Description; }</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">Name</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_Name; }</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">Version</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_Version; }</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#endregion</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#region IComponentUI</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:black;">IntPtr _Icon = </code><code style="color:#006699;font-weight:bold;">new</code> <code style="color:black;">IntPtr();</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:black;">IntPtr Icon</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_Icon; }</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:black;">System.Collections.IEnumerator Validate(</code><code style="color:#006699;font-weight:bold;">object</code> <code style="color:black;">projectSystem)</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">return</code> <code style="color:#006699;font-weight:bold;">null</code><code style="color:black;">;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#endregion</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#region IpersistPropertyBag</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">_ImageDataNodeName;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">_LocalTempDataStore;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">_FileExtension;</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">ImageDataNodeName</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_ImageDataNodeName; }</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">set</code> <code style="color:black;">{ _ImageDataNodeName = value; }</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">LocalTempDataStore</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_LocalTempDataStore; }</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">set</code> <code style="color:black;">{ _LocalTempDataStore = value; }</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">FileExtension</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_FileExtension; }</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">set</code> <code style="color:black;">{ _FileExtension = value; }</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">void</code> <code style="color:black;">GetClassID(</code><code style="color:#006699;font-weight:bold;">out</code> <code style="color:black;">Guid classID)</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">classID = </code><code style="color:#006699;font-weight:bold;">new</code> <code style="color:black;">Guid(</code><code style="color:blue;">"2B62CE58-5C73-483D-8CB2-76E2E3014057"</code><code style="color:black;">);</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">void</code> <code style="color:black;">InitNew()</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">void</code> <code style="color:black;">Load(IPropertyBag pBag, </code><code style="color:#006699;font-weight:bold;">int</code> <code style="color:black;">errorLog)</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">object</code> <code style="color:black;">val1 = </code><code style="color:#006699;font-weight:bold;">null</code><code style="color:black;">;</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">object</code> <code style="color:black;">val2 = </code><code style="color:#006699;font-weight:bold;">null</code><code style="color:black;">;</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">object</code> <code style="color:black;">val3 = </code><code style="color:#006699;font-weight:bold;">null</code><code style="color:black;">;</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">try</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">pBag.Read(</code><code style="color:blue;">"ImageDataNodeName"</code><code style="color:black;">, </code><code style="color:#006699;font-weight:bold;">out</code> <code style="color:black;">val1, 0);</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">pBag.Read(</code><code style="color:blue;">"LocalTempDataStore"</code><code style="color:black;">, </code><code style="color:#006699;font-weight:bold;">out</code> <code style="color:black;">val2, 0);</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">pBag.Read(</code><code style="color:blue;">"FileExtension"</code><code style="color:black;">, </code><code style="color:#006699;font-weight:bold;">out</code> <code style="color:black;">val3, 0);</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">catch</code> <code style="color:black;">(ArgumentException)</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">catch</code> <code style="color:black;">(Exception ex)</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:#006699;font-weight:bold;">throw</code> <code style="color:#006699;font-weight:bold;">new</code> <code style="color:black;">ApplicationException(</code><code style="color:blue;">"Error Reading Property Bag. Error Message "</code> <code style="color:black;">+ ex.Message);</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">if</code> <code style="color:black;">(val1 != </code><code style="color:#006699;font-weight:bold;">null</code><code style="color:black;">)</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">_ImageDataNodeName = (</code><code style="color:#006699;font-weight:bold;">string</code><code style="color:black;">)val1;</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">if</code> <code style="color:black;">(val2 != </code><code style="color:#006699;font-weight:bold;">null</code><code style="color:black;">)</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">_LocalTempDataStore = (</code><code style="color:#006699;font-weight:bold;">string</code><code style="color:black;">)val2;</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">if</code> <code style="color:black;">(val3 != </code><code style="color:#006699;font-weight:bold;">null</code><code style="color:black;">)</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">_FileExtension = (</code><code style="color:#006699;font-weight:bold;">string</code><code style="color:black;">)val3;</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">void</code> <code style="color:black;">Save(IPropertyBag pBag, </code><code style="color:#006699;font-weight:bold;">bool</code> <code style="color:black;">clearDirty, </code><code style="color:#006699;font-weight:bold;">bool</code> <code style="color:black;">saveAllProperties)</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">object</code> <code style="color:black;">val1 = (</code><code style="color:#006699;font-weight:bold;">object</code><code style="color:black;">)_ImageDataNodeName;</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">object</code> <code style="color:black;">val2 = (</code><code style="color:#006699;font-weight:bold;">object</code><code style="color:black;">)_LocalTempDataStore;</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">object</code> <code style="color:black;">val3 = (</code><code style="color:#006699;font-weight:bold;">object</code><code style="color:black;">)_FileExtension;</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">pBag.Write(</code><code style="color:blue;">"ImageDataNodeName"</code><code style="color:black;">, val1);</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">pBag.Write(</code><code style="color:blue;">"LocalTempDataStore"</code><code style="color:black;">, val2);</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">pBag.Write(</code><code style="color:blue;">"FileExtension"</code><code style="color:black;">, val3);</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#endregion</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#region IComponent</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:black;">IBaseMessage Execute(IPipelineContext pContext, IBaseMessage inMsg)</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">Stream s = inMsg.BodyPart.GetOriginalDataStream();</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">XDocument xdoc = XDocument.Load(s);</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">IEnumerable&lt;XElement&gt; largeDataNodes = xdoc.Descendants(ImageDataNodeName);</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">foreach</code> <code style="color:black;">(XElement xe </code><code style="color:#006699;font-weight:bold;">in</code> <code style="color:black;">largeDataNodes)</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:#006699;font-weight:bold;">try</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">base64Data = xe.Value;</code></span></div>
<div style="background-color:white;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:black;">var data = </code><code style="color:#006699;font-weight:bold;">new</code> <code style="color:#006699;font-weight:bold;">byte</code><code style="color:black;">[4096];</code></span></div>
<div style="background-color:#f8f8f8;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">uniqueId = System.Guid.NewGuid().ToString();</code></span></div>
<div style="background-color:white;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">filePath = Path.Combine(_LocalTempDataStore, uniqueId + </code><code style="color:blue;">"."</code> <code style="color:black;">+ _FileExtension);</code></span></div>
<div style="background-color:#f8f8f8;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:#006699;font-weight:bold;">byte</code><code style="color:black;">[] imageData = Convert.FromBase64String(base64Data);</code></span></div>
<div style="background-color:white;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:black;">File.WriteAllBytes(filePath, imageData);</code></span></div>
<div style="background-color:#f8f8f8;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:#008200;">//replace the base64 string with the file path</code></span></div>
<div style="background-color:white;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:black;">xe.SetValue(filePath);</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:#006699;font-weight:bold;">catch</code> <code style="color:black;">(Exception)</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:#008200;">//Swallow the exception and move on to next node.</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">System.Diagnostics.EventLog.WriteEntry(</code><code style="color:blue;">"Xdoc"</code><code style="color:black;">, xdoc.ToString());</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">byte</code><code style="color:black;">[] outBytes = System.Text.Encoding.UTF8.GetBytes(xdoc.ToString());</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">MemoryStream ms = </code><code style="color:#006699;font-weight:bold;">new</code> <code style="color:black;">MemoryStream(outBytes);</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">inMsg.BodyPart.Data = ms;</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">inMsg;</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#endregion</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:black;">}</code></span></div>
</div>
<span style="font-size:12.1px;">The receive pipeline is created by registering above component to COM and then adding it to the decoder stage followed by an xml disassembler in the Disassemble <span style="font-size:12.1px;">stage of the pipeline. The Pipeline shown below</span></span><span style="font-size:12.1px;">.</span><span style="font-size:12.1px;"> </span>

<span style="font-size:12.1px;"><a style="color:#ff6600;font-size:12.1px;white-space:pre;" href="http://social.technet.microsoft.com/wiki/cfs-file.ashx/__key/communityserver-wikis-components-files/00-00-00-00-05/7624.spxdoc.JPG"> <img style="border-style:solid;border-width:0;" src="https://social.technet.microsoft.com/wiki/resized-image.ashx/__size/550x0/__key/communityserver-wikis-components-files/00-00-00-00-05/7624.spxdoc.JPG" alt="" /></a>

</span>
<h2>Custom Send Pipeline Using XDocument</h2>
<span style="font-size:12.1px;">The code for the receive pipeline using the XDocument is as follows. The Component loads the entire message and memory which causes a high memory utilization.</span>
<div class="reCodeBlock" style="border:1px solid #7f9db9;overflow-y:auto;">
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.Collections.Generic;</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.Linq;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.Text;</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.Threading.Tasks;</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">Microsoft.BizTalk.Message.Interop;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">Microsoft.BizTalk.Component.Interop;</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.IO;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.Xml;</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.Xml.Linq;</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">namespace</code> <code style="color:black;">ProcessLargeFilesInBizTalk.Components</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[ComponentCategory(CategoryTypes.CATID_PipelineComponent)]</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[ComponentCategory(CategoryTypes.CATID_Encoder)]</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[System.Runtime.InteropServices.Guid(</code><code style="color:blue;">"A5A1F8AD-3C4B-4CAB-A51D-D68EEAD56D67"</code><code style="color:black;">)]</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">class</code> <code style="color:black;">InjectorClassesUsingXDocument : IBaseComponent, IComponentUI, IComponent, IPersistPropertyBag</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#region IBaseComponent</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">const</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">_Description = </code><code style="color:blue;">"Encode Pipeline Component To Inject the Data into Node from file saved By Extractor"</code><code style="color:black;">;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">const</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">_Name = </code><code style="color:blue;">"ImageInjectorXDocument"</code><code style="color:black;">;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">const</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">_Version = </code><code style="color:blue;">"1.0.0.0"</code><code style="color:black;">;</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">Description</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_Description; }</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">Name</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_Name; }</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">Version</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_Version; }</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#endregion</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#region IComponentUI</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:black;">IntPtr _Icon = </code><code style="color:#006699;font-weight:bold;">new</code> <code style="color:black;">IntPtr();</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:black;">IntPtr Icon</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_Icon; }</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:black;">System.Collections.IEnumerator Validate(</code><code style="color:#006699;font-weight:bold;">object</code> <code style="color:black;">projectSystem)</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">return</code> <code style="color:#006699;font-weight:bold;">null</code><code style="color:black;">;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#endregion</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#region IpersistPropertyBag</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">_ImageDataNodeName;</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">ImageDataNodeName</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_ImageDataNodeName; }</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">set</code> <code style="color:black;">{ _ImageDataNodeName = value; }</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">void</code> <code style="color:black;">GetClassID(</code><code style="color:#006699;font-weight:bold;">out</code> <code style="color:black;">Guid classID)</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">classID = </code><code style="color:#006699;font-weight:bold;">new</code> <code style="color:black;">Guid(</code><code style="color:blue;">"A5A1F8AD-3C4B-4CAB-A51D-D68EEAD56D67"</code><code style="color:black;">);</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">void</code> <code style="color:black;">InitNew()</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">void</code> <code style="color:black;">Load(IPropertyBag pBag, </code><code style="color:#006699;font-weight:bold;">int</code> <code style="color:black;">errorLog)</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">object</code> <code style="color:black;">val1 = </code><code style="color:#006699;font-weight:bold;">null</code><code style="color:black;">;</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">try</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">pBag.Read(</code><code style="color:blue;">"ImageDataNodeName"</code><code style="color:black;">, </code><code style="color:#006699;font-weight:bold;">out</code> <code style="color:black;">val1, 0);</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">catch</code> <code style="color:black;">(ArgumentException)</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">catch</code> <code style="color:black;">(Exception ex)</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:#006699;font-weight:bold;">throw</code> <code style="color:#006699;font-weight:bold;">new</code> <code style="color:black;">ApplicationException(</code><code style="color:blue;">"Error Reading Property Bag. Error Message "</code> <code style="color:black;">+ ex.Message);</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">if</code> <code style="color:black;">(val1 != </code><code style="color:#006699;font-weight:bold;">null</code><code style="color:black;">)</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">_ImageDataNodeName = (</code><code style="color:#006699;font-weight:bold;">string</code><code style="color:black;">)val1;</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">void</code> <code style="color:black;">Save(IPropertyBag pBag, </code><code style="color:#006699;font-weight:bold;">bool</code> <code style="color:black;">clearDirty, </code><code style="color:#006699;font-weight:bold;">bool</code> <code style="color:black;">saveAllProperties)</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">object</code> <code style="color:black;">val1 = (</code><code style="color:#006699;font-weight:bold;">object</code><code style="color:black;">)_ImageDataNodeName;</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">pBag.Write(</code><code style="color:blue;">"ImageDataNodeName"</code><code style="color:black;">, val1);</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#endregion</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#region IComponent</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:black;">IBaseMessage Execute(IPipelineContext pContext, IBaseMessage inMsg)</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"> </span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">Stream s = inMsg.BodyPart.GetOriginalDataStream();</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">XDocument xDoc = XDocument.Load(s);</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">System.Diagnostics.EventLog.WriteEntry(</code><code style="color:blue;">"xdoc"</code><code style="color:black;">, xDoc.ToString());</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">XNamespace ns0 = </code><code style="color:blue;">"<a href="http://schemas.microsoft.com/Sql/2008/05/TypedProcedures/dbo">http://schemas.microsoft.com/Sql/2008/05/TypedProcedures/dbo</a>"</code><code style="color:black;">;</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">IEnumerable&lt;XElement&gt; imageDataNodes = xDoc.Descendants(ns0 + _ImageDataNodeName);</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">foreach</code> <code style="color:black;">(XElement xe </code><code style="color:#006699;font-weight:bold;">in</code> <code style="color:black;">imageDataNodes)</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:#006699;font-weight:bold;">try</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">filePath = xe.Value;</code></span></div>
<div style="background-color:#f8f8f8;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:black;">System.Diagnostics.EventLog.WriteEntry(</code><code style="color:blue;">"File Path"</code><code style="color:black;">, filePath);</code></span></div>
<div style="background-color:white;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:#006699;font-weight:bold;">byte</code><code style="color:black;">[] imageData = File.ReadAllBytes(filePath);</code></span></div>
<div style="background-color:#f8f8f8;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">image = Convert.ToBase64String(imageData);</code></span></div>
<div style="background-color:white;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:black;">xe.SetValue(image);</code></span></div>
<div style="background-color:#f8f8f8;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:black;">File.Delete(filePath); </code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:#006699;font-weight:bold;">catch</code> <code style="color:black;">(Exception ex)</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:#008200;">//Swallow the exception and move to next node.</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"> </span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">byte</code><code style="color:black;">[] outBytes = System.Text.Encoding.UTF8.GetBytes(xDoc.ToString());</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">MemoryStream ms = </code><code style="color:#006699;font-weight:bold;">new</code> <code style="color:black;">MemoryStream(outBytes);</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">inMsg.BodyPart.Data = ms;</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">inMsg;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#endregion</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">}</code></span></div>
</div>
The send pipeline is created by registering above component to the COM and then adding it to the encoder stage of the Send Pipeline. Following is a sample send pipeline.

<span style="white-space:pre;"> <a href="http://social.technet.microsoft.com/wiki/cfs-file.ashx/__key/communityserver-wikis-components-files/00-00-00-00-05/7624.spxdoc.JPG"><img style="border-style:solid;border-width:0;" src="https://social.technet.microsoft.com/wiki/resized-image.ashx/__size/550x0/__key/communityserver-wikis-components-files/00-00-00-00-05/7624.spxdoc.JPG" alt="" /></a></span>

&nbsp;
<h2>Custom Receive Pipeline Using BizTalk Streaming Classes</h2>
The streaming classes are located in the the Microsoft.BizTalk.Streaming.dll assembly and should be used while dealing with large messages. Following piece of code uses the XmlTranslator stream to modify the message and it does so at the boundary of the BizTalk so that the messages getting published in the message box are small messages. Following is the code for the receive pipeline decoder component.
<div class="reCodeBlock" style="border:1px solid #7f9db9;overflow-y:auto;">
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.Collections.Generic;</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.Linq;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.Text;</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.Threading.Tasks;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">Microsoft.BizTalk.Streaming;</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">Microsoft.BizTalk.Message.Interop;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">Microsoft.BizTalk.Component.Interop;</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.IO;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.Xml;</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">namespace</code> <code style="color:black;">ProcessLargeFilesInBizTalk.Components</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:grey;">/// &lt;summary&gt;</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:grey;">/// Class to implement the Custom Decoder Pipeline component which in turn uses the XMLTranlsatorStream to modify message contents.</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:grey;">/// This gives the advantage of modifying the message at the boundary of BizTalk and avoid publishing big messages to MSgBoxDatabase</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:grey;">/// &lt;/summary&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[ComponentCategory(CategoryTypes.CATID_PipelineComponent)]</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[ComponentCategory(CategoryTypes.CATID_Decoder)]</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[System.Runtime.InteropServices.Guid(</code><code style="color:blue;">"FA8EC02F-A33C-4F2A-A804-44E57FA4EEA3"</code><code style="color:black;">)]</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">class</code> <code style="color:black;">Extractor : IBaseComponent, IComponentUI, IComponent, IPersistPropertyBag</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#region IBaseComponent</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">const</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">_Description = </code><code style="color:blue;">"Decode Pipeline Component To Extract the Data from Node and save To File"</code><code style="color:black;">;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">const</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">_Name = </code><code style="color:blue;">"ImageExtractor"</code><code style="color:black;">;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">const</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">_Version = </code><code style="color:blue;">"1.0.0.0"</code><code style="color:black;">;</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">Description</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_Description; }</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">Name</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_Name; }</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">Version</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_Version; }</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#endregion</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#region IComponentUI</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:black;">IntPtr _Icon = </code><code style="color:#006699;font-weight:bold;">new</code> <code style="color:black;">IntPtr();</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:black;">IntPtr Icon</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_Icon; }</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:black;">System.Collections.IEnumerator Validate(</code><code style="color:#006699;font-weight:bold;">object</code> <code style="color:black;">projectSystem)</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">return</code> <code style="color:#006699;font-weight:bold;">null</code><code style="color:black;">;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#endregion</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#region IpersistPropertyBag</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">_ImageDataNodeName;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">_LocalTempDataStore;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">_FileExtension;</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">ImageDataNodeName</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_ImageDataNodeName; }</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">set</code> <code style="color:black;">{ _ImageDataNodeName = value; }</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">LocalTempDataStore</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_LocalTempDataStore; }</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">set</code> <code style="color:black;">{ _LocalTempDataStore = value; }</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">FileExtension</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_FileExtension; }</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">set</code> <code style="color:black;">{ _FileExtension = value; }</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">void</code> <code style="color:black;">GetClassID(</code><code style="color:#006699;font-weight:bold;">out</code> <code style="color:black;">Guid classID)</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">classID = </code><code style="color:#006699;font-weight:bold;">new</code> <code style="color:black;">Guid(</code><code style="color:blue;">"FA8EC02F-A33C-4F2A-A804-44E57FA4EEA3"</code><code style="color:black;">);</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">void</code> <code style="color:black;">InitNew()</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">void</code> <code style="color:black;">Load(IPropertyBag pBag, </code><code style="color:#006699;font-weight:bold;">int</code> <code style="color:black;">errorLog)</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">object</code> <code style="color:black;">val1 = </code><code style="color:#006699;font-weight:bold;">null</code><code style="color:black;">;</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">object</code> <code style="color:black;">val2 = </code><code style="color:#006699;font-weight:bold;">null</code><code style="color:black;">;</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">object</code> <code style="color:black;">val3 = </code><code style="color:#006699;font-weight:bold;">null</code><code style="color:black;">;</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">try</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">pBag.Read(</code><code style="color:blue;">"ImageDataNodeName"</code><code style="color:black;">, </code><code style="color:#006699;font-weight:bold;">out</code> <code style="color:black;">val1, 0);</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">pBag.Read(</code><code style="color:blue;">"LocalTempDataStore"</code><code style="color:black;">, </code><code style="color:#006699;font-weight:bold;">out</code> <code style="color:black;">val2, 0);</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">pBag.Read(</code><code style="color:blue;">"FileExtension"</code><code style="color:black;">, </code><code style="color:#006699;font-weight:bold;">out</code> <code style="color:black;">val3, 0);</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">catch</code> <code style="color:black;">(ArgumentException)</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">catch</code> <code style="color:black;">(Exception ex)</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:#006699;font-weight:bold;">throw</code> <code style="color:#006699;font-weight:bold;">new</code> <code style="color:black;">ApplicationException(</code><code style="color:blue;">"Error Reading Property Bag. Error Message "</code> <code style="color:black;">+ ex.Message);</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">if</code> <code style="color:black;">(val1 != </code><code style="color:#006699;font-weight:bold;">null</code><code style="color:black;">)</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">_ImageDataNodeName = (</code><code style="color:#006699;font-weight:bold;">string</code><code style="color:black;">)val1;</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">if</code> <code style="color:black;">(val2 != </code><code style="color:#006699;font-weight:bold;">null</code><code style="color:black;">)</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">_LocalTempDataStore = (</code><code style="color:#006699;font-weight:bold;">string</code><code style="color:black;">)val2;</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">if</code> <code style="color:black;">(val3 != </code><code style="color:#006699;font-weight:bold;">null</code><code style="color:black;">)</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">_FileExtension = (</code><code style="color:#006699;font-weight:bold;">string</code><code style="color:black;">)val3;</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">void</code> <code style="color:black;">Save(IPropertyBag pBag, </code><code style="color:#006699;font-weight:bold;">bool</code> <code style="color:black;">clearDirty, </code><code style="color:#006699;font-weight:bold;">bool</code> <code style="color:black;">saveAllProperties)</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">object</code> <code style="color:black;">val1 = (</code><code style="color:#006699;font-weight:bold;">object</code><code style="color:black;">)_ImageDataNodeName;</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">object</code> <code style="color:black;">val2 = (</code><code style="color:#006699;font-weight:bold;">object</code><code style="color:black;">)_LocalTempDataStore;</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">object</code> <code style="color:black;">val3 = (</code><code style="color:#006699;font-weight:bold;">object</code><code style="color:black;">)_FileExtension;</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">pBag.Write(</code><code style="color:blue;">"ImageDataNodeName"</code><code style="color:black;">, val1);</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">pBag.Write(</code><code style="color:blue;">"LocalTempDataStore"</code><code style="color:black;">, val2);</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">pBag.Write(</code><code style="color:blue;">"FileExtension"</code><code style="color:black;">, val3);</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#endregion</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#region IComponent</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:black;">IBaseMessage Execute(IPipelineContext pContext, IBaseMessage inMsg)</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">VirtualStream vs = </code><code style="color:#006699;font-weight:bold;">new</code> <code style="color:black;">VirtualStream(inMsg.BodyPart.GetOriginalDataStream());</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">XmlReader xReader = XmlReader.Create(vs);</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">inMsg.BodyPart.Data = </code><code style="color:#006699;font-weight:bold;">new</code> <code style="color:black;">ExtractorStream(xReader, _FileExtension, _ImageDataNodeName, _LocalTempDataStore);</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">inMsg;</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#endregion</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:grey;">/// &lt;summary&gt;</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:grey;">/// This class is derived from the XmlTranslatorStream class( Present in the BizTalk Streaming Namespace)</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:grey;">/// It overrides some of the functions to replace the image recieved as a base64string in the request to BizTalk with the path to the temp file folder where the image</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:grey;">/// is temporarily stored.</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:grey;">/// &lt;/summary&gt;</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">class</code> <code style="color:black;">ExtractorStream : XmlTranslatorStream</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:black;">XmlReader reader;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:black;">XmlWriter writer;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">fileExtension;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">imageNodeName;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">localTempDataStorePath;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:black;">XmlReader xReader;</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:black;">ExtractorStream(XmlReader xReader, </code><code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">fileExtension, </code><code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">imageNodeName, </code><code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">localTempDataStorePath)</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">: </code><code style="color:#006699;font-weight:bold;">base</code><code style="color:black;">(xReader)</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">this</code><code style="color:black;">.fileExtension = fileExtension;</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">this</code><code style="color:black;">.imageNodeName = imageNodeName;</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">this</code><code style="color:black;">.localTempDataStorePath = localTempDataStorePath;</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">Initialise();</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">void</code> <code style="color:black;">Initialise()</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">reader = m_reader;</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">writer = m_writer;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">/// &lt;summary&gt;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">/// Ovverrides the TranslateStartElement function to check if the current node is the node which contains the image( as base64String)</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">/// If so, the function, reads the value of the node and saves the base64String as a jpg file in the temporary data store. It then replaces the </code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">/// image string with path to the data store.</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">/// &lt;/summary&gt;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">/// &lt;param name="prefix"&gt;&lt;/param&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">/// &lt;param name="localName"&gt;&lt;/param&gt;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">/// &lt;param name="nsURI"&gt;&lt;/param&gt;</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">protected</code> <code style="color:#006699;font-weight:bold;">override</code> <code style="color:#006699;font-weight:bold;">void</code> <code style="color:black;">TranslateStartElement(</code><code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">prefix, </code><code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">localName, </code><code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">nsURI)</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">if</code> <code style="color:black;">(reader.LocalName == imageNodeName)</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">var document = reader.ReadSubtree();</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">document.Read();</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">var data = </code><code style="color:#006699;font-weight:bold;">new</code> <code style="color:#006699;font-weight:bold;">byte</code><code style="color:black;">[4096];</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">uniqueId = System.Guid.NewGuid().ToString();</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">filePath = Path.Combine(localTempDataStorePath, uniqueId + </code><code style="color:blue;">"."</code> <code style="color:black;">+ fileExtension);</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">(FileStream fs = File.Open(filePath, FileMode.CreateNew, FileAccess.Write, FileShare.Read))</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:#006699;font-weight:bold;">int</code> <code style="color:black;">bytesRead;</code></span></div>
<div style="background-color:white;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:#006699;font-weight:bold;">while</code> <code style="color:black;">((bytesRead = document.ReadElementContentAsBase64(data, 0, data.Length)) &gt; 0)</code></span></div>
<div style="background-color:#f8f8f8;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>                        </code><span style="margin-left:72px !important;"><code style="color:black;">fs.Write(data, 0, bytesRead);</code></span></div>
<div style="background-color:#f8f8f8;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:black;">fs.Flush();</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">writer.WriteElementString(prefix, imageNodeName, nsURI, filePath);</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">else</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:#006699;font-weight:bold;">base</code><code style="color:black;">.TranslateStartElement(prefix, localName, nsURI);</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">protected</code> <code style="color:#006699;font-weight:bold;">override</code> <code style="color:#006699;font-weight:bold;">void</code> <code style="color:black;">TranslateEndElement(</code><code style="color:#006699;font-weight:bold;">bool</code> <code style="color:black;">full)</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">if</code> <code style="color:black;">(reader.LocalName == imageNodeName)</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">else</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:#006699;font-weight:bold;">base</code><code style="color:black;">.TranslateEndElement(full);</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:black;">}</code></span></div>
</div>
The receive pipeline is created by registering above component to COM and then adding it to the decoder stage followed by an xml disassembler in the Disassemble stage of the pipeline. The Pipeline shown below.

<a href="http://social.technet.microsoft.com/wiki/cfs-file.ashx/__key/communityserver-wikis-components-files/00-00-00-00-05/5287.rpstreaming.JPG"> <img style="border-style:solid;border-width:0;" src="https://social.technet.microsoft.com/wiki/resized-image.ashx/__size/550x0/__key/communityserver-wikis-components-files/00-00-00-00-05/5287.rpstreaming.JPG" alt="" /></a>
<h2>Custom Send Pipeline Using BizTalk Streaming Classes</h2>
<span style="font-size:12.1px;">The streaming classes are located in the the Microsoft.BizTalk.Streaming.dll assembly and should be used while dealing with large messages. Following piece of code uses the XmlTranslator stream to modify the message and it does so at the boundary of the BizTalk so that the messages getting published in the message box are small messages. Following is the code for the send pipeline coder component.</span>
<div class="reCodeBlock" style="border:1px solid #7f9db9;overflow-y:auto;">
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.Collections.Generic;</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.Linq;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.Text;</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.Threading.Tasks;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">Microsoft.BizTalk.Streaming;</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">Microsoft.BizTalk.Message.Interop;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">Microsoft.BizTalk.Component.Interop;</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.IO;</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">using</code> <code style="color:black;">System.Xml;</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#006699;font-weight:bold;">namespace</code> <code style="color:black;">ProcessLargeFilesInBizTalk.Components</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:grey;">/// &lt;summary&gt;</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:grey;">/// Class to implement the Custom Encoder Pipeline component which in turn uses the XMLTranlsatorStream to modify message contents.</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:grey;">/// This gives the advantage of modifying the message at the boundary of BizTalk and avoid publishing big messages to MSgBoxDatabase</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:grey;">/// &lt;/summary&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[ComponentCategory(CategoryTypes.CATID_PipelineComponent)]</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[ComponentCategory(CategoryTypes.CATID_Encoder)]</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[System.Runtime.InteropServices.Guid(</code><code style="color:blue;">"DD061000-A85D-4339-BF2A-531F9F4D529D"</code><code style="color:black;">)]</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">class</code> <code style="color:black;">Injector : IBaseComponent, IComponentUI, IComponent, IPersistPropertyBag</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#region IBaseComponent</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">const</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">_Description = </code><code style="color:blue;">"Encode Pipeline Component To Inject the Data into Node from file saved By Extractor"</code><code style="color:black;">;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">const</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">_Name = </code><code style="color:blue;">"ImageInjector"</code><code style="color:black;">;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">const</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">_Version = </code><code style="color:blue;">"1.0.0.0"</code><code style="color:black;">;</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">Description</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_Description; }</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">Name</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_Name; }</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">Version</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_Version; }</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#endregion</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#region IComponentUI</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:black;">IntPtr _Icon = </code><code style="color:#006699;font-weight:bold;">new</code> <code style="color:black;">IntPtr();</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:black;">IntPtr Icon</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_Icon; }</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:black;">System.Collections.IEnumerator Validate(</code><code style="color:#006699;font-weight:bold;">object</code> <code style="color:black;">projectSystem)</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">return</code> <code style="color:#006699;font-weight:bold;">null</code><code style="color:black;">;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#endregion</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#region IpersistPropertyBag</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">_ImageDataNodeName;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"> </span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">ImageDataNodeName</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">get</code> <code style="color:black;">{ </code><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">_ImageDataNodeName; }</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">set</code> <code style="color:black;">{ _ImageDataNodeName = value; }</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"> </span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">void</code> <code style="color:black;">GetClassID(</code><code style="color:#006699;font-weight:bold;">out</code> <code style="color:black;">Guid classID)</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">classID = </code><code style="color:#006699;font-weight:bold;">new</code> <code style="color:black;">Guid(</code><code style="color:blue;">"DD061000-A85D-4339-BF2A-531F9F4D529D"</code><code style="color:black;">);</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">void</code> <code style="color:black;">InitNew()</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">void</code> <code style="color:black;">Load(IPropertyBag pBag, </code><code style="color:#006699;font-weight:bold;">int</code> <code style="color:black;">errorLog)</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">object</code> <code style="color:black;">val1 = </code><code style="color:#006699;font-weight:bold;">null</code><code style="color:black;">;</code></span></div>
<div style="background-color:#f8f8f8;"><code>           </code><span style="margin-left:33px !important;"> </span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">try</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">pBag.Read(</code><code style="color:blue;">"ImageDataNodeName"</code><code style="color:black;">, </code><code style="color:#006699;font-weight:bold;">out</code> <code style="color:black;">val1, 0);</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"> </span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">catch</code> <code style="color:black;">(ArgumentException)</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">catch</code> <code style="color:black;">(Exception ex)</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:#006699;font-weight:bold;">throw</code> <code style="color:#006699;font-weight:bold;">new</code> <code style="color:black;">ApplicationException(</code><code style="color:blue;">"Error Reading Property Bag. Error Message "</code> <code style="color:black;">+ ex.Message);</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">if</code> <code style="color:black;">(val1 != </code><code style="color:#006699;font-weight:bold;">null</code><code style="color:black;">)</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">_ImageDataNodeName = (</code><code style="color:#006699;font-weight:bold;">string</code><code style="color:black;">)val1;</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">void</code> <code style="color:black;">Save(IPropertyBag pBag, </code><code style="color:#006699;font-weight:bold;">bool</code> <code style="color:black;">clearDirty, </code><code style="color:#006699;font-weight:bold;">bool</code> <code style="color:black;">saveAllProperties)</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">object</code> <code style="color:black;">val1 = (</code><code style="color:#006699;font-weight:bold;">object</code><code style="color:black;">)_ImageDataNodeName; </code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">pBag.Write(</code><code style="color:blue;">"ImageDataNodeName"</code><code style="color:black;">, val1);</code></span></div>
<div style="background-color:#f8f8f8;"><code>           </code><span style="margin-left:33px !important;"> </span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#endregion</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#region IComponent</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:black;">IBaseMessage Execute(IPipelineContext pContext, IBaseMessage inMsg)</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">VirtualStream vs = </code><code style="color:#006699;font-weight:bold;">new</code> <code style="color:black;">VirtualStream(inMsg.BodyPart.GetOriginalDataStream());</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">XmlReader xReader = XmlReader.Create(vs);</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">inMsg.BodyPart.Data = </code><code style="color:#006699;font-weight:bold;">new</code> <code style="color:black;">InjectorStream(xReader,_ImageDataNodeName);</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">return</code> <code style="color:black;">inMsg;</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">#endregion</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:grey;">/// &lt;summary&gt;</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:grey;">/// This class is derived from the XmlTranslatorStream class( Present in the BizTalk Streaming Namespace)</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:grey;">/// It overrides some of the dinctions to replace the image file location with the the actual file stored at the </code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:grey;">/// Physical data store. Once the File is converted to base64 string, and written to the message, The file in the temp data store is deleted</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:grey;">/// &lt;/summary&gt;</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:#006699;font-weight:bold;">class</code> <code style="color:black;">InjectorStream : XmlTranslatorStream</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:black;">XmlReader reader;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:black;">XmlWriter writer;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">imageDataNodeName;</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">/// &lt;summary&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">/// </code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">/// &lt;/summary&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">/// &lt;param name="xReader"&gt; Reader created from the virtual stream created from the BizTalk message to SQL adapter&lt;/param&gt;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">/// &lt;param name="imageDataNodeName"&gt;Node with the base64binary type which stores the image data too be inserted into SQL table&lt;/param&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">public</code> <code style="color:black;">InjectorStream(XmlReader xReader, </code><code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">imageDataNodeName)</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">:</code><code style="color:#006699;font-weight:bold;">base</code><code style="color:black;">(xReader)</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">this</code><code style="color:black;">.imageDataNodeName = imageDataNodeName;</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">Initialise();</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">private</code> <code style="color:#006699;font-weight:bold;">void</code> <code style="color:black;">Initialise()</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">reader = m_reader;</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">writer = m_writer;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">/// &lt;summary&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">/// Ovverride the TranslateStartElement method to check if the current node is the node which contains the path to the image stored at file store</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">/// If the current node matches the image node, the method, reads the file from the data store, updates the value of the node</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">/// usng writer and once done deletes the file from temp folder</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">/// &lt;/summary&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">/// &lt;param name="prefix"&gt;&lt;/param&gt;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">/// &lt;param name="localName"&gt;&lt;/param&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:grey;">/// &lt;param name="nsURI"&gt;&lt;/param&gt;</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">protected</code> <code style="color:#006699;font-weight:bold;">override</code> <code style="color:#006699;font-weight:bold;">void</code> <code style="color:black;">TranslateStartElement(</code><code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">prefix, </code><code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">localName, </code><code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">nsURI)</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">if</code> <code style="color:black;">(reader.LocalName == imageDataNodeName)</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">var document = reader.ReadSubtree();</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">document.Read();</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">var imageData = String.Empty;</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:#006699;font-weight:bold;">string</code> <code style="color:black;">tempFilePath = document.ReadElementContentAsString();</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:#006699;font-weight:bold;">if</code> <code style="color:black;">(File.Exists(tempFilePath))</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:#006699;font-weight:bold;">byte</code><code style="color:black;">[] imageBytes = File.ReadAllBytes(tempFilePath);</code></span></div>
<div style="background-color:white;"><code>                    </code><span style="margin-left:60px !important;"><code style="color:black;">imageData = Convert.ToBase64String(imageBytes);</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">writer.WriteElementString(localName, nsURI, imageData);</code></span></div>
<div style="background-color:#f8f8f8;"><code>                </code><span style="margin-left:48px !important;"><code style="color:black;">File.Delete(tempFilePath);</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">else</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>                </code><span style="margin-left:48px !important;"><code style="color:#006699;font-weight:bold;">base</code><code style="color:black;">.TranslateStartElement(prefix, localName, nsURI);</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"><code style="color:#006699;font-weight:bold;">protected</code> <code style="color:#006699;font-weight:bold;">override</code> <code style="color:#006699;font-weight:bold;">void</code> <code style="color:black;">TranslateEndElement(</code><code style="color:#006699;font-weight:bold;">bool</code> <code style="color:black;">full)</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:36px !important;"><code style="color:#006699;font-weight:bold;">base</code><code style="color:black;">.TranslateEndElement(full);</code></span></div>
<div style="background-color:white;"><code>        </code><span style="margin-left:24px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:24px !important;"> </span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">}</code></span></div>
</div>
The send pipeline is created by registering above component in COM and adding it to the encoder stage of the send pipeline. Following is a sample pipeline.

<a href="http://social.technet.microsoft.com/wiki/cfs-file.ashx/__key/communityserver-wikis-components-files/00-00-00-00-05/5430.spStreaming.JPG"> <img style="border-style:solid;border-width:0;" src="https://social.technet.microsoft.com/wiki/resized-image.ashx/__size/550x0/__key/communityserver-wikis-components-files/00-00-00-00-05/5430.spStreaming.JPG" alt="" /></a>

This completes the implementation part of the walkthrough.

<a href="https://www.blogger.com/blogger.g?blogID=8130563676268724620#Top">↑Back To Top </a>

<hr />

Deploying BizTalk Solution


Following are the general steps that are followed to deploy and configure the BizTalk Solution.
<ol>
	<li>Deploy the BizTalk solution.</li>
	<li>Expose the BizTalk Schema BizTalkServiceTypes as WCF Service.</li>
	<li>Configure the IIS location by mapping the IIS application to a correct Application Pool.</li>
	<li>Once the schemas are exposed as wcf service, a receive location and port will be created in the BizTalk application. Next step is to set up the maximum size the allowed for the messages. Refer to following sample screenshot.<span style="white-space:pre;"> <a style="color:#ff6600;font-size:12.1px;" href="http://social.technet.microsoft.com/wiki/cfs-file.ashx/__key/communityserver-wikis-components-files/00-00-00-00-05/8867.receiveLocationHighMessageSize.JPG"><img style="border-style:solid;border-width:0;" src="https://social.technet.microsoft.com/wiki/resized-image.ashx/__size/550x0/__key/communityserver-wikis-components-files/00-00-00-00-05/8867.receiveLocationHighMessageSize.JPG" alt="" /></a> </span>

<span style="font-size:12.1px;">The configuration of the receive pipeline using the XDocument class is shown below.</span>

<span style="white-space:pre;"> <a style="color:#ff6600;font-size:12.1px;" href="http://social.technet.microsoft.com/wiki/cfs-file.ashx/__key/communityserver-wikis-components-files/00-00-00-00-05/6683.receivexDocPipeline.JPG"><img style="border-style:solid;border-width:0;" src="https://social.technet.microsoft.com/wiki/resized-image.ashx/__size/550x0/__key/communityserver-wikis-components-files/00-00-00-00-05/6683.receivexDocPipeline.JPG" alt="" /></a> </span>

<span style="font-size:12.1px;white-space:pre;">The configuration of the receive pipeline using the Streaming classes is shown below.</span>



<span style="white-space:pre;"> <a href="http://social.technet.microsoft.com/wiki/cfs-file.ashx/__key/communityserver-wikis-components-files/00-00-00-00-05/7345.receiveStreamingPipeline.JPG"><img style="border-style:solid;border-width:0;" src="https://social.technet.microsoft.com/wiki/resized-image.ashx/__size/550x0/__key/communityserver-wikis-components-files/00-00-00-00-05/7345.receiveStreamingPipeline.JPG" alt="" /></a> </span></li>
	<li>Import the Binding File generated by Consuming the stored proc. This will created the Sendport to insert the data into SQL. Set the Action Mapping to <strong>CompositeOperation</strong> as shown below.<span style="white-space:pre;"> <a href="http://social.technet.microsoft.com/wiki/cfs-file.ashx/__key/communityserver-wikis-components-files/00-00-00-00-05/1447.actionmapping.JPG"><img style="border-style:solid;border-width:0;" src="https://social.technet.microsoft.com/wiki/resized-image.ashx/__size/350x0/__key/communityserver-wikis-components-files/00-00-00-00-05/1447.actionmapping.JPG" alt="" /></a> </span>

The configuration for the send pipeline using streaming classes is shown below.

<span style="white-space:pre;"> <a href="http://social.technet.microsoft.com/wiki/cfs-file.ashx/__key/communityserver-wikis-components-files/00-00-00-00-05/0245.sndstreampipeline.JPG"><img style="border-style:solid;border-width:0;" src="https://social.technet.microsoft.com/wiki/resized-image.ashx/__size/550x0/__key/communityserver-wikis-components-files/00-00-00-00-05/0245.sndstreampipeline.JPG" alt="" /></a> </span>

<span style="font-size:12.1px;">The configuration for the sen pipeline using streaming classes is shown below.

</span>

<span style="white-space:pre;"> <a href="http://social.technet.microsoft.com/wiki/cfs-file.ashx/__key/communityserver-wikis-components-files/00-00-00-00-05/1586.receivexDocPipeline.JPG"><img style="border-style:solid;border-width:0;" src="https://social.technet.microsoft.com/wiki/resized-image.ashx/__size/550x0/__key/communityserver-wikis-components-files/00-00-00-00-05/1586.receivexDocPipeline.JPG" alt="" /></a> </span></li>
	<li>In order to take care of the stripped image files in the temporary file store, a PowerShell script can be configured to clear the files out at the end of the business day. A sample script which deletes the files older than 1 day is as follows.
<div class="reCodeBlock" style="border:1px solid #7f9db9;overflow-y:auto;">
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:black;">param</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">(</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[Parameter(Mandatory=$true)]</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">$ThresholdMinutes,</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[Parameter(Mandatory=$true)]</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">$PathFromWhereToDelete</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">)</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:black;">if(Test-Path $PathFromWhereToDelete)</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">$currentDate = get-date</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"> </span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">Get-ChildItem -Path $PathFromWhereToDelete -File | where-object{$_.LastWriteTime -lt $currentDate.AddMinutes(-$ThresholdMinutes)} | Remove-Item</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">}</code></span></div>
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:black;">else</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">write-Host (</code><code style="color:blue;">"Path Not found to delete files from"</code><code style="color:black;">)</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">}</code></span></div>
</div></li>
</ol>
&nbsp;

<hr />

<h1>Testing</h1>
The testing of the application was done by sending out multiple calls to the BizTalk service each containing a message of size around 20 MB and the memory and response times were noted for each of following scenarios.
<ol>
	<li>Direct Mapping allowing the entire message to publish into the message box</li>
	<li>Using the Pipleine Components with the XDocument Class</li>
	<li>Using the Pipeline Components with the Streaming Classes</li>
</ol>
The results are mentioned below.
<h2>Direct  Mapping Results</h2>
Response Time: 32308ms

<span style="font-size:12.1px;">The response time for the large message and its properties are noted in the SOAP UI request made to the BizTalk WCF service. </span>

<span style="white-space:pre;"> <a href="http://social.technet.microsoft.com/wiki/cfs-file.ashx/__key/communityserver-wikis-components-files/00-00-00-00-05/3582.WithoutPipelines.JPG"><img style="border-style:solid;border-width:0;" src="https://social.technet.microsoft.com/wiki/resized-image.ashx/__size/550x0/__key/communityserver-wikis-components-files/00-00-00-00-05/3582.WithoutPipelines.JPG" alt="" /></a> </span>

<strong>Memory Utilization</strong>

Following Screen Shot captures the Perfmon counter .

<span style="white-space:pre;"> <a href="http://social.technet.microsoft.com/wiki/cfs-file.ashx/__key/communityserver-wikis-components-files/00-00-00-00-05/8231.WithoutPipelinesPerfmon.jpg"><img style="border-style:solid;border-width:0;" src="https://social.technet.microsoft.com/wiki/resized-image.ashx/__size/550x0/__key/communityserver-wikis-components-files/00-00-00-00-05/8231.WithoutPipelinesPerfmon.jpg" alt="" /></a> </span>
<h2><span style="white-space:pre;">Pipeline Components Using XDocument Class</span></h2>
<span style="white-space:pre;"><span style="font-size:12.1px;"><strong>Response Time</strong>: 5227ms</span><br style="font-size:12.1px;" />
<span style="font-size:12.1px;">The response time for the large message and its properties are noted in the SOAP UI request made to the BizTalk WCF service. </span>

</span>

<span style="font-size:12.1px;white-space:pre;"> <a style="color:#ff6600;font-size:12.1px;" href="http://social.technet.microsoft.com/wiki/cfs-file.ashx/__key/communityserver-wikis-components-files/00-00-00-00-05/0005.withxDocumentPipelines.JPG"><img style="border-style:solid;border-width:0;" src="https://social.technet.microsoft.com/wiki/resized-image.ashx/__size/550x0/__key/communityserver-wikis-components-files/00-00-00-00-05/0005.withxDocumentPipelines.JPG" alt="" /></a></span>

<span style="font-size:12.1px;white-space:pre;">

</span>
<div style="font-size:12.1px;"><strong>Memory Utilization</strong></div>
<div style="font-size:12.1px;">Following Screen Shot captures the Perfmon counter</div>
<span style="white-space:pre;"> <a style="color:#ff6600;font-size:12.1px;" href="http://social.technet.microsoft.com/wiki/cfs-file.ashx/__key/communityserver-wikis-components-files/00-00-00-00-05/3443.withxDocumentPipelinesPerfmon.JPG"><img style="border-style:solid;border-width:0;" src="https://social.technet.microsoft.com/wiki/resized-image.ashx/__size/550x0/__key/communityserver-wikis-components-files/00-00-00-00-05/3443.withxDocumentPipelinesPerfmon.JPG" alt="" /></a> </span>


<span style="color:#3a3e43;font-family:&quot;font-size:20px;white-space:pre;">Pipeline Components Using Streaming Class</span>

<span style="font-size:12.1px;white-space:pre;"><strong>Response Time</strong>: 5227ms</span><br style="font-size:12.1px;white-space:pre;" />
<span style="font-size:12.1px;white-space:pre;">The response time for the large message and its properties are noted in the SOAP UI request made to the BizTalk WCF service.</span>

<span style="font-size:12.1px;white-space:pre;"> <a style="color:#ff6600;font-size:12.1px;" href="http://social.technet.microsoft.com/wiki/cfs-file.ashx/__key/communityserver-wikis-components-files/00-00-00-00-05/4606.withxDocumentStreamingPipelines.JPG"><img style="border-style:solid;border-width:0;" src="https://social.technet.microsoft.com/wiki/resized-image.ashx/__size/550x0/__key/communityserver-wikis-components-files/00-00-00-00-05/4606.withxDocumentStreamingPipelines.JPG" alt="" /></a></span>

<span style="font-size:12.1px;white-space:pre;">

</span>
<div style="font-size:12.1px;"><strong>Memory Utilization</strong></div>
<div style="font-size:12.1px;">Following Screen Shot captures the Perfmon counter</div>
<span style="white-space:pre;"> <a style="color:#ff6600;font-size:12.1px;" href="http://social.technet.microsoft.com/wiki/cfs-file.ashx/__key/communityserver-wikis-components-files/00-00-00-00-05/2287.withxDocumentStreamingPipelinesPerfmon.JPG"><img style="border-style:solid;border-width:0;" src="https://social.technet.microsoft.com/wiki/resized-image.ashx/__size/550x0/__key/communityserver-wikis-components-files/00-00-00-00-05/2287.withxDocumentStreamingPipelinesPerfmon.JPG" alt="" /></a> </span>

<hr />

<h1>Conclusion</h1>
Based upon the results from the testing it is clear that the Streaming classes should be used while dealing with large messages to improve the performance. In case a decision needs to be made between XDocument or XmlDocument and Streaming Classes, a decision can be made based upon the size of the message that needs to be processed.

<hr />

<h1>See Also</h1>
Following articles can be visited for more reading up on the topics discussed in this article.
<ol>
	<li><a href="https://docs.microsoft.com/en-us/biztalk/core/publish-schemas-as-wcf-services--use-the-biztalk-wcf-service-publishing-wizard" target="_blank" rel="noopener noreferrer">How to Use the BizTalk WCF Service Publishing Wizard to Publish Schemas as WCF Services</a></li>
	<li><a href="https://blogs.msdn.microsoft.com/brajens/2006/11/25/how-to-develop-biztalk-custom-pipeline-components-part1/" target="_blank" rel="noopener noreferrer">How to Develop BizTalk Custom Pipeline Components – Part1</a></li>
	<li><a href="https://docs.microsoft.com/en-us/biztalk/adapters-and-accelerators/adapter-sql/run-composite-operations-on-sql-server-using-biztalk-server" target="_blank" rel="noopener noreferrer">Run composite operations on SQL Server using BizTalk Server</a></li>
	<li><a href="https://social.technet.microsoft.com/wiki/contents/articles/29146.biztalk-server-2013-crud-operation-with-wcf-sql-adapter-and-correlation.aspx" target="_blank" rel="noopener noreferrer">BizTalk Server 2013: CRUD Operation With WCF-SQL Adapter and Correlation</a></li>
	<li><a href="https://social.technet.microsoft.com/wiki/contents/articles/24803.biztalk-server-sql-patterns-for-polling-and-batch-retrieve.aspx" target="_blank" rel="noopener noreferrer">BizTalk Server: SQL Patterns for Polling and Batch Retrieve</a></li>
	<li><a href="https://msdn.microsoft.com/en-us/library/dn529004.aspx" target="_blank" rel="noopener noreferrer">Windows PowerShell Basics</a></li>
</ol>
In order to learn more about the BizTalk product, a good place to start is <a href="https://social.technet.microsoft.com/wiki/contents/articles/2240.biztalk-server-resources-on-the-technet-wiki.aspx" target="_blank" rel="noopener noreferrer">BizTalk Server Resources on the TechNet Wiki</a>

<hr />

<h1>References</h1>
Following articles were referred while writing this article.
<ol>
	<li><a href="https://social.technet.microsoft.com/wiki/contents/articles/34263.biztalk-server-processing-large-files-streaming.aspx" target="_blank" rel="noopener noreferrer">BizTalk Server: Processing large files (streaming)</a> by <a href="https://social.technet.microsoft.com/profile/eldert%20%20grootenboer/" target="_blank" rel="noopener noreferrer">Eldert Grootenboer</a></li>
	<li><a href="https://social.technet.microsoft.com/wiki/contents/articles/29169.biztalk-server-custom-pipeline-optimization-using-the-virtual-stream-class.aspx" target="_blank" rel="noopener noreferrer">BizTalk Server: Custom pipeline optimization using the Virtual Stream class</a> by <a href="https://social.technet.microsoft.com/profile/steef-jan%20wiggers/" target="_blank" rel="noopener noreferrer">Steef-Jan Wiggers</a></li>
	<li><a href="https://msdn.microsoft.com/en-us/library/microsoft.biztalk.streaming.aspx" target="_blank" rel="noopener noreferrer">Microsoft.BizTalk.Streaming Namespace</a></li>
	<li><a href="https://docs.microsoft.com/en-us/biztalk/technical-guides/optimizing-memory-usage-with-streaming" target="_blank" rel="noopener noreferrer">Optimizing Memory Usage with Streaming</a></li>
</ol>
&nbsp;

<hr />

&nbsp;

</div>