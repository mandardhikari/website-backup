---
ID: 587
post_title: >
  Running POSTMAN collections in GitHub
  Action Workflows
author: Mandar Dharmadhikari
post_excerpt: ""
layout: post
permalink: >
  https://theabodeofcode.com/running-postman-collections-in-github-action-workflows/
published: true
post_date: 2020-05-11 16:40:58
---
<!-- wp:jetpack/markdown {"source":"In my previous post [Deploying and Testing Logic Apps with GitHub Actions](https://theabodeofcode.com/deploying-and-testing-logic-apps-with-GitHub-actions/), I talked about how to deploy the logic apps and test the HTTP triggered logic apps using POSTMAN. I wanted to write a separate blog on how to specifically test any REST APIs, SOAP or WCF services and how to generate beautiful reports and upload them as build results artifacts in GitHub Actions.\nBefore that let us go through some basic information on GitHub actions.\n\n# What are GitHub actions?\nGitHub actions are the workflows that allow use to perform a set of actions when certain events occur on the repository. An event on repository can be as simple as \u0022when a push is made to master branch\u0022 or \u0022A pull request is raised on master branch\u0022. GitHub allows us to automate certain responses to these events using automated workflows called as GitHub actions. These actions react to one or multiple events and perform certain tasks e.g. \u0022Build and Run the tests\u0022, \u0022Deploy the code\u0022, \u0022Merge the code from one branch to another\u0022. This is kind of a reactive programming approach where when a developer or admin performs some action on the repository, the GitHub action reacts to that and performs something.\n\nWe can build our end to end continuous integration(CI) and deployment (CD) for the code in the repository directly in the GitHub. The great thing that I find about GitHub actions is that there are lot of actions are available out of box for us and there are some custom ones developed and shared by the community which we can implement in our code.\n\nFollowing terminology will come handy later during the post\n\n1. [Workflow](https://help.GitHub.com/en/actions/getting-started-with-GitHub-actions/core-concepts-for-GitHub-actions#workflow,\u0022Workflow\u0022): This is the automated process which will be run and can perform various tasks like builing, testing, publishing, deploying the code in the GitHub repository.\n2. [Runners](https://help.GitHub.com/en/actions/getting-started-with-GitHub-actions/core-concepts-for-GitHub-actions#runner, \u0022Runners\u0022): Runners are the machines on which the workflows are executed. These runners can be either the default runners hosted by GitHub or you can use your own hosted runners.\n3. [Job](https://help.GitHub.com/en/actions/getting-started-with-GitHub-actions/core-concepts-for-GitHub-actions#job,\u0022Job\u0022): Job is a set of steps that are executed on the same runner. It is there fore important for us to understand that all the processes that need data sharing must be clubbed under same job.\n4. [Action](https://help.GitHub.com/en/actions/getting-started-with-GitHub-actions/core-concepts-for-GitHub-actions#action, \u0022Action\u0022): Actions are the individual steps that are defined under a job.\n\nYou can read more about the GitHub actions at [About GitHub Actions](https://help.GitHub.com/en/actions/getting-started-with-GitHub-actions/about-GitHub-actions \u0022About GitHub Actions\u0022)\n\n# Scenario\n\nFor the purpose of this blog post, I am going to test the free [Pokemon API](https://pokeapi.co/). I will invoke this API through POSTMAN and run some tests on the results returned. All this is done using the postman collections. This collection then is run through the GitHub Actions workflow and the test results will be uploaded on to the actions workspace.\n\nLet us look at various steps required to set up the POSTMAN run job inside the GitHub actions workflow. Even though this blog explains only the process to run the Tests, this process can be incorporated as a part of the Continuous Integration and then in each of the release pipelines for the Continuous Deployment or a step prior to release in the Continuous Delivery.\n\n# Setting up GitHub actions\n\n## Adding an ACTION\n\nThe actions are defined in a `yaml` file and are located in the `.github/workflows` folder. This folder structure can be created using as part of coding exercise or alternatively, the workflows can be created directly on repository using the Actions pane on the GitHub website. Following image shows how to create the action on the website."} -->
<div class="wp-block-jetpack-markdown"><p>In my previous post <a href="https://theabodeofcode.com/deploying-and-testing-logic-apps-with-GitHub-actions/">Deploying and Testing Logic Apps with GitHub Actions</a>, I talked about how to deploy the logic apps and test the HTTP triggered logic apps using POSTMAN. I wanted to write a separate blog on how to specifically test any REST APIs, SOAP or WCF services and how to generate beautiful reports and upload them as build results artifacts in GitHub Actions.
Before that let us go through some basic information on GitHub actions.</p>
<h1>What are GitHub actions?</h1>
<p>GitHub actions are the workflows that allow use to perform a set of actions when certain events occur on the repository. An event on repository can be as simple as &quot;when a push is made to master branch&quot; or &quot;A pull request is raised on master branch&quot;. GitHub allows us to automate certain responses to these events using automated workflows called as GitHub actions. These actions react to one or multiple events and perform certain tasks e.g. &quot;Build and Run the tests&quot;, &quot;Deploy the code&quot;, &quot;Merge the code from one branch to another&quot;. This is kind of a reactive programming approach where when a developer or admin performs some action on the repository, the GitHub action reacts to that and performs something.</p>
<p>We can build our end to end continuous integration(CI) and deployment (CD) for the code in the repository directly in the GitHub. The great thing that I find about GitHub actions is that there are lot of actions are available out of box for us and there are some custom ones developed and shared by the community which we can implement in our code.</p>
<p>Following terminology will come handy later during the post</p>
<ol>
<li><a href="https://help.GitHub.com/en/actions/getting-started-with-GitHub-actions/core-concepts-for-GitHub-actions#workflow,%22Workflow%22">Workflow</a>: This is the automated process which will be run and can perform various tasks like builing, testing, publishing, deploying the code in the GitHub repository.</li>
<li><a href="https://help.GitHub.com/en/actions/getting-started-with-GitHub-actions/core-concepts-for-GitHub-actions#runner," title="Runners">Runners</a>: Runners are the machines on which the workflows are executed. These runners can be either the default runners hosted by GitHub or you can use your own hosted runners.</li>
<li><a href="https://help.GitHub.com/en/actions/getting-started-with-GitHub-actions/core-concepts-for-GitHub-actions#job,%22Job%22">Job</a>: Job is a set of steps that are executed on the same runner. It is there fore important for us to understand that all the processes that need data sharing must be clubbed under same job.</li>
<li><a href="https://help.GitHub.com/en/actions/getting-started-with-GitHub-actions/core-concepts-for-GitHub-actions#action," title="Action">Action</a>: Actions are the individual steps that are defined under a job.</li>
</ol>
<p>You can read more about the GitHub actions at <a href="https://help.GitHub.com/en/actions/getting-started-with-GitHub-actions/about-GitHub-actions" title="About GitHub Actions">About GitHub Actions</a></p>
<h1>Scenario</h1>
<p>For the purpose of this blog post, I am going to test the free <a href="https://pokeapi.co/">Pokemon API</a>. I will invoke this API through POSTMAN and run some tests on the results returned. All this is done using the postman collections. This collection then is run through the GitHub Actions workflow and the test results will be uploaded on to the actions workspace.</p>
<p>Let us look at various steps required to set up the POSTMAN run job inside the GitHub actions workflow. Even though this blog explains only the process to run the Tests, this process can be incorporated as a part of the Continuous Integration and then in each of the release pipelines for the Continuous Deployment or a step prior to release in the Continuous Delivery.</p>
<h1>Setting up GitHub actions</h1>
<h2>Adding an ACTION</h2>
<p>The actions are defined in a <code>yaml</code> file and are located in the <code>.github/workflows</code> folder. This folder structure can be created using as part of coding exercise or alternatively, the workflows can be created directly on repository using the Actions pane on the GitHub website. Following image shows how to create the action on the website.</p>
</div>
<!-- /wp:jetpack/markdown -->

<!-- wp:image {"id":589} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2020/05/CreateActtion-1024x312.jpg" alt="" class="wp-image-589"/><figcaption>Create Action Workflow on Repository on website directly</figcaption></figure>
<!-- /wp:image -->

<!-- wp:image {"id":590} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2020/05/RepositoryLocation-1024x303.jpg" alt="" class="wp-image-590"/><figcaption>Workflow location in repository</figcaption></figure>
<!-- /wp:image -->

<!-- wp:jetpack/markdown {"source":"## Workflow File\n\nWe start by adding the trigger on which our GitHub Action workflow will run. This is done in the `yaml` file as shown below\n\n```yaml\n#Name of the GitHub Action\nname: Test-Build\n\n#Set the action on which the workflow will trigger\non:\n push:\n   branches:\n     - master\n\n```\nAbove lines of code define the name of the workflow and the that it will run when a `push` event occurs on the `master` branch of the repository. Now we create the job which will actually run the POSTMAN collection. \n\n```yaml\njobs:\n  test-api:\n    runs-on: ubuntu-latest\n```\nNow we start defining the individual steps that will be executed inside the job. \n\n### Checking out the code\n\nThe code is checked out using following set of command\n\n```yaml\n   steps:\n\n    #Checkout the repository\n    - name: Checkout Repository\n      uses: actions/checkout@master\n```\n`actions/checkout@master` is the action available across all the runners and we can use actions offered by GitHub or developed by community.\n\n### Installing Node and Newman\n\nPOSTMAN can be run inside the CICD agents using command prompt utility called [`Newman`](https://learning.postman.com/docs/postman/collection-runs/command-line-integration-with-newman/,\u0022`NewMan`\u0022). To install newman we first need to install `node.js` on the runner. It is done as shown below.\n\n```yml\n# Install Node.js environment\n- name: set up node\n      uses: actions/setup-node@v1\n      with:\n          node-version: '12.x'\n\n# Install newman\n    - name: Install newman\n      run: | \n       npm install -g newman\n       npm install -g newman-reporter-htmlextra\n```\nThe `newman-reporter-htmlextra` installs a custom reporter which will generate visually appealing reports for us.\n\n### Directory for Test Results\n\nAs a custom in CICD process, we need a workspace to upload the artifacts generated during the run, these artifacts can be the logs generated by the agent, custom logs written in steps, test results etc. GitHub Actions provide a way to upload the files from the job to the workspace. The trick is to create a directory and upload the results to it. The directory can be created using following step.\n\n```yaml\n- name: Make Directory for results\n  run: mkdir -p testResults\n```\n### Running the POSTMAN collection\nAs the groundwork is now done, next task is to execute the tests defined in the POSTMAN collection. The POSTMAN collection and the environment files are available in the repository as shown below. We just need to provide the relative path to the root of the repository."} -->
<div class="wp-block-jetpack-markdown"><h2>Workflow File</h2>
<p>We start by adding the trigger on which our GitHub Action workflow will run. This is done in the <code>yaml</code> file as shown below</p>
<pre><code class="language-yaml">#Name of the GitHub Action
name: Test-Build

#Set the action on which the workflow will trigger
on:
 push:
   branches:
     - master

</code></pre>
<p>Above lines of code define the name of the workflow and the that it will run when a <code>push</code> event occurs on the <code>master</code> branch of the repository. Now we create the job which will actually run the POSTMAN collection.</p>
<pre><code class="language-yaml">jobs:
  test-api:
    runs-on: ubuntu-latest
</code></pre>
<p>Now we start defining the individual steps that will be executed inside the job.</p>
<h3>Checking out the code</h3>
<p>The code is checked out using following set of command</p>
<pre><code class="language-yaml">   steps:

    #Checkout the repository
    - name: Checkout Repository
      uses: actions/checkout@master
</code></pre>
<p><code>actions/checkout@master</code> is the action available across all the runners and we can use actions offered by GitHub or developed by community.</p>
<h3>Installing Node and Newman</h3>
<p>POSTMAN can be run inside the CICD agents using command prompt utility called <a href="https://learning.postman.com/docs/postman/collection-runs/command-line-integration-with-newman/,%22%60NewMan%60%22"><code>Newman</code></a>. To install newman we first need to install <code>node.js</code> on the runner. It is done as shown below.</p>
<pre><code class="language-yml"># Install Node.js environment
- name: set up node
      uses: actions/setup-node@v1
      with:
          node-version: '12.x'

# Install newman
    - name: Install newman
      run: | 
       npm install -g newman
       npm install -g newman-reporter-htmlextra
</code></pre>
<p>The <code>newman-reporter-htmlextra</code> installs a custom reporter which will generate visually appealing reports for us.</p>
<h3>Directory for Test Results</h3>
<p>As a custom in CICD process, we need a workspace to upload the artifacts generated during the run, these artifacts can be the logs generated by the agent, custom logs written in steps, test results etc. GitHub Actions provide a way to upload the files from the job to the workspace. The trick is to create a directory and upload the results to it. The directory can be created using following step.</p>
<pre><code class="language-yaml">- name: Make Directory for results
  run: mkdir -p testResults
</code></pre>
<h3>Running the POSTMAN collection</h3>
<p>As the groundwork is now done, next task is to execute the tests defined in the POSTMAN collection. The POSTMAN collection and the environment files are available in the repository as shown below. We just need to provide the relative path to the root of the repository.</p>
</div>
<!-- /wp:jetpack/markdown -->

<!-- wp:image {"id":593} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2020/05/TestCollectionLocation-1024x329.jpg" alt="" class="wp-image-593"/><figcaption>Test Collection in the repository</figcaption></figure>
<!-- /wp:image -->

<!-- wp:jetpack/markdown {"source":"The step shown below executes the POSTMAN collection and creates the html report for the test run.\n\n```yml\n# Run POSTMAN collection and output the file as html\n- name: Run POSTMAN collection\n      run: |\n       newman run ./tests/pokemon-api-collection.json -e ./tests/pokemonapi-env.json -r htmlextra \u002d\u002dreporter-htmlextra-export testResults/htmlreport.html \u002d\u002dreporter-htmlextra-darkTheme  \u003e testResults/runreport1.html   \n```\n### Uploading Artifacts\n\nAs the last step we upload the artifacts to the workspace, it can be done using the `actions/upload-artifact@v2` as shown below. We are uploading the entire contents of the `testResults` directory that we created earlier.\n\n```yaml\n# Upload test results\n- name: Output the run Details\n      uses: actions/upload-artifact@v2\n      with: \n       name: RunReports\n       path: testResults\n\n```\nThis completes the workflow. The entire workflow file is as below.\n\n```yaml\nname: Test-Build\non:\n  push:\n    branches:\n      - master\njobs:\n  test-api:\n    runs-on: ubuntu-latest\n    steps:\n    # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it\n    - uses: actions/checkout@v2\n      \n    # INstall Node on the runner\n    - name: Install Node\n      uses: actions/setup-node@v1\n      with: \n        node-version: '12.x'\n    \n    # Install the newman command line utility and also install the html extra reporter\n    - name: Install newman\n      run: |\n       npm install -g newman\n       npm install -g newman-reporter-htmlextra\n\n    # Make directory to upload the test results\n    - name: Make Directory for results\n      run: mkdir -p testResults\n\n\n    # Run the POSTMAN collection\n    - name: Run POSTMAN collection\n      run: |\n       newman run ./tests/pokemon-api-collection.json -e ./tests/pokemonapi-env.json -r htmlextra \u002d\u002dreporter-htmlextra-export testResults/htmlreport.html \u002d\u002dreporter-htmlextra-darkTheme  \u003e testResults/runreport1.html\n\n    # Upload the contents of Test Results directory to workspace\n    - name: Output the run Details\n      uses: actions/upload-artifact@v2\n      with: \n       name: RunReports\n       path: testResults\n     \n```\n\n# Testing\n\nWhen a push is made to the `master` repository, the workflow is triggered. The output of each build is available on the Actions pane of the repository and we can drill down into each flow to see why it failed. In case of the failure, by default a notification is sent to the owner of the repository to notify them of the failed build. This behavior can be changed to send build success notifications as well. Following images show the result of the run and the output generated."} -->
<div class="wp-block-jetpack-markdown"><p>The step shown below executes the POSTMAN collection and creates the html report for the test run.</p>
<pre><code class="language-yml"># Run POSTMAN collection and output the file as html
- name: Run POSTMAN collection
      run: |
       newman run ./tests/pokemon-api-collection.json -e ./tests/pokemonapi-env.json -r htmlextra --reporter-htmlextra-export testResults/htmlreport.html --reporter-htmlextra-darkTheme  &gt; testResults/runreport1.html   
</code></pre>
<h3>Uploading Artifacts</h3>
<p>As the last step we upload the artifacts to the workspace, it can be done using the <code>actions/upload-artifact@v2</code> as shown below. We are uploading the entire contents of the <code>testResults</code> directory that we created earlier.</p>
<pre><code class="language-yaml"># Upload test results
- name: Output the run Details
      uses: actions/upload-artifact@v2
      with: 
       name: RunReports
       path: testResults

</code></pre>
<p>This completes the workflow. The entire workflow file is as below.</p>
<pre><code class="language-yaml">name: Test-Build
on:
  push:
    branches:
      - master
jobs:
  test-api:
    runs-on: ubuntu-latest
    steps:
    # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
    - uses: actions/checkout@v2
      
    # INstall Node on the runner
    - name: Install Node
      uses: actions/setup-node@v1
      with: 
        node-version: '12.x'
    
    # Install the newman command line utility and also install the html extra reporter
    - name: Install newman
      run: |
       npm install -g newman
       npm install -g newman-reporter-htmlextra

    # Make directory to upload the test results
    - name: Make Directory for results
      run: mkdir -p testResults


    # Run the POSTMAN collection
    - name: Run POSTMAN collection
      run: |
       newman run ./tests/pokemon-api-collection.json -e ./tests/pokemonapi-env.json -r htmlextra --reporter-htmlextra-export testResults/htmlreport.html --reporter-htmlextra-darkTheme  &gt; testResults/runreport1.html

    # Upload the contents of Test Results directory to workspace
    - name: Output the run Details
      uses: actions/upload-artifact@v2
      with: 
       name: RunReports
       path: testResults
     
</code></pre>
<h1>Testing</h1>
<p>When a push is made to the <code>master</code> repository, the workflow is triggered. The output of each build is available on the Actions pane of the repository and we can drill down into each flow to see why it failed. In case of the failure, by default a notification is sent to the owner of the repository to notify them of the failed build. This behavior can be changed to send build success notifications as well. Following images show the result of the run and the output generated.</p>
</div>
<!-- /wp:jetpack/markdown -->

<!-- wp:image {"id":594} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2020/05/Build-Result-1024x362.jpg" alt="" class="wp-image-594"/><figcaption>Successful build with Run Reports uploaded to workspace</figcaption></figure>
<!-- /wp:image -->

<!-- wp:image {"id":595} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2020/05/JobResults-1024x414.jpg" alt="" class="wp-image-595"/><figcaption>Steps execution in a job</figcaption></figure>
<!-- /wp:image -->

<!-- wp:image {"id":596} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2020/05/JobRunSuccessNotification-1024x409.jpg" alt="" class="wp-image-596"/><figcaption>Job Success notification</figcaption></figure>
<!-- /wp:image -->

<!-- wp:jetpack/markdown {"source":"The `html-extra` reporter creates a beautiful report which allows us to examine the entire collection run. Few screen shots of the run report are shown below.\n"} -->
<div class="wp-block-jetpack-markdown"><p>The <code>html-extra</code> reporter creates a beautiful report which allows us to examine the entire collection run. Few screen shots of the run report are shown below.</p>
</div>
<!-- /wp:jetpack/markdown -->

<!-- wp:image {"id":597} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2020/05/RunReport1-1024x674.jpg" alt="" class="wp-image-597"/><figcaption>Run Summary</figcaption></figure>
<!-- /wp:image -->

<!-- wp:image {"id":598} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2020/05/RunReport2-1024x965.jpg" alt="" class="wp-image-598"/><figcaption>Successful runs details</figcaption></figure>
<!-- /wp:image -->

<!-- wp:jetpack/markdown {"source":"# Wrapping up\n\nIn this post we saw how we can run the POSTMAN collections in the GitHub actions workflows and how we can create beautiful reports from the run. This sequence of step can be used in different CI CD scenarios."} -->
<div class="wp-block-jetpack-markdown"><h1>Wrapping up</h1>
<p>In this post we saw how we can run the POSTMAN collections in the GitHub actions workflows and how we can create beautiful reports from the run. This sequence of step can be used in different CI CD scenarios.</p>
</div>
<!-- /wp:jetpack/markdown -->