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
6. Select **All** from the **Strategy** drop down under **Discover Branches** <p><img src="img/intro/org_folder_scm_config.png" width=500/>
7. **DON'T SAVE**

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
2. Under **Project Recognizers** select **Custom Script**
3. In **Marker file** type `.nodejs-app`
4. Under **Pipeline** - **Definition** select **Pipeline script from SCM**
5. Select **Git** from **SCM**
6. In **Repository URL** enter: `https://github.com/{your_org_name}/custom-marker-files`
7. Select the credentials you created in the previous exercise.
8. In **Script path** enter: `nodejs-app/Jenkinsfile.template`. Other than the GitHub Organization name it should look like the following:
<p><img src="img/intro/org_folder_marker_config.png" width=500/>
9. Click on **Save**
10. Click on **Scan Organization Now**

When the scan is complete your **Github Organization** project should be **empty**! But, when the project was created it also should have created a webhook in Github. Verify that the webhook was created in Github by checking **Webhooks** within your Organization's Github **Settings**.

> **NOTE:** The ***custom-marker-files*** repository does not get added to your **Github Organization** project since in doesn't contain a matching marker file yet: `.nodejs-app`.

## Basic Declarative Syntax Structure

In this exercise we update your `nodejs-app/Jenkinsfile.template` Declarative Pipeline using the GitHub editor so that it will actually do something!

Declarative Pipelines must be enclosed within a `pipeline` block and must contain a top-level `agent` declaration, and then must contain exactly one `stages` block. The `stages` block must have at least one `stage` block but can have an unlimited number of additional stages. Each `stage` block must have exactly one `steps` block. The Blue Ocean editor takes care of much of this for you but we will need to add a `stage` and `steps`.

Using the Blue Ocean Pipeline editor we setup in the previous exercise, do the following:

1. You should see the **You don't have any branches that contain a Jenkinsfile** dialog, click on the **Create Pipeline** button (NOTE: If you already had a `Jenkinsfile` in your repository then the editor should open straight-away) <p><img src="img/1-1-create-pipeline-no-jenkinsfile.png" width=300/>
2. Click on the **+** icon next to the pipeline's **Start** node to add a `stage` <p><img src="img/1-basic-syntax-add-stage.png" width=400/>
3. Click into **Name your stage** and type in the text 'Say Hello'
4. Click on the **+ Add step** button <p><img src="img/1-basic-syntax-add-step.png" width=400/>
5. Click on the **Print Message** step and type in ***Hello World!*** as the **Message**
6. Click on the **<-** (arrow) next to the **'Say Hello / Print Message'** text to add another step <p><img src="img/1-1-print-message-then-add-step.png" width=300/>
7. Click on the **+ Add step** button
8. Click on the **Shell Script** step and enter `java -version` into the text area
9. Press the key combination `CTRL + S` to open the Blue Ocean free-form editor and you should see a Pipeline similar to the one below **(click anywhere outside the editor to close it)**:

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

10. Click the **Save** button <p><img src="img/1-1-shell-step-save.png" width=350/>
11. Enter a commit message into the **Save Pipeline** pop up and click **Save & Run** <p><img src="img/1-basic-syntax-save-run.png" width=350/>
  
>NOTE: You may have noticed that your Pipeline GitHub repository is being cloned even though you didn't specify that in your Jenkinsfile. Declarative Pipeline checks out source code by default using the `checkout scm` step.

## Jenkins Kubernetes Agents

In this exercise we will get an introduction to the [Jenkins Kubernetes plugin](https://github.com/jenkinsci/kubernetes-plugin) and use the `container` block to run a set of Pipelein `steps` inside a Docker container set-up in a Jenkins Kubernetes Agent Pod template. Initially we only used `agent any`. But now we want to use Node.js and our CloudBees Core Jenkins Administrator has configured the CloudBees Kubernetes Shared Cloud to include an Agent Pod template to provide a Node.js container.

1. Replace the **Testing** stage with the following stage:

```
      stage('Testing') {
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