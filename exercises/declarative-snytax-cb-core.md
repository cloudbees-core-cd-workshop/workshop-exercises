# More Declarative Syntax with CloudBees Core

## Interactive Input

For this exercise we are going to add a new stage after the **Build and Push Image** stage that will demonstrate how to pause a Pipeline job and prompt for interactive input. 

>**NOTE:** The [Declarative `input` directive](https://jenkins.io/doc/book/pipeline/syntax/#input) blocks the `stage` from acquiring an agent.

1. Use the GitHub file editor to update the `nodejs-app/Jenkinsfile.template` file in your forked **custom-marker-pipelines** repository - adding the following `stage` to your Pipeline after the ***Build and Push Image*** `stage` and commit the change:

```
    stage('Deploy') {
      input {
        message "Should we continue?"
      }
      steps {
        echo "Continuing with deployment"
      }
    }
```

2. **Run** your updated Pipeline job in Blue Ocean and note the `input` prompt during the `Deploy` stage.  *This `input` prompt is also available in the Console log and classic Stage View.* <p><img src="img/more/input_basic.png" width=550/>

3. Your Team Master will wait indefinitely for a user response to an `input` step. Let's fix that by setting a timeout. Earlier we used `options` at the global `pipeline` level to set the *Discard old builds* strategy for your Team Master with the `buildDiscarder` `option`. Now we will configure `options` at the `stage` level -a  `timeout` for the `stage` using the `stage` `options` directive. Update the **Deploy** `stage` to match the following and commit the changes:

```
    stage('Deploy') {
      options {
        timeout(time: 30, unit: 'SECONDS') 
      }
      input {
        message "Should we continue?"
      }
      steps {
        echo "Continuing with deployment"
      }
    }
```

4. **Run** your updated Pipeline job in Blue Ocean and wait at least 30 seconds. Your pipeline should be automatically **aborted** after 30 seconds after the 'Deploy' `stage` starts.<p><img src="img/more/input_timeout.png" width=550/> <p>Run it again if you would like - but this time approving it before 30 seconds expires - the job should complete successfully.

## Input Approval for Team Members

The `input` directive supports a [number of interesting configuration options](https://jenkins.io/doc/book/pipeline/syntax/#configuration-options). In this exercise we are going to use the `submitter` option to control what Team Master member is allowed to submit the `input` directive. But first we need to add another member to our CloudBees Team Master. Team Masters provide an easy to use authorization model right out-of-the-box. The following roles are available (there is a CLI to add or modify roles):

- **Team Admin:** administrator of the Team Master.
- **Team Member:** read, write and execute permission on the pipelines.
- **Team Guest:** read only.

We want to add a **Team Guest** to our Team Masters and the set that Team member as the `submitter` for our `input` directive. Before you beging pick a person next to you and share each other's Jenkins account name with each other. You will use that account name when added a new member to your Team Master.

1. On your Team Master, navigate to the Team list by clicking on the ***Administration*** link on the top right (this link is available on all Blue Ocean pages accept for the [Pipeline Run Details view](https://jenkins.io/doc/book/blueocean/pipeline-run-details/#pipeline-run-details-view)). <p><img src="img/more/input_submitter_admin_link.png" width=600/>
2. Next, click on the cog icon for your team.  <p><img src="img/more/input_submitter_team_cog.png" width=500/>
3. Click on the ***Members*** link in the left menu and then click on the ***Add a user or group*** link. <p><img src="img/more/input_submitter_members_link.png" width=600/>
4. select **Team Guest** from the role drop-down, enter the account name for the person next to you in the ***Add user or group*** input (I will use **beedemo-ops**), press your **enter/return** key, and then click the **Save changes** button.  <p><img src="img/more/input_submitter_add_team_guest.png" width=600/>

Now that we have a new team member, you can add them as a `submitter` for the `input` directive in your `nodejs-app/Jenkinsfile.template` Pipeline script.

1. 

## Post Actions

What happens if your `input` step times out or if the *approver* clicks the **Abort** button? There is a special `post` section for Delcarative Pipelines that allows you to define one or more additional steps that are run upon the completion of a Pipeline’s or stage’s execution and are designed to handle a variety of conditions (not only **aborted**) that could occur outside the standard pipeline flow.

In this example we will add a `post` section to our **Deploy** stage to handle a time out (aborted run). 

>NOTE: The `post` section is available at either the global `pipeline` level or at individual `stage` levels.

1. Add the following to the bottom of your `pipeline` - right before the close curly brace for the entire `pipeline` using the GitHub editor:

```
  post {
    aborted {
      echo 'Why didn\'t you push my button?'
    }
  }
```

2. Commit your changes.
3. Run your pipeline from the **Branches** view of the Blue Ocean Activity View for your pipeline.
4. Wait for 30 seconds and you should see the following line in your console output: `Why didn't you push my button?` **OR** you could just click the **Abort** button.<p><img src="img/2-post-action-abort.png" width=550/>
5. Finally, remove the `Deploy` stage from your pipeline so that you will not have to manually approve the job each time it runs for the rest of the workshop.

## Restartable Stages

Declarative Pipelines support a feature referred to as ***Restartable Stages***. You can restart any completed Declarative Pipeline from any top-level `stage` which ran in that Pipeline. This allows you to re-run a Pipeline from a `stage` which failed due to transient or environmental reasons - among other reasons. All inputs to the Pipeline will be the same. This includes SCM information, build parameters, and the contents of any `stash` step calls in the original Pipeline, if specified. Stages which were skipped due to an earlier failure will not be available to be restarted, but `stages` which were skipped due to a `when` condition not being satisfied will be available.

1. Navigate to the **master** branch of your **helloworld-nodejs** job in Blue Ocean on your Team Master and run the job.
2. 

## Parallel Stages



## Sequential Stages



