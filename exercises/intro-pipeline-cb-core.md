# Introduction to Pipelines with CloudBees Core

## GitHub Organization Project

In this exercise we are going to create a special type of Jenkins Pipeline project referred to as an *Organization Folder* and more specifically a *GitHub Organization* project. The *GitHub Organization* project will scan a GitHub Organization to discover the organization’s repositories, automatically creating **managed** *Multibranch Pipeline* jobs for any repository with at least one branch containing a *project recognizer* - typically **Jenkinsfile**. We will use the GitHub Organization that you created in **[Setup - Create a GitHub Organization](./Setup.md#create-a-github-organization)**.

First, let's add your GitHub credentials to the Jenkins' Credentials manager:

1. Navigate back to your personal folder in Jenkins
2. Click on **Credentials**
3. Click on the **(global)** link under **Stores Scoped to [YourFolderName]** (in this case **beedemo-dev**) <p><img src="img/intro/credential_folder_scope.png" width=500/>
4. Click on **Add Credentials** in the left menu
5. Fill out the form (**Username with password**)
  - **Username**: The GitHub user name
  - **Password**: Your GitHub personal access token [created in setup](../Setup.md#create-a-github-personal-access-token)
  - **ID**: Create an ID for your credentials (something like **yourorg-github-token**)
  - **Description**: Can be left blank if you want <p><img src="img/intro/credential_github_token_save.png" width=500/>
6. Click on **OK**

Now let's create the Github Organization project:

1. Click on **New Item**
2. Enter your GitHub Organization name as the **Item Name** 
3. Select **GitHub Organization** <p><img src="img/intro/org_folder_item.png" width=500/>
4. Click **Ok**
5. Select the credentials you created above from the **Credentials** drop down
6. Select **All** from the **Strategy** drop down under **Discover Branches** 
7. Make sure that the **Owner** field matches the name of your GitHub Organization name. <p><img src="img/intro/org_folder_scm_config.png" width=500/>
8. **DON'T SAVE YET**

Continue to the next exercise.

## Custom Marker Files

In this exercise we are going to demonstrate how you can use the **[Custom Marker feature](https://go.cloudbees.com/docs/cloudbees-documentation/cje-user-guide/#pipeline-custom-factories)** of CloudBees Core to associate an externally managed Pipeline script to a job based on any arbitrary **marker file** like `pom.xml` or something slightly more declarative like `.nodejs-app`.

In order to complete the following exercise you should have [forked the following repositories](../Setup.md#fork-the-workshop-repositories) into the Github Organization you created in **[Setup - Create a GitHub Organization](../Setup.md#create-a-github-organization)**:

1. https://github.com/cloudbees-cd-acceleration-workshop/custom-marker-pipelines 
2. https://github.com/cloudbees-cd-acceleration-workshop/helloworld-nodejs 

Your GitHub Organization should look like the following:
<p><img src="img/intro/fork_repos_into_org.png" width=500/>

Once those repositories are forked:

1. In the Github organization project you started to create in the previous exercise scroll down to the **Project Recognizers** section.
2. Delete the default **Project Recognizer** **Pipeline Jenkinsfile**. <p><img src="img/intro/custom_marker_delete_default.png" width=500/>
3. Next, under **Project Recognizers** click the **Add** button and select **Custom Script** <p><img src="img/intro/custom_marker_add_custom_script.png" width=400/>
3. In **Marker file** type `.nodejs-app`
4. Under **Pipeline** - **Definition** select **Pipeline script from SCM**
5. Select **Git** from **SCM**
6. In **Repository URL** enter: `https://github.com/{your_org_name}/custom-marker-files`
7. Select the credentials you created in the previous exercise.
8. In **Script path** enter: `nodejs-app/Jenkinsfile.template`. Other than the GitHub Organization name it should look like the following: <p><img src="img/intro/custom_marker_config.png" width=550/>
9. Click on **Save**
11. When the scan is complete your **Github Organization** project should be **empty**! <p><img src="img/intro/custom_marker_empty.png" width=500/> <p>But, when the project was created it also should have created a webhook in Github. Verify that the webhook was created in Github by checking **Webhooks** within your Organization's Github **Settings**. <p><img src="img/intro/custom_marker_org_webhook.png" width=500/>
12. In your forked copy of the **helloworld-nodejs** repository click on the **Add file** button towards the top right of the screen. <p><img src="img/intro/custom_marker_create_file.png" width=500/>
13. Name the file `.nodejs-app` - no need to add any content - and click the **Commit new file** button at the bottom of the screen to save it your repository master branch.
14. Navigate back to your GitHub Organization Folder project on your CloudBees Team Master and voila - you have a new [Pipeline Multibranch project](https://jenkins.io/doc/book/pipeline/multibranch/) mapped to the the **helloworld-nodejs** repository thanks to the the GitHub Organization webhook that was created when we first save the GitHub Organization Folder project. Notice how the **helloworld-nodej** Multibranch Pipeline project's description came from the GitHub repository description. <p><img src="img/intro/custom_marker_multibranch.png" width=500/>

> **NOTE:** The ***custom-marker-files*** repository does not get added to your **Github Organization** project since in doesn't and will never contain a matching marker file: `.nodejs-app`.

## Basic Declarative Syntax Structure

In this exercise we will update the `nodejs-app/Jenkinsfile.template` Declarative Pipeline using the GitHub editor so that it will actually do something as opposed to resulting in the following errors:

```
WorkflowScript: 1: Missing required section "stages" @ line 1, column 1.
   pipeline {
   ^

WorkflowScript: 1: Missing required section "agent" @ line 1, column 1.
   pipeline {
   ^

2 errors
```

Declarative Pipelines must be enclosed within a `pipeline` block - which we have. But they must also contain a top-level `agent` declaration, and must contain exactly one `stages` block. The `stages` block must have at least one `stage` block but can have an unlimited number of additional `stage` blocks. Each `stage` block must have exactly one `steps` block. 

1. We will use the GitHub file editor to update the `nodejs-app/Jenkinsfile.template` file in your forked **customer-marker-pipelines** directory. Navigate to the `custom-marker-pipelines/nodejs-app/Jenkinsfile.template` file in your forked repository and then click on the pencil icon to the upper right to edit that file. <p><img src="img/intro/basic_snytax_edit_github.png" width=500/>
2. Replace the contents of that file with the following Declarative Pipeliine:
```
pipeline {
   agent any
   stages {
      stage('Say Hello') {
         steps {
            echo 'Hello World!'   
            sh 'java -version'
         }
      }
   }
}
```
3. Add a commit description and then click the **Commit Changes** button with the default selection of *Commit directly to the master branch* selected.
4. Navigate back to the **helloworld-nodejs** *master* branch job on your Team Master and click the **Build Now** button in the left menu.
5. Your job should complete successfully. Note some things from the log:
  
  1. The custom marker script - `nodejs-app/Jenkinsfile.template` - is being pulled from your forked *custom-marker-pipelines* forked repository:
```
...
Obtained nodejs-app/Jenkinsfile.template from git https://github.com/cd-accel-beedemo/custom-marker-pipelines.git
...
```
  2. The agent is being provisioned from a Kubernetes Pod Template (more on this in the next lesson):
```
...
Agent default-jnlp-0p189 is provisioned from template Kubernetes Pod Template
...
```
  3. Your fork of the **helloworld-nodejs** repository is being checked out, even though you did not put any steps in the `Jenkinsfile.template` to do so:
```
...
Cloning repository https://github.com/cd-accel-beedemo/helloworld-nodejs.git
...
```
  4. The agent has a Java version of `1.8.0_171`:
```
...
Running shell script
+ java -version
openjdk version "1.8.0_171"
...
```
  
>NOTE: You may have noticed that your Pipeline GitHub repository is being cloned even though you didn't specify that in your Jenkinsfile. Declarative Pipeline checks out source code by default using the `checkout scm` step.

## Kubernetes Agents for CloudBees Core

In this exercise we will get an introduction to the [Jenkins Kubernetes plugin](https://github.com/jenkinsci/kubernetes-plugin/blob/master/README.md) for running dynamic and ephemeral agents in a Kubernetes cluster - leveraging the scaling abilities of Kubernetes to schedule build agents.

CloudBees Core has built-in support for Kubernetes build agents. The agents are contained in pods, where a pod is a group of one or more containers sharing a common storage system and network. A pod is the smallest deployable unit of computing that Kubernetes can create and manage (you can read more about pods in the [Kubernetes documentation](https://kubernetes.io/docs/concepts/workloads/pods/pod/)).

>NOTE: One of the containers hosts the actual Jenkins build agent (the slave.jar file). By convention, this container always exists, and has the name jnlp and default execution always goes to the jnlp container (as it did when we used `agent any` above). If needed, this may be overridden by specifying a **Container Template** with the ***Name*** `jnlp`.

We will use the Pipeline `container` block to run Pipeline `steps` inside a specific container configured as part of a Jenkins Kubernetes Agent Pod template. In our initial Pipeline, we used `agent any` which required at least one Jenkins agent configured to *Use this node as much as possible* - resulting in the use of a Pod Template that only had a JNLP container. But now we want to use Node.js in our Pipeline. Luckily, our CloudBees Core Jenkins Administrator has configured the [CloudBees Core Kubernetes Shared Cloud](https://go.cloudbees.com/docs/cloudbees-core/cloud-admin-guide/agents/#_globally_editing_pod_templates_in_operations_center) to include a Kubernetes Pod Template to provide a Node.js container. <p><img src="img/intro/k8s_agent_nodejs_template.png" width=500/> <p>Take note of the ***Labels*** field and the **Container Template** ***Name*** field - both of these are important.

1. First, we need to update the `agent any` directive with the following so that we will get the Kubernetes Pod Template with the containers we need for our Pipeline:
```
  agent 
```
2. Next, replace the **Say Hello** `stage` with the following stage:

```
      stage('Test') {
        parallel {
          stage('Java 8') {
            agent { label 'jdk9' }
            steps {
              container('maven8') {
                sh 'mvn -v'
              }
            }
          }
          stage('Java 9') {
            agent { label 'jdk8' }
            steps {
              container('maven9') {
                sh 'mvn -v'
              }
            }
          }
        }
      }
```
  You see above that the `container` block runs all of the steps within that block in the specified container based on the **name** of the **Container Template** - so it doesn't matter what version of the JDK is running in the agent.
2. Commit your change to the master branch.
3. Compare the logs for the **Java 8** and **Java 9** stages.

## Conditional Execution

In this exercise we are going to edit the Jenkinsfile file in the **development** branch of our project to add a branch specific stage.

**Important Note** The following code demonstrates a new set of features added to Declarative Pipeline in Version 1.2.6:

  - Add beforeAgent option for when - if true, when conditions will be evaluated before entering the agent.

In order to complete the following exercise you will need to fork the following repository into the Github Organization you created in **[Setup - Create a GitHub Organization](./Setup.md#create-a-github-organization)**:

* https://github.com/PipelineHandsOn/sample-rest-server

After forking the repository, click "Scan Organization Now" in your Github organization project.  The new project should be discovered and will start building.

1. Within your **sample-rest-server** project select the **development** branch from the **Branch** drop down menu
2. Click on the **Jenkinsfile** in the file list
3. Click on the **Edit this file** button (pencil)
4. Insert the following stage after the existing **build** stage:

```
      stage('Development Tests') {
         when {
            beforeAgent true
            branch 'development'
         }
         steps {
            echo "Run the development tests!"
         }
      }
```

5. Fill out the commit information, select 'Commit directly to the development branch.', and click on **Commit Changes**

>Notice how after you commit your changes the Github web hooks trigger a build of the development branch in Jenkins.

## PRs and Merging

In this exercise we are going to edit the development branch's Jenkinsfile again but make our commit against a feature branch and use a pull request to merge the edits into our development branch.

1. Click on the **Edit this file** button (pencil)
2. Insert the following stage after the existing **build** stage:

```
      stage('Masters Tests') {
         when {
            branch 'master'
         }
         steps {
            echo "Run the master tests!"
         }
      }
```

3. Fill out the commit information, select 'Create a new branch for this commit and start a pull request.' and click on **Propose file change**
4. Flip back to your Jenkins job and notice that the new feature branch appears in your projects
5. Return back to the Github **Open a pull request** page
6. Click on the **Create pull request** button
7. Go to your Jenkins job and notice that that the PR has been added to the Pull Requests tab
8. In Github click on **Merge pull request** and then **Confirm** to close the PR and merge the results into the development branch
9. Optionally you can also delete the feature branch you created

Finally, we should merge our work into our master branch to verify that our changes work there:

1. Return back to your repository's main page where you will be on the master branch by default
2. Click on **New pull request**
3. Select your base fork (not the project we forked from)
4. Compare **master** to **development** and resolve any conflicts.
5. Click **View pull request**
6. Click **Merge pull request**
7. Click **Confirm merge**

>Notice how after you merge your changes into master the Github web hooks trigger a build of the master branch in Jenkins.