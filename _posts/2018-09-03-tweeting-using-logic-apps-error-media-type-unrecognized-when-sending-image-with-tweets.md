---
ID: 138
post_title: 'Tweeting Using Logic Apps: Error &#8220;media type unrecognized&#8221; when sending image with Tweets'
author: Mandar Dharmadhikari
post_excerpt: ""
layout: post
permalink: >
  https://theabodeofcode.com/tweeting-using-logic-apps-error-media-type-unrecognized-when-sending-image-with-tweets/
published: true
post_date: 2018-09-03 02:30:03
---
<h1>BackGround</h1>
Today while going through the Logic Apps forums , I found a question where the Original Poster had problems sending an image with the tweet text by using the twitter api. The error that the Logic App run time threw back was similar to
<div class="reCodeBlock" style="border:1px solid #7f9db9;overflow-y:auto;">
<div style="background-color:#ffffff;"><span style="margin-left:0 !important;"><code style="color:#000000;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>  </code><span style="margin-left:8px !important;"><code style="color:blue;">"status"</code><code style="color:#000000;">: 400,</code></span></div>
<div style="background-color:#ffffff;"><code>  </code><span style="margin-left:8px !important;"><code style="color:blue;">"message"</code><code style="color:#000000;">: </code><code style="color:blue;">"media type unrecognized.\r\nclientRequestId: 5c3cf7d0-c018-494d-bc74-f4d88edc040d\r\nserviceRequestId: 392220ba2115f611f7789f451003bdc4"</code><code style="color:#000000;">,</code></span></div>
<div style="background-color:#f8f8f8;"><code>  </code><span style="margin-left:8px !important;"><code style="color:blue;">"source"</code><code style="color:#000000;">: </code><code style="color:blue;">"twitter-wi.azconn-wi.p.azurewebsites.net"</code></span></div>
<div style="background-color:#ffffff;"><span style="margin-left:0 !important;"><code style="color:#000000;">}</code></span></div>
</div>
in this post I have tried to dissect and solve the issue.
<h1>Constructing The Logic App1</h1>
The logic app is simple. It accepts the tweet text and the image text in a base64 string. I have used the HTTP request trigger to fire up the Logic App. The message sent to the logic app looks something like following
<div class="reCodeBlock" style="border:1px solid #7f9db9;overflow-y:auto;">
<div style="background-color:#ffffff;"><span style="margin-left:0 !important;"><code style="color:#000000;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:16px !important;"><code style="color:blue;">"tweetText"</code> <code style="color:#000000;">: </code><code style="color:blue;">"Sample Tweet With Image Fired From Logic App"</code><code style="color:#000000;">,</code></span></div>
<div style="background-color:#ffffff;"><code>    </code><span style="margin-left:16px !important;"><code style="color:blue;">"imageData"</code> <code style="color:#000000;">: </code><code style="color:blue;">"base64stringHere"</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#000000;">}</code></span></div>
</div>
I then added the Twitter "Post a tweet " action to my logic app and passed the data from the incoming message directly to the action.

My logic app looks like below.

<img class="alignnone size-full wp-image-139" src="https://theserverlessspirit.files.wordpress.com/2018/09/logicappdesign1.jpg" alt="LogicAPpDesign1.JPG" width="1589" height="680" />
<h1>Testing the Logic App1</h1>
I tested the logic app using POSTMAN and I got following error message, when I examined the LogicApp Overview.

<img class="alignnone size-full wp-image-140" src="https://theserverlessspirit.files.wordpress.com/2018/09/test1.jpg" alt="Test1.JPG" width="1489" height="969" />

Same "media Type is unrecognized" error as posted in the forums. So I was able to replicate the issue.
<h1>Constructing the Logic App2</h1>
Let us look at the documentation of the twitter connector on the connectors page at <a href="https://docs.microsoft.com/en-us/connectors/twitterconnector/" target="_blank" rel="noopener noreferrer">Twitter Connector</a> it says that to post a tweet it has two paramters
<ol>
	<li>tweetText which is a strings</li>
	<li>Media which is binary</li>
</ol>
Once we look at this the solution comes to us easily. The reason the logic apps action failed was we were passing the base64 encoded string instead of a binary stream.  So we need to convert the base64 string to the binary and we should be good to go. So I made following change to the logic app.

<img class="alignnone size-full wp-image-141" src="https://theserverlessspirit.files.wordpress.com/2018/09/logicapp2.jpg" alt="LogicAPp2.JPG" width="1024" height="525" /> So my modified Logic App looks like

<img class="alignnone size-full wp-image-142" src="https://theserverlessspirit.files.wordpress.com/2018/09/logicapp21.jpg" alt="LogicApp21.JPG" width="1256" height="444" />

The code behind the Twitter connector is as follows.
<div class="reCodeBlock" style="border:1px solid #7f9db9;overflow-y:auto;">
<div style="background-color:#ffffff;"><span style="margin-left:0 !important;"><code style="color:#000000;">{</code></span></div>
<div style="background-color:#f8f8f8;"><code>    </code><span style="margin-left:16px !important;"><code style="color:blue;">"inputs"</code><code style="color:#000000;">: {</code></span></div>
<div style="background-color:#ffffff;"><code>        </code><span style="margin-left:32px !important;"><code style="color:blue;">"host"</code><code style="color:#000000;">: {</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:48px !important;"><code style="color:blue;">"connection"</code><code style="color:#000000;">: {</code></span></div>
<div style="background-color:#ffffff;"><code>                </code><span style="margin-left:64px !important;"><code style="color:blue;">"name"</code><code style="color:#000000;">: </code><code style="color:blue;">"@parameters('$connections')['twitter']['connectionId']"</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:48px !important;"><code style="color:#000000;">}</code></span></div>
<div style="background-color:#ffffff;"><code>        </code><span style="margin-left:32px !important;"><code style="color:#000000;">},</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:32px !important;"><code style="color:blue;">"method"</code><code style="color:#000000;">: </code><code style="color:blue;">"post"</code><code style="color:#000000;">,</code></span></div>
<div style="background-color:#ffffff;"><code>        </code><span style="margin-left:32px !important;"><code style="color:blue;">"body"</code><code style="color:#000000;">: </code><code style="color:blue;">"@base64ToBinary(triggerBody()?['imageData'])"</code><code style="color:#000000;">,</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:32px !important;"><code style="color:blue;">"path"</code><code style="color:#000000;">: </code><code style="color:blue;">"/posttweet"</code><code style="color:#000000;">,</code></span></div>
<div style="background-color:#ffffff;"><code>        </code><span style="margin-left:32px !important;"><code style="color:blue;">"queries"</code><code style="color:#000000;">: {</code></span></div>
<div style="background-color:#f8f8f8;"><code>            </code><span style="margin-left:48px !important;"><code style="color:blue;">"tweetText"</code><code style="color:#000000;">: </code><code style="color:blue;">"@triggerBody()?['tweetText']"</code></span></div>
<div style="background-color:#ffffff;"><code>        </code><span style="margin-left:32px !important;"><code style="color:#000000;">},</code></span></div>
<div style="background-color:#f8f8f8;"><code>        </code><span style="margin-left:32px !important;"><code style="color:blue;">"authentication"</code><code style="color:#000000;">: </code><code style="color:blue;">"@parameters('$authentication')"</code></span></div>
<div style="background-color:#ffffff;"><code>    </code><span style="margin-left:16px !important;"><code style="color:#000000;">}</code></span></div>
<div style="background-color:#f8f8f8;"><span style="margin-left:0 !important;"><code style="color:#000000;">}</code></span></div>
</div>
So the base64ToBinary Converts the base64 to bainary data
<h1>Testing Logic App 2</h1>
I tested the modified logic app with the same data and it ran successfully.

<img class="alignnone size-full wp-image-143" src="https://theserverlessspirit.files.wordpress.com/2018/09/successfultweet.jpg" alt="SuccessfulTweet.JPG" width="1541" height="610" />

A look at twitter page

<img class="alignnone size-full wp-image-144" src="https://theserverlessspirit.files.wordpress.com/2018/09/successfultweet2.jpg" alt="SuccessfulTweet2.JPG" width="582" height="278" />
<h1>Conclusion</h1>
It is necessary to manage the content in the request and parse it to binary format while posting an image to Twitter using Logic App.

&nbsp;