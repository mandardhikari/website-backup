---
ID: 113
post_title: 'Could not load file or assembly &#8216;SSOSettingsFileReader, Version=1.0.0.0, Culture=neutral, PublicKeyToken=b9585ffc9d454e4a&#8217; or one of its dependencies. The located assembly&#8217;s manifest definition does not match the assembly reference. (Exception from HRESULT: 0x80131040)'
author: Mandar Dharmadhikari
post_excerpt: ""
layout: post
permalink: >
  https://theabodeofcode.com/could-not-load-file-or-assembly-ssosettingsfilereader-version1-0-0-0-cultureneutral-publickeytokenb9585ffc9d454e4a-or-one-of-its-dependencies-the-located-assemblys-manifest-definition-doe/
published: true
post_date: 2018-08-30 05:43:56
---
Today during my work schedule, I was approached by one of the Front End users and the y complained that BizTalk services were returning faults to them. Upon checking up the logs for the BizTalk application, I came across following error.

<em>"Could not load file or assembly 'SSOSettingsFileReader, Version=1.0.0.0, Culture=neutral, PublicKeyToken=b9585ffc9d454e4a' or one of its dependencies. The located assembly's manifest definition does not match the assembly reference. (Exception from HRESULT: 0x80131040)</em>
<em>The located assembly's manifest definition does not match the assembly reference. (Exception from HRESULT: 0x80131040)"</em>.

My first reaction to this was: "<strong>This is not possible. The assembly has to be in the GAC!!</strong>".

so I went to the GAC located at C:\Windows\Microsoft.NET\assembly\GAC_MSIL\SSOSettingsFileReader\v4.0_1.0.0.0__b9585ffc9d454e4a, the assembly was there.

So why does this error started popping up all of a sudden?

This assembly "<strong>SSOSettingsFileReader.dll</strong>" is the assembly used by BTDF to read the values from the <strong>SSOSettingsFile</strong>. This assembly gets deployed every time we do a deployment of  BizTalk application which reads values from SSOSettingsFile. This is generally done by a BTDF deployment and the timestamp in GAC gets updated with the latest deployment.

In my case yesterday I had deployed an application which read values using above assembly. My BTDF file was tailored to restart only the Host Instances for that particular application as it had following entry in app.btdfproj file
<div class="reCodeBlock" style="border:1px solid #7f9db9;overflow-y:auto;">
<div style="background-color:#ffffff;"><span style="margin-left:0 !important;"><code style="color:#000000;">&lt;</code><code style="color:#006699;font-weight:bold;">ItemGroup</code><code style="color:#000000;">&gt;</code></span></div>
<div style="background-color:#f8f8f8;"><code>  </code><span style="margin-left:8px !important;"><code style="color:#000000;">&lt;</code><code style="color:#006699;font-weight:bold;">BizTalkHosts</code> <code style="color:#808080;">Include</code><code style="color:#000000;">=</code><code style="color:blue;">"BizTalkServerABCApplicationHost;BizTalkServerABCSendHost"</code><code style="color:#000000;">/&gt;</code></span></div>
<div style="background-color:#ffffff;"><span style="margin-left:0 !important;"><code style="color:#000000;">&lt;/</code><code style="color:#006699;font-weight:bold;">ItemGroup</code><code style="color:#000000;">&gt;</code></span></div>
</div>
&nbsp;

This forced the BTDF to restart only two host instances tied to Hosts mentioned above. Now other applications which were using the same file were not aware of this un deployment and deployment of the SSOSettingsFileReader.dll assembly.

Now when any request came to the other applications referring this assembly, they started raising exception mentioned above.
<h1>Resolution</h1>
The solution to this problem is very simple. A host instance restart helped me solve the problem.

I hope this simple solution helps some one who faces this issue as the error logged in the event log is bit misguiding.

That makes me wonder if the BTDF should be forced to restart only particular host instances only or it should straight away restart all the Host Instances??

I would love to get some opinions on this point in the chat below.