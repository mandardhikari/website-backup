---
ID: 45
post_title: 'Logic Apps 101: Inserting Data Into Multiple Tables Using SQL Connector and Trigger(SQL)'
author: Mandar Dharmadhikari
post_excerpt: ""
layout: post
permalink: >
  https://theabodeofcode.com/logic-apps-101-inserting-data-into-multiple-tables-using-sql-connector-and-triggersql/
published: true
post_date: 2018-07-09 07:54:08
---
<div dir="ltr" style="text-align:left;"><span style="font-family:'arial' , 'helvetica' , sans-serif;">
</span>
<h1><span style="font-family:'arial' , 'helvetica' , sans-serif;">
Introduction</span></h1>
<div style="text-align:left;"><span style="font-family:'arial' , 'helvetica' , sans-serif;font-size:x-normal;">In modern day integration, there are many cases where the front end(web app/api) is hosted in cloud and some or whole part of the data store is on cloud or on premises and the integration code has to populate the data /fetch and perform the CRUD operations on the tables based upon the request. In such scenarios, Logic Apps is best suited to integrate the web app/api and the data store as Logic App provides a easy and visual way of integrating the systems( in form of a work flow).</span></div>
<h2><span style="font-family:'arial' , 'helvetica' , sans-serif;">
What are Logic Apps?</span></h2>
<span style="font-family:'arial' , 'helvetica' , sans-serif;font-size:x-normal;">Logic Apps are a piece of integration workflow hosted on Azure which are used to create scale-able integrations between various systems.These are very easy to design and provide connectivity between various disparate systems using many out of the box connectors as well as with the facility to design custom connectors for specific purposes. This makes integration easier than ever as the design aspect of the earlier complex integrations is made easy with minimum steps required to get a workflow in place and get it running.</span>
<h1><span style="font-family:'arial' , 'helvetica' , sans-serif;">
Scope</span></h1>
<span style="font-family:'arial' , 'helvetica' , sans-serif;font-size:x-normal;">This article deals with how to insert similar data to multiple tables in a datastore using logic app. The logic app exposes an HTTPS endpoint which can be consumed by the front end web api/app and it can send the data to the Logic App. The end points accepts json payload and inserts the data received over the HTTPS call into multiple tables hosted on an on premises SQL server. This article aims to discuss how to insert the data to sql tables using a single call from the logic app ( by making use of SQL After Insert triggers) as opposed to the conventional way of  calling two different stored procedures/ insert row actions. The approach discussed in this article reduces the number of action that an Logic App needs to perform and thus saves user from getting billed for multiple actions ( as number of actions are billed during an logic app run). This article assumes that the reader is a beginner with a basic knowledge about Microsoft Azure and Logic Apps and guides the user to try a hands on approach to learn the concept.</span>
<h2><span style="font-family:'arial' , 'helvetica' , sans-serif;">
What are SQL Triggers?</span></h2>
<span style="font-family:'arial' , 'helvetica' , sans-serif;font-size:x-normal;">As per the MSDN Documentation at <a href="https://docs.microsoft.com/en-us/sql/t-sql/statements/create-trigger-transact-sql" target="_blank" rel="noopener noreferrer">CREATE TRIGGER (Transact-SQL)</a></span>

<span style="font-family:'arial' , 'helvetica' , sans-serif;font-size:x-normal;">"A trigger is a special kind of stored procedure that automatically executes when an event occurs in the database server. DML triggers execute when a user tries to modify data through a data manipulation language (DML) event. DML events are INSERT, UPDATE, or DELETE statements on a table or view. These triggers fire when any valid event is fired, regardless of whether or not any table rows are affected"</span>
<h1><span style="font-family:'arial' , 'helvetica' , sans-serif;">
Implementation</span></h1>
<span style="font-family:'arial' , 'helvetica' , sans-serif;font-size:x-normal;">The Implementation can be divided into two parts.</span>
<ol>
	<li><span style="font-family:'arial' , 'helvetica' , sans-serif;font-size:x-normal;">Creating SQL Tables, Stored Procedures and Triggers</span></li>
	<li><span style="font-family:'arial' , 'helvetica' , sans-serif;font-size:x-normal;">Creating Logic App and Connecting Logic App to the SQL Store</span></li>
</ol>
<h2><span style="font-family:'arial' , 'helvetica' , sans-serif;"> Creating SQL Tables, Stored Procedures and Triggers</span></h2>
<h3><span style="font-family:'arial' , 'helvetica' , sans-serif;"> SQL Tables</span></h3>
<span style="font-family:'arial' , 'helvetica' , sans-serif;font-size:x-normal;">A simple Employee Master Table is created which stores the information about the employee like the Employee ID, Employee Name, Salary, Reporting Manager Employee ID etc. The query for creating the simple employee table is as below.</span>
<div class="reCodeBlock" style="border:1px solid #7f9db9;overflow-y:auto;">
<div style="background-color:white;">

<span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:#006699;font-weight:bold;">CREATE</code> <code style="color:#006699;font-weight:bold;">TABLE</code> <code style="color:black;">[dbo].[Employee_Master]</code></span>

(

</div>
<div style="background-color:#f8f8f8;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[Emp_ID] </code><code style="color:#006699;font-weight:bold;">int</code> <code style="color:#006699;font-weight:bold;">Unique</code> <code style="color:grey;">NOT</code> <code style="color:grey;">NULL</code><code style="color:black;">,</code></span></span></div>
<div style="background-color:white;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[Emp_name] </code><code style="color:#006699;font-weight:bold;">varchar</code><code style="color:black;">(100) </code><code style="color:grey;">NULL</code><code style="color:black;">,</code></span></span></div>
<div style="background-color:#f8f8f8;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[Emp_Sal] </code><code style="color:#006699;font-weight:bold;">decimal</code><code style="color:black;">(10, 2) </code><code style="color:grey;">NULL</code><code style="color:black;">,</code></span></span></div>
<div style="background-color:white;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[Supervisor_ID] </code><code style="color:#006699;font-weight:bold;">int</code> <code style="color:#006699;font-weight:bold;">Unique</code> <code style="color:grey;">NOT</code> <code style="color:grey;">NULL</code></span></span></div>
<div style="background-color:#f8f8f8;"><span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:black;">)</code></span></div>
</div>
<span style="font-family:'arial' , 'helvetica' , sans-serif;">
</span>
<span style="font-family:'arial' , 'helvetica' , sans-serif;font-size:x-normal;">Another table which stores the Employee and Reporting Manager can be created using the sample query as mentioned below.</span>
<div class="reCodeBlock" style="border:1px solid #7f9db9;overflow-y:auto;">
<div style="background-color:white;"><span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:#006699;font-weight:bold;">CREATE</code> <code style="color:#006699;font-weight:bold;">TABLE</code> <code style="color:black;">[dbo].[Employee_Manager_Mapping]</code></span></div>
<div style="background-color:#f8f8f8;"><span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:black;">(</code></span></div>
<div style="background-color:white;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[Emp_ID] </code><code style="color:#006699;font-weight:bold;">int</code> <code style="color:#006699;font-weight:bold;">Unique</code> <code style="color:grey;">NOT</code> <code style="color:grey;">NULL</code><code style="color:black;">,</code></span></span></div>
<div style="background-color:#f8f8f8;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><code>    </code><span style="margin-left:12px !important;"><code style="color:black;">[Supervisor_ID] </code><code style="color:#006699;font-weight:bold;">int</code>  <code style="color:grey;">NOT</code> <code style="color:grey;">NULL</code></span></span></div>
<div style="background-color:white;"><span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:black;">)</code></span></div>
</div>
<h3> <span style="font-family:'arial' , 'helvetica' , sans-serif;">
Stored Procedure and SQL Trigger</span></h3>
<span style="font-family:'arial' , 'helvetica' , sans-serif;font-size:x-normal;">The stored procedure to insert the data into the the Employee_Master table is as follows.</span>
<div class="reCodeBlock" style="border:1px solid #7f9db9;overflow-y:auto;">
<div style="background-color:white;"><span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:#006699;font-weight:bold;">CREATE</code> <code style="color:#006699;font-weight:bold;">PROCEDURE</code> <code style="color:black;">usp_OnboardEmployee</code></span></div>
<div style="background-color:#f8f8f8;"><span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:black;">@employeeId </code><code style="color:#006699;font-weight:bold;">int</code><code style="color:black;">,</code></span></div>
<div style="background-color:white;"><span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:black;">@employeeName </code><code style="color:#006699;font-weight:bold;">varchar</code><code style="color:black;">(100),</code></span></div>
<div style="background-color:#f8f8f8;"><span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:black;">@salary </code><code style="color:#006699;font-weight:bold;">decimal</code><code style="color:black;">(10,2),</code></span></div>
<div style="background-color:white;"><span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:black;">@managerid </code><code style="color:#006699;font-weight:bold;">int</code></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:#006699;font-weight:bold;">AS</code></span></div>
<div style="background-color:#f8f8f8;"><span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:#006699;font-weight:bold;">BEGIN</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:#006699;font-weight:bold;">INSERT</code> <code style="color:#006699;font-weight:bold;">INTO</code> <code style="color:black;">[EMPLOYEE_MASTER] </code></span></div>
<div style="background-color:white;"><span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:#006699;font-weight:bold;">VALUES</code></span></div>
<div style="background-color:#f8f8f8;"><span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:black;">(@employeeId, @employeeName, @salary, @managerid)</code></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:#006699;font-weight:bold;">END</code></span></div>
</div>
<span style="font-family:'arial' , 'helvetica' , sans-serif;">
</span>
<span style="font-family:'arial' , 'helvetica' , sans-serif;font-size:x-normal;">The SQL trigger which is an After Insert Trigger which is created on the Employee_Master table and inserts the Employee Id and the Supervisor Id in the Employee_Manager_Mapping Table. The trigger is as follows.</span>
<div class="reCodeBlock" style="border:1px solid #7f9db9;overflow-y:auto;">
<div style="background-color:white;"><span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:#006699;font-weight:bold;">CREATE</code> <code style="color:#006699;font-weight:bold;">TRIGGER</code> <code style="color:black;">trgAfterEmployeeOnBoardSP </code><code style="color:#006699;font-weight:bold;">ON</code> <code style="color:black;">[dbo].[Employee_MASTER] </code></span></div>
<div style="background-color:#f8f8f8;"><span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:#006699;font-weight:bold;">FOR</code> <code style="color:#006699;font-weight:bold;">INSERT</code></span></div>
<div style="background-color:white;"><span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:#006699;font-weight:bold;">AS</code></span></div>
<div style="background-color:#f8f8f8;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><code>    </code><span style="margin-left:12px !important;"><code style="color:#006699;font-weight:bold;">declare</code> <code style="color:black;">@empid </code><code style="color:#006699;font-weight:bold;">int</code><code style="color:black;">;</code></span></span></div>
<div style="background-color:white;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><code>    </code><span style="margin-left:12px !important;"><code style="color:#006699;font-weight:bold;">declare</code> <code style="color:black;">@SupervisorId </code><code style="color:#006699;font-weight:bold;">int</code><code style="color:black;">;</code></span></span></div>
<div style="background-color:#f8f8f8;"></div>
<div style="background-color:white;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><code>    </code><span style="margin-left:12px !important;"><code style="color:#006699;font-weight:bold;">select</code> <code style="color:black;">@empid=i.Emp_ID </code><code style="color:#006699;font-weight:bold;">from</code> <code style="color:black;">inserted i; </code></span></span></div>
<div style="background-color:#f8f8f8;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><code>    </code><span style="margin-left:12px !important;"><code style="color:#006699;font-weight:bold;">select</code> <code style="color:black;">@SupervisorId = i.Supervisor_ID </code><code style="color:#006699;font-weight:bold;">from</code> <code style="color:black;">inserted i;</code></span></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><code>    </code><span style="margin-left:12px !important;"><code style="color:#006699;font-weight:bold;">INSERT</code> <code style="color:#006699;font-weight:bold;">INTO</code> <code style="color:black;">Employee_Manager_Mapping</code></span></span></div>
<div style="background-color:white;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><code>            </code><span style="margin-left:36px !important;"> </span></span></div>
<div style="background-color:#f8f8f8;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><code>    </code><span style="margin-left:12px !important;"><code style="color:#006699;font-weight:bold;">VALUES</code><code style="color:black;">(@empid, @SupervisorId);</code></span></span></div>
<div style="background-color:white;"></div>
<div style="background-color:#f8f8f8;"><span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:black;">GO</code></span></div>
</div>
<h2><span style="font-family:'arial' , 'helvetica' , sans-serif;">
Creating Logic App</span></h2>
<span style="font-family:'arial' , 'helvetica' , sans-serif;font-size:x-normal;">Before the Logic App can be created, it is necessary to set up an on premises data gateway to enable the communication between the Logic App on the cloud and the SQL data store on the premises. Refer <a href="https://docs.microsoft.com/en-us/azure/analysis-services/analysis-services-gateway" target="_blank" rel="noopener noreferrer">Connecting to on-premises data sources with Azure On-premises Data Gateway</a> . Once done following steps can be done to create the logic app.</span>
<div class="separator" style="clear:both;text-align:center;"></div>
<ol>
	<li><span style="font-family:'arial' , 'helvetica' , sans-serif;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><span style="font-size:x-normal;"> Select the Logic App from the Azure Market Place</span></span></span><a style="font-family:'Times New Roman';margin-left:1em;margin-right:1em;text-align:center;" href="https://theserverlessspirit.files.wordpress.com/2018/07/7f022-8507-selectlogicapp-550x0.jpg"><img src="https://theserverlessspirit.files.wordpress.com/2018/07/7f022-8507-selectlogicapp-550x0.jpg?w=280" width="372" height="400" border="0" /></a>
<span style="white-space:pre;">
</span></li>
	<li><span style="font-family:'arial' , 'helvetica' , sans-serif;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><span style="font-size:x-normal;">Select the Details like subscription, Resource Group, Location and then ht the create blade on the blade. Refer below screenshot.</span></span></span><a style="font-family:'Times New Roman';margin-left:1em;margin-right:1em;text-align:center;" href="https://theserverlessspirit.files.wordpress.com/2018/07/7e2c0-2287-createla-550x0.jpg"><img src="https://theserverlessspirit.files.wordpress.com/2018/07/7e2c0-2287-createla-550x0.jpg?w=157" width="208" height="400" border="0" /></a>&nbsp;

&nbsp;</li>
	<li><span style="font-family:'arial' , 'helvetica' , sans-serif;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><span style="font-size:x-normal;"> Select the HTTP Trigger template from the list of available templates as shown below.</span></span></span><a style="font-family:'Times New Roman';margin-left:1em;margin-right:1em;text-align:center;" href="https://theserverlessspirit.files.wordpress.com/2018/07/ee47f-6303-selecttrigger-550x0.jpg"><img src="https://theserverlessspirit.files.wordpress.com/2018/07/ee47f-6303-selecttrigger-550x0.jpg" border="0" /></a></li>
	<li><span style="font-family:'arial' , 'helvetica' , sans-serif;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><span style="font-size:x-normal;">In order to define the input message, use the Upload Sample Payload To Generate Schema (highlighted in yellow), Paste following sample message which can be used to generate the input schema. After that Click on Advanced Options and select the method as POST. </span>
</span></span>
<div class="reCodeBlock" style="border:1px solid #7f9db9;font-size:12.1104px;overflow-y:auto;">
<div style="background-color:white;"><span style="margin-left:0 !important;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:blue;">"EmployeeId"</code> <code style="color:black;">: </code><code style="color:#009900;">123</code><code style="color:black;">,</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:blue;">"EmployeeName"</code> <code style="color:black;">: </code><code style="color:blue;">"Mandar Dharmadhikari"</code><code style="color:black;">,</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:12px !important;"><code style="color:blue;">"Salary"</code> <code style="color:black;">: </code><code style="color:#009900;">10000.20</code><code style="color:black;">,</code></span></div>
<div style="background-color:white;"><code>    </code><span style="margin-left:12px !important;"><code style="color:blue;">"SupervisorId"</code> <code style="color:black;">: </code><code style="color:#009900;">345</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:black;">}</code></span></div>
</div>
<span style="font-size:x-normal;">
Refer following sample screenshot for the configuration of the HTTP Trigger</span>

<a style="font-family:'Times New Roman';margin-left:1em;margin-right:1em;text-align:center;" href="https://theserverlessspirit.files.wordpress.com/2018/07/0805d-4718-httptriggerconfig-550x0.jpg"><img src="https://theserverlessspirit.files.wordpress.com/2018/07/0805d-4718-httptriggerconfig-550x0.jpg" border="0" /></a>

&nbsp;</li>
	<li><span style="font-family:'arial' , 'helvetica' , sans-serif;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><span style="font-size:x-normal;">
Once the HTTP POST request is received , the On Premises SQL datastore can be updated with the employee details. The Action to execute the stored procedure from the list of SQL action is to be selected.
Refer sample screen shot.
</span></span></span><span style="font-size:12.1104px;white-space:pre;">
<a style="font-family:'Times New Roman';font-size:medium;margin-left:1em;margin-right:1em;text-align:center;white-space:normal;" href="https://theserverlessspirit.files.wordpress.com/2018/07/df0ae-5633-sqlselect-550x0.jpg"><img src="https://theserverlessspirit.files.wordpress.com/2018/07/df0ae-5633-sqlselect-550x0.jpg" border="0" /></a></span></li>
	<li><span style="font-family:'arial' , 'helvetica' , sans-serif;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><span style="font-size:x-normal;">Create the connection to the On Premise SQL server data store using the on premise data gateway. Refer following sample screen shot</span></span></span><span style="font-size:12.1104px;white-space:pre;"><a style="font-family:'Times New Roman';font-size:medium;margin-left:1em;margin-right:1em;text-align:center;white-space:normal;" href="https://theserverlessspirit.files.wordpress.com/2018/07/43e6c-0842-createsqlconn-550x0.jpg"><img src="https://theserverlessspirit.files.wordpress.com/2018/07/43e6c-0842-createsqlconn-550x0.jpg" border="0" /></a>
</span></li>
	<li><span style="font-family:'arial' , 'helvetica' , sans-serif;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><span style="font-size:x-normal;">Select the SQL stored procedure usp_onboardEmployee which was created earlier and use the fields that are recieved from the HTTP Post call. Refer a sample screen shot below.</span></span></span></li>
	<li><span style="background-color:white;color:#2a2a2a;font-family:'arial' , 'helvetica' , sans-serif;">Save the Logic App. This will create the end point that can be copied from the HTTP trigger and used to test the solution developed.</span></li>
</ol>
<h1><span style="font-family:'arial' , 'helvetica' , sans-serif;">
Testing</span></h1>
<span style="font-family:'arial' , 'helvetica' , sans-serif;font-size:x-normal;">A utility like POSTMAN or SOAP UI can be used to test the Logic App created above. The sample message used while creating the json payload for the HTTP trigger is used for testing. The payload is as follows.</span>
<div class="reCodeBlock" style="border:1px solid #7f9db9;overflow-y:auto;">
<div style="background-color:white;"><span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:black;">{</code></span></div>
<div style="background-color:#f8f8f8;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><code>    </code><span style="margin-left:12px !important;"><code style="color:blue;">"EmployeeId"</code> <code style="color:black;">: </code><code style="color:#009900;">123</code><code style="color:black;">,</code></span></span></div>
<div style="background-color:white;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><code>    </code><span style="margin-left:12px !important;"><code style="color:blue;">"EmployeeName"</code> <code style="color:black;">: </code><code style="color:blue;">"Mandar Dharmadhikari"</code><code style="color:black;">,</code></span></span></div>
<div style="background-color:#f8f8f8;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><code>    </code><span style="margin-left:12px !important;"><code style="color:blue;">"Salary"</code> <code style="color:black;">: </code><code style="color:#009900;">10000.20</code><code style="color:black;">,</code></span></span></div>
<div style="background-color:white;"><span style="font-family:'arial' , 'helvetica' , sans-serif;"><code>    </code><span style="margin-left:12px !important;"><code style="color:blue;">"SupervisorId"</code> <code style="color:black;">: </code><code style="color:#009900;">345</code></span></span></div>
<div style="background-color:#f8f8f8;"><span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:black;">}</code></span></div>
</div>
<span style="font-family:'arial' , 'helvetica' , sans-serif;">
</span>
<span style="font-family:'arial' , 'helvetica' , sans-serif;font-size:x-normal;">The url copied from the HTTP Trigger is the one which should be consumed in the POSTMAN. The content-type header for the request should be set to application/json. Following is the screen shot of the request that is sent to the Logic App.
</span>
<div class="separator" style="clear:both;text-align:center;"><a style="margin-left:1em;margin-right:1em;" href="https://theserverlessspirit.files.wordpress.com/2018/07/bbef2-1727-postman-550x0.jpg"><img src="https://theserverlessspirit.files.wordpress.com/2018/07/bbef2-1727-postman-550x0.jpg" border="0" /></a></div>
<span style="font-family:'arial' , 'helvetica' , sans-serif;">
</span>
<span style="font-family:'arial' , 'helvetica' , sans-serif;">
</span>
<span style="font-family:'arial' , 'helvetica' , sans-serif;font-size:x-normal;">Following screenshot confirms the successful execution of the Flow.</span>
<div class="separator" style="clear:both;text-align:center;"><span style="font-family:'arial' , 'helvetica' , sans-serif;white-space:pre;"><a style="margin-left:1em;margin-right:1em;" href="https://theserverlessspirit.files.wordpress.com/2018/07/7b695-2818-logicapprun-550x0.jpg"><img src="https://theserverlessspirit.files.wordpress.com/2018/07/7b695-2818-logicapprun-550x0.jpg" border="0" /></a> </span></div>
<div class="separator" style="clear:both;text-align:center;"><span style="font-family:'arial' , 'helvetica' , sans-serif;white-space:pre;"> </span></div>
<span style="font-family:'arial' , 'helvetica' , sans-serif;font-size:x-normal;">When Following query is run on the on premises data store, the results returned as shown in the screen shot following the query.</span>

<span style="font-family:'arial' , 'helvetica' , sans-serif;font-size:x-normal;">
</span>
<div class="reCodeBlock" style="border:1px solid #7f9db9;overflow-y:auto;">
<div style="background-color:white;"><span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:#006699;font-weight:bold;">Select</code> <code style="color:black;">* </code><code style="color:#006699;font-weight:bold;">from</code> <code style="color:black;">Employee_Master</code></span></div>
<div style="background-color:#f8f8f8;">

<span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:#006699;font-weight:bold;">Select</code> <code style="color:black;">* </code><code style="color:#006699;font-weight:bold;">from</code> <code style="color:black;">Employee_Manager_Mapping</code></span>

<span style="font-family:'arial' , 'helvetica' , sans-serif;margin-left:0;"><code style="color:black;">
</code></span>

</div>
</div>
&nbsp;
<div class="separator" style="clear:both;text-align:center;"><a style="margin-left:1em;margin-right:1em;" href="https://theserverlessspirit.files.wordpress.com/2018/07/8920d-4456-sqlresult-550x0.jpg"><img src="https://theserverlessspirit.files.wordpress.com/2018/07/8920d-4456-sqlresult-550x0.jpg" border="0" /></a></div>
<span style="font-family:'arial' , 'helvetica' , sans-serif;">  </span>
<h1><span style="font-family:'arial' , 'helvetica' , sans-serif;">
Conclusion</span></h1>
<span style="font-family:'arial' , 'helvetica' , sans-serif;font-size:x-normal;">As evident from the testing results, a single action from logic app can be used in conjunction with the SQL trigger when similar data is to be updated across multiple sql tables in the data store.</span>
<h1><span style="font-family:'arial' , 'helvetica' , sans-serif;">
References </span></h1>
<span style="font-family:'arial' , 'helvetica' , sans-serif;font-size:x-normal;">Content from following articles was referred while writing the article.</span>
<ol>
	<li><span style="font-family:'arial' , 'helvetica' , sans-serif;font-size:x-normal;"><a href="https://docs.microsoft.com/en-us/azure/analysis-services/analysis-services-gateway" target="_blank" rel="noopener noreferrer">Connecting to on-premises data sources with Azure On-premises Data Gateway</a> </span></li>
	<li><span style="font-family:'arial' , 'helvetica' , sans-serif;font-size:x-normal;"><a href="https://docs.microsoft.com/en-us/sql/t-sql/statements/create-trigger-transact-sql" target="_blank" rel="noopener noreferrer">CREATE TRIGGER (Transact-SQL)</a> </span></li>
</ol>
&nbsp;

</div>
&nbsp;