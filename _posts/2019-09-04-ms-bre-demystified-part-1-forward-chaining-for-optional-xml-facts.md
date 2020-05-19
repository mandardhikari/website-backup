---
ID: 324
post_title: 'MS BRE  Demystified &#8211; Part 1 Forward Chaining for optional XML Facts'
author: Mandar Dharmadhikari
post_excerpt: ""
layout: post
permalink: >
  https://theabodeofcode.com/ms-bre-demystified-part-1-forward-chaining-for-optional-xml-facts/
published: true
post_date: 2019-09-04 10:52:12
---
<!-- wp:paragraph -->
<p>For past few months I have been working on a big project in my organization where we are using Microsoft Business Rules engine for the conditional evaluation of incoming payloads. There are few lessons that I have learnt along the way and I will present them as a part of a blog series centered around the Business Rules Engine.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>I want to write this series as I find that the blog posts on BRE are really scarce and hard to find. I will document my learning in this series. Before begin, I would like to point out that the blog from Charles Young is a really good reading resource if you want to understand the nuances of BRE . The blog can be found at <a href="http://geekswithblogs.net/cyoung/Default.aspx" target="_blank" rel="noreferrer noopener" aria-label=" (opens in a new tab)">http://geekswithblogs.net/cyoung/Default.aspx</a> . So without ado let us dive right into it.</p>
<!-- /wp:paragraph -->

<!-- wp:heading -->
<h2>A little about MS BRE</h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>Microsoft Business Rules (MS BRE ) is  a .Net compliant inference engine which helps developers write highly readable and semantically rich rules on .Net objects, XMLs and database tables. MS BRE works on the principle of production rule system.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>A production rule system can be thought of as a series of rules each of which contains Conditions and Actions.  When the conditions are evaluated to true, then the set of steps mentioned in the actions are executed. Now, one can think, why do I need to use the rule engine? I can code the steps as a complex  branching logic containing if- else  and switch case statements. Whilst that is true, this if-else logic poses two problems when the number of rules to be implemented is very big</p>
<!-- /wp:paragraph -->

<!-- wp:list {"ordered":true} -->
<ol><li>The bigger the number of rules, the complex the branching logic</li><li>The branching logic enforces a strict "<strong>sequential</strong>" flow of information from one statement to another</li></ol>
<!-- /wp:list -->

<!-- wp:paragraph -->
<p>If this is implemented as a part of a production rule system like MS BRE, then the developer only defines the rules as a collection of conditions and the actions. The developer need not worry about the  order in which the rules get executed (though we can enforce some sense of order, more on it in future post). The rule engine at the time of the execution of the entire rules, creates a tree structure based upon the Rete(spelled "Ree-Tee") algorithm( <a href="https://en.wikipedia.org/wiki/Rete_algorithm">https://en.wikipedia.org/wiki/Rete_algorithm</a> ). The data that is fed to the rule engine then passes through related nodes in this tree and produces the output at the end.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>For MS BRE the processing of the message against the defined rules can be explained at a very high level as following steps</p>
<!-- /wp:paragraph -->

<!-- wp:list {"ordered":true} -->
<ol><li>At the time of the execution of the policy, the rule engine create a tree structure based upon the conditions provided in the rules. This tree is based on Rete algorithm</li><li>Once the data(called facts )is fed to the engine, it evaluates the pertinent facts against the conditions in the constructed tree</li><li>Once the fact passes the condition, it advances in the tree</li><li>Once a fact passes all the conditions, the rules where all conditions are satisfied are updated in the agenda of the rule engine</li><li>Once the agenda update is completed, the rules which were added to the agenda are fired and the actions defined in these rules are executed.</li><li>Facts are removed from the rule engine working memory</li></ol>
<!-- /wp:list -->

<!-- wp:paragraph -->
<p>In order to learn more about how the MS BRE works in detail, I would urge you to read the MSDN documentation at <a href="https://docs.microsoft.com/en-us/biztalk/core/rule-engine" target="_blank" rel="noreferrer noopener" aria-label=" (opens in a new tab)">https://docs.microsoft.com/en-us/biztalk/core/rule-engine</a></p>
<!-- /wp:paragraph -->

<!-- wp:heading -->
<h2>What is Forward Chaining</h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>In layman's term, forward chaining can be defined as "<strong><em>Effects due to execution of one rule, changes the evaluation of another rule</em></strong>". Though this is a very high level and incomplete definition of forward chaining, this will serve us for the time being.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Consider following simple rule system</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Rule 1 : IF A == 10, Then B= 20</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Rule 2: IF B == 10, Then A = 10</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Let us say we provide the rule engine  with following values A= 0 and B = 10.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Since A = 0, Rule 1 was never fired,</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Since B = 10, Rule 2 was fired and the rule engine set the value of A = 10. </p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>In this case ideally, as the Rule 2 sets the value of A to 10, Rule 1 should be fired.  Meaning, the result of execution of action (setting A = 10)  of Rule 2 should trigger the execution of Action(Setting B = 20) of Rule 1. This is forward chaining at a very level.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>In case of MS BRE, forward chaining can achieved using the update and assert function provided with the engine. As the name suggest, update is used to update the value of the existing data(facts) that is already loaded in the working memory of the rule engine. Assert is used to load new data into the working memory of the rule engine. So let us apply this knowledge to the Rule system mentioned above</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Rule 1: IF A == 10, Then B = 20,</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Rule 2: IF B ==10, Then A = 10 and Update(A)</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Again let us provide A = 0, B = 10 to the rule engine.  The rule engine working memory now contains the value of A = 0 and B = 10.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Since A = 0, Rule 1 is not fired.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Since B = 10 , Rule 2 gets fired and sets A = 10. At the same time, using update function on A, the value of A is updated to 10 in the working memory of the rule engine.  The rule engine now has A = 10, B = 10 in the working memory. This update triggers the rule engine to reevaluate all the rules which had conditions checking values of A(in this case Rule 1). So conditions in Rule 1 get evaluated to true and we get value of B = 20. Since there are no more update or assert encountered in the actions of the rules, the rule engine finishes the execution and final value we have are A = 10, B = 20</p>
<!-- /wp:paragraph -->

<!-- wp:heading -->
<h2>XML Facts and MS BRE</h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>Let us now understand how  MS BRE deals with a XML bases facts . Consider a simple XSD as shown below.</p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":326} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/08/Sample-XSD-1024x283.jpg" alt="" class="wp-image-326"/></figure>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p>In XML world, the standard way to access and modify the values of the elements is to use the XPATH query language.  In a traditional sense if I want to read value of fiedls A in the Req, I would access the element using Xpath</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/6a73cc668352b4f03fa921980baa51fe.js"></script></p>

<!-- wp:paragraph -->
<p>MS BRE also works with the XML facts using XPATHs but slightly in a different way. Following image shows the XPATH components generated for the Field A in the Req.</p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":332} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/08/XPath-1024x482.jpg" alt="" class="wp-image-332"/></figure>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p>BRE Generated XPath Selector and XPath fields to access the value of filed A. If we look carefully, the Xpath Selector + Xpath Field gives us the complete xpath for the field A. The BRE does it with following intentions, when the XML facts are asserted into the working memory, BRE will assert all the facts that it finds under Req to the rules defined in our system. Thus to access all the values of fact A that are inserted, the BRE uses the XPath Field to narrow down. Charles Young has explained this phenomenon in a great detail at <a rel="noreferrer noopener" aria-label=" (opens in a new tab)" href="http://geekswithblogs.net/cyoung/archive/2006/09/02/90102.aspx" target="_blank">http://geekswithblogs.net/cyoung/archive/2006/09/02/90102.aspx</a> I strongly urge you to read the blog. I am just scratching the surface here.</p>
<!-- /wp:paragraph -->

<!-- wp:heading -->
<h2>Forward Chaining</h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>Now let us put our new found knowledge to test by defining the rule System we discussed while understanding the Forward chaining above. The rules are shown as below.</p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":333} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/08/Rule1-1024x514.jpg" alt="" class="wp-image-333"/></figure>
<!-- /wp:image -->

<!-- wp:image {"id":334} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/08/Rule2-1024x555.jpg" alt="" class="wp-image-334"/></figure>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p>I am going to assert following XML message to the rule engine to test the rule system that we created.</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/8312c9241ac796371ef5c125e24f529d.js"></script></p>

<!-- wp:paragraph -->
<p>BRE produces following out put</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/9e737de3120091af0233e4791efe07ae.js"></script></p>

<!-- wp:paragraph -->
<p><strong>YAY!! Forward Chaining works!!</strong></p>
<!-- /wp:paragraph -->

<!-- wp:heading -->
<h2>Houston We Have a Problem</h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>All works good when both A and B are present in the input XML message that we fed to the rule engine. Let see what happens when we feed following XML to the rules engine</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/0baf613e0cbc0944189183d90bc8db20.js"></script></p>

<!-- wp:paragraph -->
<p>BRE crashes and gives following error</p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":347} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/08/MissingFieldA-1024x472.jpg" alt="" class="wp-image-347"/></figure>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p>Why did this happen?? Let us think back to how the BRE  uses XPaths , by breaking them into the XPathSelector and XPathField. Following images shows how the Rule 1 reads the field A from the request</p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":348} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/08/XPathFieldA-1024x345.jpg" alt="" class="wp-image-348"/></figure>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p>It is expecting the  field A which is read using XPath field be present in the XML message always. Hence when the filed is missing from the input, BRE engine threw an error.</p>
<!-- /wp:paragraph -->

<!-- wp:heading -->
<h2>Fixing the Problem</h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>BRE comes with an in built predicate "exist" which can be used on the XML based facts. As the name suggests, the exist predicate checks the existence of the xpath in the document that was asserted to the BRE.   </p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Let us drag the xpath of the field A from Facts Explorer and  into our rule.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Following are our Rule 1 and Rule 2 respectively</p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":349} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/08/Exist1-1024x339.jpg" alt="" class="wp-image-349"/></figure>
<!-- /wp:image -->

<!-- wp:image {"id":351} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/08/Exist2-1024x349.jpg" alt="" class="wp-image-351"/></figure>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p>Let us again feed following message to the rule engine.</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/0baf613e0cbc0944189183d90bc8db20.js"></script></p>

<!-- wp:paragraph -->
<p>BRE gives following output to us</p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":356} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/09/XpathExistDefaultXPathAdded-1024x502.jpg" alt="" class="wp-image-356"/></figure>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p>Confusing isn't it?? Why did the exist predicate not work with the xpath that we dragged and dropped from the schema?? If we examine the condition in the Rule 1 shared above, we see that the Xpath we are checking in the exist predicate is </p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/6a73cc668352b4f03fa921980baa51fe.js"></script></p>
<p><!-- wp:paragraph>
While we are equating the value of field A to 20 using the XPath
<!--/wp:paragraph>
</p>









--></p>

<!-- wp:paragraph -->
<p>But the XPath with which we are reading and equating the value of field A to 20 is</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/9d6015047998dd2a9d18a0a66e2c8462.js"></script></p>

<!-- wp:paragraph -->
<p>It is clear now that both of the <strong><em>XPaths are different and hence will be handled as two different nodes in the tree structure that the Rule engine will create</em></strong>.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Now as we understand this, what we need to do is modify the Xpath in the exist predicate. The modified Rules look as follows.</p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":366} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/09/ModifiedRule1-2-1024x452.jpg" alt="" class="wp-image-366"/></figure>
<!-- /wp:image -->

<!-- wp:image {"id":367} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/09/ModifiedRule2-1024x369.jpg" alt="" class="wp-image-367"/></figure>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p>Now each of the condition in the Rule is pointing to the same Xpath. Hence, <strong><em>when the rule engine transforms these rules into the tree structure, it will create only one node for the Field A and one node For Field B</em></strong>.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Let us again feed following message to the Rule Engine and see what happens</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/0baf613e0cbc0944189183d90bc8db20.js"></script></p>

<!-- wp:paragraph -->
<p>Rule engine produces below output</p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":371} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/09/WorkingXPath-1024x489.jpg" alt="" class="wp-image-371"/></figure>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p><strong><em>Rule Engine did not produce any error and handled the scenario gracefully. No rules were fired but the engine did not exception out!!</em></strong></p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>This approach thus provides us a fail safe way to implement forward chaining on the optional elements in our xml payloads.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>This wraps up my first post in the BRE series. In next post we will take a look at how we can  execute the BRE policy from .NET code and how we can use the log4net to track the execution of the policy whenever required.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Till then have a good time!! Please share feedback </p>
<!-- /wp:paragraph -->