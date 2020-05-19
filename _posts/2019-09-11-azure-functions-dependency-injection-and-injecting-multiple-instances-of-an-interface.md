---
ID: 409
post_title: >
  Dependency Injection and Injecting
  Multiple Instances of an Interface in
  Azure Functions
author: Mandar Dharmadhikari
post_excerpt: ""
layout: post
permalink: >
  https://theabodeofcode.com/azure-functions-dependency-injection-and-injecting-multiple-instances-of-an-interface/
published: true
post_date: 2019-09-11 14:14:16
---
<!-- wp:paragraph -->
<p>This is a short and fun scenario that I tried out while playing around with the dependency injection in Azure Functions Runtime V2 (.NET Core run-time).</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>The Dependency Injection support  in Azure Functions comes with a lot of advantages. Some are</p>
<!-- /wp:paragraph -->

<!-- wp:list {"ordered":true} -->
<ol><li>We get loosely coupled code,  our functions are no more tied with concrete classes</li><li>We get the ability to  unit test our code more easily as we can implement fakes of our interfaces</li><li>There are no more static functions, we can inject the dependencies directly into the class constructors (Constructor Injection)</li><li>With inbuilt support for dependency injection, we do not have to rely on external DI containers like Autofac(Which I like very much by the way!)</li></ol>
<!-- /wp:list -->

<!-- wp:paragraph -->
<p>Now let us take look at a fictional scenario</p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":3} -->
<h3>Scenario</h3>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>Let us consider a scenario where we are catering to different types of customers who avail the products of our dummy company.  I have one function which caters to normal customers and one function for elite customers. The function for normal customers sends SMS notifications to customer of company discounts etc, while the function for the Elite customer sends both a SMS and a phone call from company execute. Now  I understand this not a very viable scenario, but it will suffice for our POC.</p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":3} -->
<h3>Implementation</h3>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>We begin by creating a new function project in Visual studio. You can also replicate this example in VS code.</p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":410} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/09/CreateProject.jpg" alt="" class="wp-image-410"/></figure>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p>We now add following NuGet packages to set up support for dependency injection in Azure functions</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p><strong>Microsoft.Azure.Functions.Extensions</strong> available at ( <a href="https://www.nuget.org/packages/Microsoft.Azure.Functions.Extensions/">https://www.nuget.org/packages/Microsoft.Azure.Functions.Extensions/</a> )</p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":412} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/09/InstallPackage1-1-1024x550.jpg" alt="" class="wp-image-412"/></figure>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p><strong>Microsoft.NET.Sdk.Functions</strong> available at  <a href="https://www.nuget.org/packages/Microsoft.NET.Sdk.Functions/">https://www.nuget.org/packages/Microsoft.NET.Sdk.Functions/</a> </p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":413} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/09/InstallPackage2-1024x651.jpg" alt="" class="wp-image-413"/></figure>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p>Now let us add our interface for Notification Services. It is as shown below</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/daa1e008456caea2d83c24a4674d2bbe.js"></script></p>

<!-- wp:paragraph -->
<p>Now Let us create the classes that implement this interface. </p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Following is the class which invokes the phone call process. Right now it does not do anything other than logging, but that should be sufficient to understand the concept.</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/779054a9ee6025e17945f488eaf1bac9.js"></script></p>

<!-- wp:paragraph -->
<p>Following is the class which invokes the SMS notification process. Right now it does not do anything other than logging, but that should be sufficient to understand the concept.</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/f7668adae7d3c5bcf54d9fe923077be9.js"></script></p>

<!-- wp:paragraph -->
<p>Now we register these concrete implementations with the <strong><em>FunctionHostBuilder</em></strong> so that they are injected into  the function class constructors. We do this by adding a class called Startup.cs to the project. The code in startup looks as follows</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/246f94bd58f1a49d736a03cac3a0765a.js"></script></p>

<!-- wp:paragraph -->
<p>To keep things simple, I have registered the concrete implementations as Scoped, meaning the instances are created once per a service request. To read more about the Instance lifetimes refer to official documentation at <a href="https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-2.2#service-lifetimes" target="_blank" rel="noreferrer noopener" aria-label=" (opens in a new tab)">https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-2.2#service-lifetimes</a></p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Now the hard work is done. All we need to do is implement our functions.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>My Function to Process normal customers looks like following.</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/0e0a556a5958255a4aca9865bf95fdae.js"></script></p>

<!-- wp:paragraph -->
<p><strong><em>Notice we do not have static instances of classes or function any more!!</em></strong></p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>My Function to Process Elite customers looks like following.</p>
<!-- /wp:paragraph -->

<p><script src="https://gist.github.com/mandardhikari/164fc0727fd1fc5fa8c21475a4d40cca.js"></script></p>

<!-- wp:paragraph -->
<p>Notice I am providing an <strong><em>IEnumerable</em></strong> of Type <strong><em>INotificationSeervice</em></strong> instead of INotificationService as parameter to my function class constructor. </p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>This <strong><em>IEnumberable&lt;T&gt;</em></strong> is what does the magic!!. That is it </p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Now let us test our functions</p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":3} -->
<h3>Testing the Functions<br></h3>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>I started by running the functions project in a debug mode. The functions runtime generated following urls for me.</p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":420} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/09/RUntime-Urls-1024x597.jpg" alt="" class="wp-image-420"/></figure>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p>I tested the ProcessNormalCustomer function first by executing the Get request from the browser. We can see that an instance of SMSNotification which implements the <strong>INotificationService</strong> service is injected in the constructor.</p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":421} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/09/SMSNotificationInvoked-1024x591.jpg" alt="" class="wp-image-421"/></figure>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p>After that  I tested the ProcessEliteCustomer functions and following was the output.</p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":422} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2019/09/BothNotification-1024x589.jpg" alt="" class="wp-image-422"/></figure>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p>The output clearly shows that instances for both the implementations for INotificationService were injected into the function.</p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":3} -->
<h3>Wrapping Up</h3>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>Following can be concluded from the POC</p>
<!-- /wp:paragraph -->

<!-- wp:list {"ordered":true} -->
<ol><li>The in built support for Dependency Injection makes life much more simpler as we do not need to deal with static functions</li><li><strong><em>When multiple implementations of an interface are registered, the  last one always gets injected</em></strong></li><li>To inject all the registered implementations of the interface, we pass IEnumerable&lt;T&gt; in the constructors.</li></ol>
<!-- /wp:list -->

<!-- wp:paragraph -->
<p></p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p></p>
<!-- /wp:paragraph -->