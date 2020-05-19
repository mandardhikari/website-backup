---
ID: 377
post_title: >
  MS BRE Demystified – Part 2 Custom
  Tracking in BRE Policies
author: Mandar Dharmadhikari
post_excerpt: 'In this post, I will talk about how we can implement correlated tracking while executing BRE Policies from C# code.'
layout: post
permalink: >
  https://theabodeofcode.com/ms-bre-demystified-part-2-custom-tracking-in-bre-policies/
published: true
post_date: 2019-09-10 15:05:12
---
<!-- wp:paragraph -->
<p>This is my second post in MS BRE Demystified series. If you have not read the previous post, you can read it at <a href="https://theabodeofcode.com/ms-bre-demystified-part-1-forward-chaining-for-optional-xml-facts/" target="_blank" rel="noreferrer noopener" aria-label=" (opens in a new tab)">https://theabodeofcode.com/index.php/2019/09/04/ms-bre-demystified-part-1-forward-chaining-for-optional-xml-facts/</a> . The post talks about forward chaining for optional xml based facts.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>In this post, I will talk about how we can implement correlated tracking while executing BRE Policies from C# code. </p>
<!-- /wp:paragraph -->

<!-- wp:heading -->
<h2>A Little Background</h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>We have two options of executing a MS BRE Policy.</p>
<!-- /wp:paragraph -->

<!-- wp:list {"ordered":true} -->
<ol><li>Execute the policy using the calls rules shape in BizTalk orchestration </li><li>Execute the policy from a function inside a C# class.</li></ol>
<!-- /wp:list -->

<!-- wp:paragraph -->
<p>In both the cases we have an ability to track the execution of rules and other activities that the Rule Engine go through.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>In case when the policy is executed using a Calls Rules shape,  just enabling the policy tracking in BizTalk admin console is sufficient. We can do it as shown below.</p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":378} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/09/TrackPolicy.jpg" alt="" class="wp-image-378"/></figure>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p>After this the policy execution will be available in the tracked events. For more details on this refer <a rel="noreferrer noopener" aria-label=" (opens in a new tab)" href="https://docs.microsoft.com/en-us/biztalk/core/walkthrough-tracking-policy-execution" target="_blank">https://docs.microsoft.com/en-us/biztalk/core/walkthrough-tracking-policy-execution</a> </p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>We can also track the  events when testing the policy in Business Rules Composer. Following screen shot shows a sample of  events recorded.</p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":379} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/09/RulesComposer-1024x558.jpg" alt="" class="wp-image-379"/></figure>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p>Let us now see how we can track execution events when the policy is called from a function in C# class</p>
<!-- /wp:paragraph -->

<!-- wp:heading -->
<h2>Executing policy from a Function</h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>MS BRE gives us a simple way of executing policy from a function in C# class library. This gives us the ability to call Business Rules from non BizTalk services. </p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Following is a sample showing a programmatic execution of BRE policy</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/37b70e1eda3110461a59ec48752ac02d.js"></script></p>

<!-- wp:paragraph -->
<p><strong><em>Pretty simple, isn't it??  But wait, now we have lost the ability to track policy execution</em></strong></p>
<!-- /wp:paragraph -->

<!-- wp:heading -->
<h2>Default tracking for Policy executed from Function</h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>MS BRE comes with a out of the box Debug Tracking interceptor, which records the execution details for us. This interceptor is available in the <strong>Microsoft.RuleEngine</strong> assembly. </p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Below is how we can attach the interceptor to the policy execution</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/23260948a7a38c4e2fc797646e10ada6.js"></script></p>

<!-- wp:paragraph -->
<p>When any service executes policy as shown above,  output as shown below gets logged.</p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":384} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/09/DebugTrackingInterceptor-1024x741.jpg" alt="" class="wp-image-384"/></figure>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p>If we compare this output with the one generated while testing the rules in the compose, we find that they are same.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p><strong>Yay!! A way to track the policy execution</strong></p>
<!-- /wp:paragraph -->

<!-- wp:heading -->
<h2>Correlated tracking for Policy executed from Function </h2>
<!-- /wp:heading -->

<!-- wp:heading {"level":3} -->
<h3>Why use Structured Logging</h3>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>Sometimes, it is possible that Rules Execution is just a small part of a loosely coupled distributed system.   In such systems,  it is not always easy to visualize the flow of the message. So it becomes imperative that we have a strong logging in place to debug and address any issues.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Correlated logging is a best practice to log the events across distributed systems.  Correlated logging uses a unique id to track the end to end flow of a message in the integration.  So, when we have to debug, we just use the unique id and we can trace the entire flow for the message.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>In such cases, the <strong><em>DebugTrackingInterceptor </em></strong>is not sufficient as  does not allow us to write custom information to the log file.  </p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Fortunately, MS BRE provides us a way to implement a custom interceptor of our own. We need to implement the <strong><em>IRuleSetTrackingInterceptor </em></strong>interface on our custom interceptor class. This interface like the DebugTrackingInterceptor is available in <strong><em>Microsoft.RuleEngine</em></strong> namespace.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>so without ado, let us start. </p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":3} -->
<h3>Setting up Log4Net</h3>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>Let us add the Log4net NuGet package to our  solution.  If you are not aware of log4net, I suggest you to read <a href="https://logging.apache.org/log4net/" target="_blank" rel="noreferrer noopener" aria-label=" (opens in a new tab)">https://logging.apache.org/log4net/</a></p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":391} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/09/Log4netNuackage.jpg" alt="" class="wp-image-391"/></figure>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p>Post that we add the log4net information to AssemblyInfo.cs class of our project as shown below.</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/c862412d03e5a640d2a0eb6067c3761c.js"></script></p>

<!-- wp:paragraph -->
<p>Now let us add the log for net configuration to our app.config file. The entire app.config looks as below</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/7e60fda9b0ccb3c4ddd2d671132d0c53.js"></script></p>

<!-- wp:paragraph -->
<p>The log4net logger is configured in code as shown below.</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/2e62f0be97b7af900d07e002d96ef5fa.js"></script></p>

<!-- wp:paragraph -->
<p>Now let us take a look at how we can implement our custom tracking interceptor.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>We implement the logger in the constructor of our custom interface. </p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/b6df321ad3acc1ebd2c04404872131a3.js"></script></p>

<!-- wp:paragraph -->
<p>We need to implement following methods from the <strong><em>IRuleSetTrackingInterceptor</em></strong></p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p><strong>SetTrackingConfig</strong> : We can keep the method body as  empty.</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/599558017fe12026ea394a626d509233.js"></script></p>

<!-- wp:paragraph -->
<p></p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p><strong>TrackAgendaUpdate</strong>: Rule engine invokes this method when it either adds or removes a rule from the agenda. Following sample shows how we track agenda update events</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/903be970e79fb7b7553ff9dce7479b7b.js"></script></p>

<!-- wp:paragraph -->
<p><strong>TrackConditionEvaluation</strong>:  Rule Engine calls this method when evaluating the conditions in the rules. This method is invoked for each condition evaluation.</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/0cd0aa0a5a2ca67be481131d1ee4a826.js"></script></p>

<!-- wp:paragraph -->
<p><strong>TrackFactActivity</strong>: Track the activity of the facts in the rule engine working memory.</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/a4226b08bae3a829205e491ac5ea37ee.js"></script></p>

<!-- wp:paragraph -->
<p><strong>TrackRuleFiring</strong>: Rule engine invokes this for each of the rule firing.</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/79f14864c894ee936135d6bb6a58fb85.js"></script></p>

<!-- wp:paragraph -->
<p><strong>TrackRuleSetEngineAssociation</strong>: Invoked at start of each rule engine execution cycle</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/4a8a1c65e4c894fe6d278cceb720f25a.js"></script></p>

<!-- wp:paragraph -->
<p>Final implementation looks as following</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/0f610a74716921898921f8963da3ff7c.js"></script></p>

<!-- wp:paragraph -->
<p>This Interceptor can be passed to policy execution as shown below.</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/5a5ec75c66f3d1087512b0b35cea00e9.js"></script></p>

<!-- wp:paragraph -->
<p>Below is the tracking of execution of policy shown above.</p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":404} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/09/CustomTracking-1024x389.jpg" alt="" class="wp-image-404"/></figure>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p>As we can see, we can now track the execution of Policy in a correlated way. Plus it gives us more freedom to log the events as we like.</p>
<!-- /wp:paragraph -->

<!-- wp:heading -->
<h2>Summing up</h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>As seen above, we have a lot of different ways of tracking policy execution of MS BRE policy. </p>
<!-- /wp:paragraph -->

<!-- wp:list {"ordered":true} -->
<ol><li>Tracking with <strong>"Call Rules"</strong> shape gives us tracking in tracked instances</li><li>Rules Composer shows us the execution of policy</li><li>DebugTrackingInterceptor implements the same logic as Rules composer testing. It outputs a simple trace, but we do not have control over the output</li><li>We can create custom tracking interceptors using logging framework of our choice.</li><li>Custom tracking gives us more control on what and how we want to track the execution events.</li></ol>
<!-- /wp:list -->

<!-- wp:paragraph -->
<p>That is it for this post. </p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Hope you liked it. I will see you in next post.</p>
<!-- /wp:paragraph -->