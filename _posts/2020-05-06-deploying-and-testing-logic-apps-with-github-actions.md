---
ID: 545
post_title: >
  Deploying and Testing Logic Apps with
  GitHub Actions
author: Mandar Dharmadhikari
post_excerpt: >
  In this post I have examined the process
  to deploy a simple Logic App using ARM
  templates and GitHub Actions
layout: post
permalink: >
  https://theabodeofcode.com/deploying-and-testing-logic-apps-with-github-actions/
published: true
post_date: 2020-05-06 19:15:58
---
<!-- wp:jetpack/markdown {"source":"\nI have been examining GitHub actions for last few days and I decided to try out a few workflows myself to learn how the CI CD process is implemented using them. In this post I will examine mostly on how to deploy a Logic App to a resource group and test it after the deployment. What follows now is the basic scenario which I used to understand the GitHub actions.\n\nLast few months there has been an increase in chatter over the internet around GitHub actions so let's try to answer the question\n\n# What are GitHub actions?\nGitHub actions are the workflows that allow use to perform a set of actions when certain events occur on the repository. An event on repository can be as simple as \u0022when a push is made to master branch\u0022 or \u0022A pull request is raised on master branch\u0022. GitHub allows us to automate certain responses to these events using automated workflows called as GitHub Actions. These actions react to one or multiple events and perform certain tasks e.g. \u0022Build and Run the tests\u0022, \u0022Deploy the code\u0022, \u0022Merge the code from one branch to another\u0022. This is kind of a reactive programming approach where when a developer or admin performs some action on the repository, the GitHub Actions workflow reacts to that and performs something.\n\nWe can build our end to end continuous integration(CI) and deployment (CD) for the code in the repository directly in the GitHub. The great thing that I find about GitHub actions is that there are lot of actions are available out of box for us and there are some custom ones developed and shared by the community which we can implement in our code.\n\nFollowing terminology will come handy later during the post\n\n1. [Workflow](https://help.github.com/en/actions/getting-started-with-github-actions/core-concepts-for-github-actions#workflow,\u0022Workflow\u0022): This is the automated process which will be run and can perform various tasks like building, testing, publishing, deploying the code in the GitHub repository.\n2. [Runners](https://help.github.com/en/actions/getting-started-with-github-actions/core-concepts-for-github-actions#runner, \u0022Runners\u0022): Runners are the machines on which the workflows are executed. These runners can be either the default runners hosted by GitHub or you can use your own hosted runners.\n3. [Job](https://help.github.com/en/actions/getting-started-with-github-actions/core-concepts-for-github-actions#job,\u0022Job\u0022): Job is a set of steps that are executed on the same runner. It is therefore important for us to understand that all the processes that need data sharing must be clubbed under same job.\n4. [Action](https://help.github.com/en/actions/getting-started-with-github-actions/core-concepts-for-github-actions#action, \u0022Action\u0022): Actions are the individual steps that are defined under a job.\n\nYou can read more about the GitHub actions at [About GitHub Actions](https://help.github.com/en/actions/getting-started-with-github-actions/about-github-actions, \u0022About GitHUb Actions\u0022)\n\n# Scenario\nFor the purpose of this post, I have created a simple HTTP triggered logic app which accepts following input\n\n```Json\n{\n\u0022FirstName\u0022: \u0022Jon\u0022,\n\u0022LastName\u0022: \u0022Snow\u0022\n}\n```\nand returns following response\n\n```Json\n{\n    \u0022FirstName\u0022: \u0022Jon\u0022,\n    \u0022FullName\u0022: \u0022Jon Snow\u0022,\n    \u0022LastName\u0022: \u0022Snow\u0022\n}\n```\nThe logic app is very simple as shown below."} -->
<div class="wp-block-jetpack-markdown"><p>I have been examining GitHub actions for last few days and I decided to try out a few workflows myself to learn how the CI CD process is implemented using them. In this post I will examine mostly on how to deploy a Logic App to a resource group and test it after the deployment. What follows now is the basic scenario which I used to understand the GitHub actions.</p>
<p>Last few months there has been an increase in chatter over the internet around GitHub actions so let's try to answer the question</p>
<h1>What are GitHub actions?</h1>
<p>GitHub actions are the workflows that allow use to perform a set of actions when certain events occur on the repository. An event on repository can be as simple as &quot;when a push is made to master branch&quot; or &quot;A pull request is raised on master branch&quot;. GitHub allows us to automate certain responses to these events using automated workflows called as GitHub Actions. These actions react to one or multiple events and perform certain tasks e.g. &quot;Build and Run the tests&quot;, &quot;Deploy the code&quot;, &quot;Merge the code from one branch to another&quot;. This is kind of a reactive programming approach where when a developer or admin performs some action on the repository, the GitHub Actions workflow reacts to that and performs something.</p>
<p>We can build our end to end continuous integration(CI) and deployment (CD) for the code in the repository directly in the GitHub. The great thing that I find about GitHub actions is that there are lot of actions are available out of box for us and there are some custom ones developed and shared by the community which we can implement in our code.</p>
<p>Following terminology will come handy later during the post</p>
<ol>
<li><a href="https://help.github.com/en/actions/getting-started-with-github-actions/core-concepts-for-github-actions#workflow,%22Workflow%22">Workflow</a>: This is the automated process which will be run and can perform various tasks like building, testing, publishing, deploying the code in the GitHub repository.</li>
<li><a href="https://help.github.com/en/actions/getting-started-with-github-actions/core-concepts-for-github-actions#runner," title="Runners">Runners</a>: Runners are the machines on which the workflows are executed. These runners can be either the default runners hosted by GitHub or you can use your own hosted runners.</li>
<li><a href="https://help.github.com/en/actions/getting-started-with-github-actions/core-concepts-for-github-actions#job,%22Job%22">Job</a>: Job is a set of steps that are executed on the same runner. It is therefore important for us to understand that all the processes that need data sharing must be clubbed under same job.</li>
<li><a href="https://help.github.com/en/actions/getting-started-with-github-actions/core-concepts-for-github-actions#action," title="Action">Action</a>: Actions are the individual steps that are defined under a job.</li>
</ol>
<p>You can read more about the GitHub actions at <a href="https://help.github.com/en/actions/getting-started-with-github-actions/about-github-actions," title="About GitHUb Actions">About GitHub Actions</a></p>
<h1>Scenario</h1>
<p>For the purpose of this post, I have created a simple HTTP triggered logic app which accepts following input</p>
<pre><code class="language-Json">{
&quot;FirstName&quot;: &quot;Jon&quot;,
&quot;LastName&quot;: &quot;Snow&quot;
}
</code></pre>
<p>and returns following response</p>
<pre><code class="language-Json">{
    &quot;FirstName&quot;: &quot;Jon&quot;,
    &quot;FullName&quot;: &quot;Jon Snow&quot;,
    &quot;LastName&quot;: &quot;Snow&quot;
}
</code></pre>
<p>The logic app is very simple as shown below.</p>
</div>
<!-- /wp:jetpack/markdown -->

<!-- wp:image {"id":548} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2020/05/SampleLogicApp.jpg" alt="" class="wp-image-548"/><figcaption>HTTP triggered Logic APP</figcaption></figure>
<!-- /wp:image -->

<!-- wp:jetpack/markdown {"source":"To deploy this logic app, we need to use the Azure ARM template. So the next step is to download the ARM template for the Logic App. This can be done using Visual Studio cloud explorer. The detailed instructions are available at [Manage logic apps with Visual Studio](https://docs.microsoft.com/en-us/azure/logic-apps/manage-logic-apps-with-visual-studio \u0022Manage logic apps with Visual Studio\u0022)\nThe ARM template that we will use in this sample is shown below\n\n```Json\n{\n  \u0022$schema\u0022: \u0022https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#\u0022,\n  \u0022contentVersion\u0022: \u00221.0.0.0\u0022,\n  \u0022parameters\u0022: {\n    \u0022LogicAppLocation\u0022: {\n      \u0022type\u0022: \u0022string\u0022,\n      \u0022minLength\u0022: 1,\n      \u0022allowedValues\u0022: [\n        \u0022australiaeast\u0022,\n        \u0022australiasoutheast\u0022\n      ]\n      \n    },\n    \u0022LogicAppName\u0022: {\n      \u0022type\u0022: \u0022string\u0022,\n      \u0022minLength\u0022: 1\n      \n    }\n  },\n  \u0022variables\u0022: {},\n  \u0022resources\u0022: [\n    {\n      \u0022properties\u0022: {\n        \u0022state\u0022: \u0022Enabled\u0022,\n        \u0022definition\u0022: {\n          \u0022$schema\u0022: \u0022https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#\u0022,\n          \u0022actions\u0022: {\n            \u0022Compose\u0022: {\n              \u0022type\u0022: \u0022Compose\u0022,\n              \u0022inputs\u0022: \u0022@concat(triggerBody()?['FirstName'],' ',triggerBody()?['LastName'])\u0022,\n              \u0022runAfter\u0022: {}\n            },\n            \u0022Response\u0022: {\n              \u0022type\u0022: \u0022Response\u0022,\n              \u0022kind\u0022: \u0022Http\u0022,\n              \u0022inputs\u0022: {\n                \u0022statusCode\u0022: 200,\n                \u0022headers\u0022: {\n                  \u0022Content-Type\u0022: \u0022application/json\u0022\n                },\n                \u0022body\u0022: {\n                  \u0022FirstName\u0022: \u0022@triggerBody()?['FirstName']\u0022,\n                  \u0022FullName\u0022: \u0022@outputs('Compose')\u0022,\n                  \u0022LastName\u0022: \u0022@triggerBody()?['LastName']\u0022\n                }\n              },\n              \u0022runAfter\u0022: {\n                \u0022Compose\u0022: [\n                  \u0022Succeeded\u0022\n                ]\n              }\n            }\n          },\n          \u0022parameters\u0022: {},\n          \u0022triggers\u0022: {\n            \u0022manual\u0022: {\n              \u0022type\u0022: \u0022Request\u0022,\n              \u0022kind\u0022: \u0022Http\u0022,\n              \u0022inputs\u0022: {\n                \u0022schema\u0022: {\n                  \u0022properties\u0022: {\n                    \u0022FirstName\u0022: {\n                      \u0022type\u0022: \u0022string\u0022\n                    },\n                    \u0022LastName\u0022: {\n                      \u0022type\u0022: \u0022string\u0022\n                    }\n                  },\n                  \u0022type\u0022: \u0022object\u0022\n                }\n              }\n            }\n          },\n          \u0022contentVersion\u0022: \u00221.0.0.0\u0022,\n          \u0022outputs\u0022: {}\n        },\n        \u0022parameters\u0022: {}\n      },\n      \u0022name\u0022: \u0022[parameters('LogicAppName')]\u0022,\n      \u0022type\u0022: \u0022Microsoft.Logic/workflows\u0022,\n      \u0022location\u0022: \u0022[parameters('LogicAppLocation')]\u0022,\n      \u0022tags\u0022: {\n        \u0022displayName\u0022: \u0022LogicApp\u0022\n      },\n      \u0022apiVersion\u0022: \u00222016-06-01\u0022\n    }\n  ],\n  \u0022outputs\u0022: {\n     \u0022logicAppUrl\u0022: {\n      \u0022type\u0022: \u0022string\u0022,\n      \u0022value\u0022: \u0022[listCallbackURL(concat(resourceId('Microsoft.Logic/workflows/', parameters('LogicAppName')), '/triggers/manual'), '2016-06-01').value]\u0022\n   }\n  }\n}\n```\nThe parameters file associated with this is as following.\n```Json\n{\n    \u0022$schema\u0022: \u0022https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#\u0022,\n    \u0022contentVersion\u0022: \u00221.0.0.0\u0022,\n    \u0022parameters\u0022: {\n      \u0022LogicAppName\u0022: {\n        \u0022value\u0022: \u0022az-httplogicapp-prod\u0022\n      },\n      \u0022LogicAppLocation\u0022: {\n        \u0022value\u0022: \u0022australiaeast\u0022\n      }\n    }\n  }\n```\nThese templates are checked into the repository as shown below."} -->
<div class="wp-block-jetpack-markdown"><p>To deploy this logic app, we need to use the Azure ARM template. So the next step is to download the ARM template for the Logic App. This can be done using Visual Studio cloud explorer. The detailed instructions are available at <a href="https://docs.microsoft.com/en-us/azure/logic-apps/manage-logic-apps-with-visual-studio" title="Manage logic apps with Visual Studio">Manage logic apps with Visual Studio</a>
The ARM template that we will use in this sample is shown below</p>
<pre><code class="language-Json">{
  &quot;$schema&quot;: &quot;https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#&quot;,
  &quot;contentVersion&quot;: &quot;1.0.0.0&quot;,
  &quot;parameters&quot;: {
    &quot;LogicAppLocation&quot;: {
      &quot;type&quot;: &quot;string&quot;,
      &quot;minLength&quot;: 1,
      &quot;allowedValues&quot;: [
        &quot;australiaeast&quot;,
        &quot;australiasoutheast&quot;
      ]
      
    },
    &quot;LogicAppName&quot;: {
      &quot;type&quot;: &quot;string&quot;,
      &quot;minLength&quot;: 1
      
    }
  },
  &quot;variables&quot;: {},
  &quot;resources&quot;: [
    {
      &quot;properties&quot;: {
        &quot;state&quot;: &quot;Enabled&quot;,
        &quot;definition&quot;: {
          &quot;$schema&quot;: &quot;https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#&quot;,
          &quot;actions&quot;: {
            &quot;Compose&quot;: {
              &quot;type&quot;: &quot;Compose&quot;,
              &quot;inputs&quot;: &quot;@concat(triggerBody()?['FirstName'],' ',triggerBody()?['LastName'])&quot;,
              &quot;runAfter&quot;: {}
            },
            &quot;Response&quot;: {
              &quot;type&quot;: &quot;Response&quot;,
              &quot;kind&quot;: &quot;Http&quot;,
              &quot;inputs&quot;: {
                &quot;statusCode&quot;: 200,
                &quot;headers&quot;: {
                  &quot;Content-Type&quot;: &quot;application/json&quot;
                },
                &quot;body&quot;: {
                  &quot;FirstName&quot;: &quot;@triggerBody()?['FirstName']&quot;,
                  &quot;FullName&quot;: &quot;@outputs('Compose')&quot;,
                  &quot;LastName&quot;: &quot;@triggerBody()?['LastName']&quot;
                }
              },
              &quot;runAfter&quot;: {
                &quot;Compose&quot;: [
                  &quot;Succeeded&quot;
                ]
              }
            }
          },
          &quot;parameters&quot;: {},
          &quot;triggers&quot;: {
            &quot;manual&quot;: {
              &quot;type&quot;: &quot;Request&quot;,
              &quot;kind&quot;: &quot;Http&quot;,
              &quot;inputs&quot;: {
                &quot;schema&quot;: {
                  &quot;properties&quot;: {
                    &quot;FirstName&quot;: {
                      &quot;type&quot;: &quot;string&quot;
                    },
                    &quot;LastName&quot;: {
                      &quot;type&quot;: &quot;string&quot;
                    }
                  },
                  &quot;type&quot;: &quot;object&quot;
                }
              }
            }
          },
          &quot;contentVersion&quot;: &quot;1.0.0.0&quot;,
          &quot;outputs&quot;: {}
        },
        &quot;parameters&quot;: {}
      },
      &quot;name&quot;: &quot;[parameters('LogicAppName')]&quot;,
      &quot;type&quot;: &quot;Microsoft.Logic/workflows&quot;,
      &quot;location&quot;: &quot;[parameters('LogicAppLocation')]&quot;,
      &quot;tags&quot;: {
        &quot;displayName&quot;: &quot;LogicApp&quot;
      },
      &quot;apiVersion&quot;: &quot;2016-06-01&quot;
    }
  ],
  &quot;outputs&quot;: {
     &quot;logicAppUrl&quot;: {
      &quot;type&quot;: &quot;string&quot;,
      &quot;value&quot;: &quot;[listCallbackURL(concat(resourceId('Microsoft.Logic/workflows/', parameters('LogicAppName')), '/triggers/manual'), '2016-06-01').value]&quot;
   }
  }
}
</code></pre>
<p>The parameters file associated with this is as following.</p>
<pre><code class="language-Json">{
    &quot;$schema&quot;: &quot;https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#&quot;,
    &quot;contentVersion&quot;: &quot;1.0.0.0&quot;,
    &quot;parameters&quot;: {
      &quot;LogicAppName&quot;: {
        &quot;value&quot;: &quot;az-httplogicapp-prod&quot;
      },
      &quot;LogicAppLocation&quot;: {
        &quot;value&quot;: &quot;australiaeast&quot;
      }
    }
  }
</code></pre>
<p>These templates are checked into the repository as shown below.</p>
</div>
<!-- /wp:jetpack/markdown -->

<!-- wp:image {"id":552} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2020/05/Add-ARM-Templates-1024x237.jpg" alt="" class="wp-image-552"/><figcaption>ARM Templates in the repository</figcaption></figure>
<!-- /wp:image -->

<!-- wp:jetpack/markdown {"source":"And that completes the set up of the logic app ARM templates.\n\nI like testing my APIs using POSTMAN collections. So, I am adding the steps to run POSTMAN collections in the GitHub Actions workflow. I have exported the POSTMAN collection and the environment file and added it to the repository as shown below."} -->
<div class="wp-block-jetpack-markdown"><p>And that completes the set up of the logic app ARM templates.</p>
<p>I like testing my APIs using POSTMAN collections. So, I am adding the steps to run POSTMAN collections in the GitHub Actions workflow. I have exported the POSTMAN collection and the environment file and added it to the repository as shown below.</p>
</div>
<!-- /wp:jetpack/markdown -->

<!-- wp:image {"id":553} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2020/05/Add-POSTMAN-collection-1024x222.jpg" alt="" class="wp-image-553"/><figcaption>POSTMAN test collection in repository</figcaption></figure>
<!-- /wp:image -->

<!-- wp:jetpack/markdown {"source":"# Setting Up GitHub Action Workflow\n\n## Adding AZURE Credentials\n\nI am using `Azure CLI` to do deploy the ARM templates to the Resource Groups. The workflow needs access to the azure credentials to log on to azure and execute the `Azure CLI` commands. For this we create the service principal inside the Azure Active Directory. In this scenario, we are creating a new resource group to deploy the logic app to. Hence the service principal needs the `Contributor` level access on the subscription. This is done by executing the command shown below\n\n```bash\naz ad sp create-for-rbac \u002d\u002dname \u0022arm-template-deployment\u0022 \u002d\u002drole contributor \u002d\u002dscopes /subscriptions/{subscription-id} \u002d\u002dsdk-auth\n\n```\nThis command can be run from either the console on local machine or by using `Azure Cloud Shell`. once executed it will return a JSON containing the details of the service principal.\n\nThe next step is to add this secret to the GitHub repository so that the workflow has access to it. The secrets can be accessed inside the workflow using `${{secrets.secretName}}` syntax. This allows us to save the sensitive information in the secrets and access it later on in the workflow. The secret can be added to the repository as shown in following images."} -->
<div class="wp-block-jetpack-markdown"><h1>Setting Up GitHub Action Workflow</h1>
<h2>Adding AZURE Credentials</h2>
<p>I am using <code>Azure CLI</code> to do deploy the ARM templates to the Resource Groups. The workflow needs access to the azure credentials to log on to azure and execute the <code>Azure CLI</code> commands. For this we create the service principal inside the Azure Active Directory. In this scenario, we are creating a new resource group to deploy the logic app to. Hence the service principal needs the <code>Contributor</code> level access on the subscription. This is done by executing the command shown below</p>
<pre><code class="language-bash">az ad sp create-for-rbac --name &quot;arm-template-deployment&quot; --role contributor --scopes /subscriptions/{subscription-id} --sdk-auth

</code></pre>
<p>This command can be run from either the console on local machine or by using <code>Azure Cloud Shell</code>. once executed it will return a JSON containing the details of the service principal.</p>
<p>The next step is to add this secret to the GitHub repository so that the workflow has access to it. The secrets can be accessed inside the workflow using <code>${{secrets.secretName}}</code> syntax. This allows us to save the sensitive information in the secrets and access it later on in the workflow. The secret can be added to the repository as shown in following images.</p>
</div>
<!-- /wp:jetpack/markdown -->

<!-- wp:image {"id":556} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2020/05/Create-Azure-Credential-Secret-1024x488.jpg" alt="" class="wp-image-556"/><figcaption>Secrets is available in the settings Pane</figcaption></figure>
<!-- /wp:image -->

<!-- wp:image {"id":559} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2020/05/Add-New-Secret-1-1024x589.jpg" alt="" class="wp-image-559"/><figcaption>Add secret AZURE_CREDENTIALS</figcaption></figure>
<!-- /wp:image -->

<!-- wp:jetpack/markdown {"source":"## Adding an ACTION\n\nThe actions are defined in a `yaml` file and are located in the `.github/workflows` folder. This folder structure can be created using as part of coding exercise or alternatively, the workflows can be created directly on repository using the Actions pane on the GitHub website. Following image shows how to create the action on the website."} -->
<div class="wp-block-jetpack-markdown"><h2>Adding an ACTION</h2>
<p>The actions are defined in a <code>yaml</code> file and are located in the <code>.github/workflows</code> folder. This folder structure can be created using as part of coding exercise or alternatively, the workflows can be created directly on repository using the Actions pane on the GitHub website. Following image shows how to create the action on the website.</p>
</div>
<!-- /wp:jetpack/markdown -->

<!-- wp:image {"id":561} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2020/05/Add-Action-1024x366.jpg" alt="" class="wp-image-561"/><figcaption>Creating action on the webpage</figcaption></figure>
<!-- /wp:image -->

<!-- wp:image {"id":562} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2020/05/Workflow-location-in-repository-1024x329.jpg" alt="" class="wp-image-562"/><figcaption>W<br>Workflow location in repository</figcaption></figure>
<!-- /wp:image -->

<!-- wp:jetpack/markdown {"source":"I prefer writing my `yaml` file using `Visual Studio Code`. And I like the `GitHub Actions` extension available. You can check out the extension at [GitHub Actions Extension](https://marketplace.visualstudio.com/items?itemName=me-dutour-mathieu.vscode-github-actions, \u0022GitHub Actions Extension\u0022). This extension makes it easier to author the workflow files as it provides a built in intellisense. \nNow that we have the basic stuff in place, let us take a look at the flow of the workflow. The various steps that this flow will perform are shown below"} -->
<div class="wp-block-jetpack-markdown"><p>I prefer writing my <code>yaml</code> file using <code>Visual Studio Code</code>. And I like the <code>GitHub Actions</code> extension available. You can check out the extension at <a href="https://marketplace.visualstudio.com/items?itemName=me-dutour-mathieu.vscode-github-actions," title="GitHub Actions Extension">GitHub Actions Extension</a>. This extension makes it easier to author the workflow files as it provides a built in intellisense.
Now that we have the basic stuff in place, let us take a look at the flow of the workflow. The various steps that this flow will perform are shown below</p>
</div>
<!-- /wp:jetpack/markdown -->

<!-- wp:image {"id":564} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2020/05/flow.jpg" alt="" class="wp-image-564"/><figcaption>workflow steps</figcaption></figure>
<!-- /wp:image -->

<!-- wp:jetpack/markdown {"source":"## The Workflow File\n\nWe start by adding the trigger on which our GitHub Action workflow will run. This is done in the `yaml` file as shown below\n\n```yaml\n#Name of the GitHub Action\nname: Deploy Logic App\n\n#Set the action on which the workflow will trigger\non:\n push:\n   branches:\n     - master\n\n```\nAbove lines of code define the name of the workflow and the that it will run when a `push` event occurs on the `master` branch of the repository.\n\nNext we create the job and  define the environment variables that are used and the runner on which the job will be run.\n\n```yaml\njobs:\n  validate-and-deploy:\n   runs-on: ubuntu-latest\n   env:\n     ENV_RESOURCEGROUP: az-logicapp-githubactions-demo-rg\n     ENV_RESOURCEGROUPLOCATION: australiaeast\n```\nThis piece of code will create  a job called `validate-and-deploy` and this will be run on a `ubuntu` runner. Also the value of the `resource-group` and `location` variables are also set so as to be available across the entire job.\n\nNow we start defining the individual steps that will be executed inside the job. \n\n### Checking out the code\n\nThe code is checked out using following set of command\n\n```yaml\n   steps:\n\n    #Checkout the repository\n    - name: Checkout Repository\n      uses: actions/checkout@master\n```\n`actions/checkout@master` is the action available across all the runners and we can use actions offered by GitHub or developed by community.\n\n### Logging into Azure\nWe login to the Azure using the credentials we saved while creating the service principal along with the `azure/login@v1` action available from GitHub community. The code is shown below.\n\n```yaml\n#Use the azure provided action to log on to azure using service principal\n    - name: Login to Azure\n      uses: azure/login@v1\n      with:\n          creds: ${{secrets.AZURE_CREDENTIALS}}\n```\nNotice how the credentials are accessed using `secrets.AZURE_CREDENTIALS`\n\n### Creating the Resource Group\n\nThe resource group is created using following\n\n```yml\n# Create the resource group\n    - name: Create Resource Group\n      run: az group create -n ${{env.ENV_RESOURCEGROUP}} -l ${{env.ENV_RESOURCEGROUPLOCATION}}\n```\n\n### Validating and Deploying the ARM Template\n\nThe ARM templates are validated and deployed using the `az deployment` CLI commands. The can be found at [Az Deployment CLI](\u0022https://docs.microsoft.com/en-us/cli/azure/deployment?view=azure-cli-latest\u0022, \u0022Az Deployment CLI\u0022)\n\nThe code for validation and the deployment is as following.\n\n```yml\n#Validate the  ARM template\n    - name: Validate ARM template\n      run: |\n        az deployment group validate -g ${{env.ENV_RESOURCEGROUP}} \u002d\u002dmode Incremental \u002d\u002dtemplate-file ./src/Deployment/azure-logicapp-demo.deploy.json \u002d\u002dparameters ./src/Deployment/azure-logicapp-demo.deploy.parameters.json\n    \n    #Deploy the ARM Template\n    - name: Deploy Logic App\n      run: |\n       echo \u0022::set-env name=logicappurl::$(az deployment group create -g az-logicapp-githubactions-demo-rg \u002d\u002dtemplate-file ./src/Deployment/azure-logicapp-demo.deploy.json \u002d\u002dparameters ./src/Deployment/azure-logicapp-demo.deploy.parameters.json \u002d\u002dquery 'properties.outputs.logicAppUrl.value' -o tsv)\u0022\n      shell: bash   \n\n```\nHere we are also saving the logic app trigger URL by reading it from the `az deployment group create` CLI command and saving it into a variable called `logicappurl`. It should be noted that though the value is set in the `Deploy Logic App` step, it will not be accessible to the actions in this step, all following steps will have access to the `logicappurl` variable.\n\n### Logging Out\n\nOnce the deployment is done, the workflow will log out of the azure. It can be done as shown below.\n\n```yml\n# Log Out From Azure \n    - name: Logout\n      run: az logout\n```\nThis completes the deployment part of the workflow, now we want to run some integration tests against the Logic App that was deployed. As mentioned earlier, I used POSTMAN collections to test the API.\n\n### Installing Node and Newman\n\nPOSTMAN can be run inside the CI CD agents using command prompt utility called [`Newman`](https://learning.postman.com/docs/postman/collection-runs/command-line-integration-with-newman/,\u0022`NewMan`\u0022). To install newman we first need to install `node.js` on the runner. It is done as shown below.\n\n```yml\n# Install Node.js environment\n- name: set up node\n      uses: actions/setup-node@v1\n      with:\n          node-version: '12.x'\n\n# Install newman\n    - name: Install newman\n      run: npm install -g newman\n```\n\n### Running POSTMAN collections\n\nOnce `newman` is installed, it becomes easy to run the postman collection. It is done as shown in following code snippet.\n\n```yml\n- name: Run Postman collection\n      run: |\n       newman run ./src/Test/logicapp-githubactions-test-collection.json -e ./src/Test/logicapp-githubactions-test-env.json \u002d\u002denv-var \u0022url= ${{env.logicappurl}}\u0022\n```\nIt is evident why we required the `logicappurl` variable in the first place ;)\n\nThis completes our workflow. The entire workflow definition as a `yaml` file is shown below.\n```yml\n#Name of the GitHub Action\nname: Deploy Logic App\n\n#Set the action on which the workflow will trigger\non:\n push:\n   branches:\n     - master\n\njobs:\n  validate-and-deploy:\n   runs-on: ubuntu-latest\n   env:\n     ENV_RESOURCEGROUP: az-logicapp-githubactions-demo-rg\n     ENV_RESOURCEGROUPLOCATION: australiaeast\n\n   steps:\n\n    #Checkout the repository\n    - name: Checkout Repository\n      uses: actions/checkout@master\n\n    #Use the azure provided action to log on to azure using service pricipal\n    - name: Login to Azure\n      uses: azure/login@v1\n      with:\n          creds: ${{secrets.AZURE_CREDENTIALS}}\n    \n    # Create the resource group\n    - name: Create Resource Group\n      run: az group create -n ${{env.ENV_RESOURCEGROUP}} -l ${{env.ENV_RESOURCEGROUPLOCATION}}\n\n    #Validate the  ARM template\n    - name: Validate ARM template\n      run: |\n        az deployment group validate -g ${{env.ENV_RESOURCEGROUP}} \u002d\u002dmode Incremental \u002d\u002dtemplate-file ./src/Deployment/azure-logicapp-demo.deploy.json \u002d\u002dparameters ./src/Deployment/azure-logicapp-demo.deploy.parameters.json\n    \n    #Deploy the ARM Template\n    - name: Deploy Logic App\n      run: |\n       echo \u0022::set-env name=logicappurl::$(az deployment group create -g az-logicapp-githubactions-demo-rg \u002d\u002dtemplate-file ./src/Deployment/azure-logicapp-demo.deploy.json \u002d\u002dparameters ./src/Deployment/azure-logicapp-demo.deploy.parameters.json \u002d\u002dquery 'properties.outputs.logicAppUrl.value' -o tsv)\u0022\n      shell: bash   \n      \n    # Log Out From Azure \n    - name: Logout\n      run: az logout\n    \n    - name: set up node\n      uses: actions/setup-node@v1\n      with:\n          node-version: '12.x'\n\n    - name: Install newman\n      run: npm install -g newman\n    \n    - name: Run Postman collection\n      run: |\n       newman run ./src/Test/logicapp-githubactions-test-collection.json -e ./src/Test/logicapp-githubactions-test-env.json \u002d\u002denv-var \u0022url= ${{env.logicappurl}}\u0022\n\n```"} -->
<div class="wp-block-jetpack-markdown"><h2>The Workflow File</h2>
<p>We start by adding the trigger on which our GitHub Action workflow will run. This is done in the <code>yaml</code> file as shown below</p>
<pre><code class="language-yaml">#Name of the GitHub Action
name: Deploy Logic App

#Set the action on which the workflow will trigger
on:
 push:
   branches:
     - master

</code></pre>
<p>Above lines of code define the name of the workflow and the that it will run when a <code>push</code> event occurs on the <code>master</code> branch of the repository.</p>
<p>Next we create the job and  define the environment variables that are used and the runner on which the job will be run.</p>
<pre><code class="language-yaml">jobs:
  validate-and-deploy:
   runs-on: ubuntu-latest
   env:
     ENV_RESOURCEGROUP: az-logicapp-githubactions-demo-rg
     ENV_RESOURCEGROUPLOCATION: australiaeast
</code></pre>
<p>This piece of code will create  a job called <code>validate-and-deploy</code> and this will be run on a <code>ubuntu</code> runner. Also the value of the <code>resource-group</code> and <code>location</code> variables are also set so as to be available across the entire job.</p>
<p>Now we start defining the individual steps that will be executed inside the job.</p>
<h3>Checking out the code</h3>
<p>The code is checked out using following set of command</p>
<pre><code class="language-yaml">   steps:

    #Checkout the repository
    - name: Checkout Repository
      uses: actions/checkout@master
</code></pre>
<p><code>actions/checkout@master</code> is the action available across all the runners and we can use actions offered by GitHub or developed by community.</p>
<h3>Logging into Azure</h3>
<p>We login to the Azure using the credentials we saved while creating the service principal along with the <code>azure/login@v1</code> action available from GitHub community. The code is shown below.</p>
<pre><code class="language-yaml">#Use the azure provided action to log on to azure using service principal
    - name: Login to Azure
      uses: azure/login@v1
      with:
          creds: ${{secrets.AZURE_CREDENTIALS}}
</code></pre>
<p>Notice how the credentials are accessed using <code>secrets.AZURE_CREDENTIALS</code></p>
<h3>Creating the Resource Group</h3>
<p>The resource group is created using following</p>
<pre><code class="language-yml"># Create the resource group
    - name: Create Resource Group
      run: az group create -n ${{env.ENV_RESOURCEGROUP}} -l ${{env.ENV_RESOURCEGROUPLOCATION}}
</code></pre>
<h3>Validating and Deploying the ARM Template</h3>
<p>The ARM templates are validated and deployed using the <code>az deployment</code> CLI commands. The can be found at <a href="%22https://docs.microsoft.com/en-us/cli/azure/deployment?view=azure-cli-latest%22," title="Az Deployment CLI">Az Deployment CLI</a></p>
<p>The code for validation and the deployment is as following.</p>
<pre><code class="language-yml">#Validate the  ARM template
    - name: Validate ARM template
      run: |
        az deployment group validate -g ${{env.ENV_RESOURCEGROUP}} --mode Incremental --template-file ./src/Deployment/azure-logicapp-demo.deploy.json --parameters ./src/Deployment/azure-logicapp-demo.deploy.parameters.json
    
    #Deploy the ARM Template
    - name: Deploy Logic App
      run: |
       echo &quot;::set-env name=logicappurl::$(az deployment group create -g az-logicapp-githubactions-demo-rg --template-file ./src/Deployment/azure-logicapp-demo.deploy.json --parameters ./src/Deployment/azure-logicapp-demo.deploy.parameters.json --query 'properties.outputs.logicAppUrl.value' -o tsv)&quot;
      shell: bash   

</code></pre>
<p>Here we are also saving the logic app trigger URL by reading it from the <code>az deployment group create</code> CLI command and saving it into a variable called <code>logicappurl</code>. It should be noted that though the value is set in the <code>Deploy Logic App</code> step, it will not be accessible to the actions in this step, all following steps will have access to the <code>logicappurl</code> variable.</p>
<h3>Logging Out</h3>
<p>Once the deployment is done, the workflow will log out of the azure. It can be done as shown below.</p>
<pre><code class="language-yml"># Log Out From Azure 
    - name: Logout
      run: az logout
</code></pre>
<p>This completes the deployment part of the workflow, now we want to run some integration tests against the Logic App that was deployed. As mentioned earlier, I used POSTMAN collections to test the API.</p>
<h3>Installing Node and Newman</h3>
<p>POSTMAN can be run inside the CI CD agents using command prompt utility called <a href="https://learning.postman.com/docs/postman/collection-runs/command-line-integration-with-newman/,%22%60NewMan%60%22"><code>Newman</code></a>. To install newman we first need to install <code>node.js</code> on the runner. It is done as shown below.</p>
<pre><code class="language-yml"># Install Node.js environment
- name: set up node
      uses: actions/setup-node@v1
      with:
          node-version: '12.x'

# Install newman
    - name: Install newman
      run: npm install -g newman
</code></pre>
<h3>Running POSTMAN collections</h3>
<p>Once <code>newman</code> is installed, it becomes easy to run the postman collection. It is done as shown in following code snippet.</p>
<pre><code class="language-yml">- name: Run Postman collection
      run: |
       newman run ./src/Test/logicapp-githubactions-test-collection.json -e ./src/Test/logicapp-githubactions-test-env.json --env-var &quot;url= ${{env.logicappurl}}&quot;
</code></pre>
<p>It is evident why we required the <code>logicappurl</code> variable in the first place ;)</p>
<p>This completes our workflow. The entire workflow definition as a <code>yaml</code> file is shown below.</p>
<pre><code class="language-yml">#Name of the GitHub Action
name: Deploy Logic App

#Set the action on which the workflow will trigger
on:
 push:
   branches:
     - master

jobs:
  validate-and-deploy:
   runs-on: ubuntu-latest
   env:
     ENV_RESOURCEGROUP: az-logicapp-githubactions-demo-rg
     ENV_RESOURCEGROUPLOCATION: australiaeast

   steps:

    #Checkout the repository
    - name: Checkout Repository
      uses: actions/checkout@master

    #Use the azure provided action to log on to azure using service pricipal
    - name: Login to Azure
      uses: azure/login@v1
      with:
          creds: ${{secrets.AZURE_CREDENTIALS}}
    
    # Create the resource group
    - name: Create Resource Group
      run: az group create -n ${{env.ENV_RESOURCEGROUP}} -l ${{env.ENV_RESOURCEGROUPLOCATION}}

    #Validate the  ARM template
    - name: Validate ARM template
      run: |
        az deployment group validate -g ${{env.ENV_RESOURCEGROUP}} --mode Incremental --template-file ./src/Deployment/azure-logicapp-demo.deploy.json --parameters ./src/Deployment/azure-logicapp-demo.deploy.parameters.json
    
    #Deploy the ARM Template
    - name: Deploy Logic App
      run: |
       echo &quot;::set-env name=logicappurl::$(az deployment group create -g az-logicapp-githubactions-demo-rg --template-file ./src/Deployment/azure-logicapp-demo.deploy.json --parameters ./src/Deployment/azure-logicapp-demo.deploy.parameters.json --query 'properties.outputs.logicAppUrl.value' -o tsv)&quot;
      shell: bash   
      
    # Log Out From Azure 
    - name: Logout
      run: az logout
    
    - name: set up node
      uses: actions/setup-node@v1
      with:
          node-version: '12.x'

    - name: Install newman
      run: npm install -g newman
    
    - name: Run Postman collection
      run: |
       newman run ./src/Test/logicapp-githubactions-test-collection.json -e ./src/Test/logicapp-githubactions-test-env.json --env-var &quot;url= ${{env.logicappurl}}&quot;

</code></pre>
</div>
<!-- /wp:jetpack/markdown -->

<!-- wp:jetpack/markdown {"source":"# Results\n\nOnce some code changes are pushed, the workflow will fire up automatically and in happy case deploy and successfully test the logic app. The output of each build is available on the Actions pane of the repository and we can drill down into each flow to see why it failed. In case of the failure, by default a notification is sent to the owner of the repository to notify them of the failed build. Following images show the successful as well as the failed results."} -->
<div class="wp-block-jetpack-markdown"><h1>Results</h1>
<p>Once some code changes are pushed, the workflow will fire up automatically and in happy case deploy and successfully test the logic app. The output of each build is available on the Actions pane of the repository and we can drill down into each flow to see why it failed. In case of the failure, by default a notification is sent to the owner of the repository to notify them of the failed build. Following images show the successful as well as the failed results.</p>
</div>
<!-- /wp:jetpack/markdown -->

<!-- wp:image {"id":570} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2020/05/Workflow-Run-1024x299.jpg" alt="" class="wp-image-570"/><figcaption>List of all the workflow instances</figcaption></figure>
<!-- /wp:image -->

<!-- wp:image {"id":571} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2020/05/Successful-Run-1024x413.jpg" alt="" class="wp-image-571"/><figcaption>Successful run</figcaption></figure>
<!-- /wp:image -->

<!-- wp:image {"id":572} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2020/05/Failed-Job-1024x403.jpg" alt="" class="wp-image-572"/><figcaption>Failed run</figcaption></figure>
<!-- /wp:image -->

<!-- wp:image {"id":573} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2020/05/Failed-Job-Notification-1024x407.jpg" alt="" class="wp-image-573"/><figcaption>Failed run notification</figcaption></figure>
<!-- /wp:image -->

<!-- wp:jetpack/markdown {"source":"\nThe run status can be beautified on the `readme.md` file as shown below.\n"} -->
<div class="wp-block-jetpack-markdown"><p>The run status can be beautified on the <code>readme.md</code> file as shown below.</p>
</div>
<!-- /wp:jetpack/markdown -->

<!-- wp:image {"id":574} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2020/05/Passing-Build-1024x286.jpg" alt="" class="wp-image-574"/><figcaption>Passing Run badge</figcaption></figure>
<!-- /wp:image -->

<!-- wp:image {"id":575} -->
<figure class="wp-block-image"><img src="https://theabodeofcode.com/wp-content/uploads/2020/05/Failing-Build-1024x248.jpg" alt="" class="wp-image-575"/><figcaption>Failing Run Badge</figcaption></figure>
<!-- /wp:image -->

<!-- wp:jetpack/markdown {"source":"# Wrapping Up\n\nIn this post we saw how easily we can set up the GitHub Actions workflow to deploy and test the Logic Apps. All we need is a basic understanding of the GitHub Actions and `yaml`. The workflow discussed today is a basic workflow and it does not cover the failure strategy, I will discuss that in future posts."} -->
<div class="wp-block-jetpack-markdown"><h1>Wrapping Up</h1>
<p>In this post we saw how easily we can set up the GitHub Actions workflow to deploy and test the Logic Apps. All we need is a basic understanding of the GitHub Actions and <code>yaml</code>. The workflow discussed today is a basic workflow and it does not cover the failure strategy, I will discuss that in future posts.</p>
</div>
<!-- /wp:jetpack/markdown -->