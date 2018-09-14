# Pipeline Approvals and More with CloudBees Core

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

3. Your Team Master will wait indefinitely for a user response to an `input` step. Let's fix that by setting a timeout. Earlier we used `options` at the global `pipeline` level to set the ***Discard old builds*** strategy for your Team Master with the `buildDiscarder` `option`. Now we will configure `options` at the `stage` level -a  `timeout` for the `stage` using the `stage` `options` directive. Update the **Deploy** `stage` to match the following and commit the changes:

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

4. **Run** your updated Pipeline job in Blue Ocean and wait at least 30 seconds once it reaches the 'Deploy' `stage`. Your pipeline should be automatically **aborted** 30 seconds after the 'Deploy' `stage` starts.<p><img src="img/more/input_timeout.png" width=550/> <p>Run it again if you would like - but this time approving it before 30 seconds expires - the job should will successfully.

## Input Approval for Team Members

The `input` directive supports a [number of interesting configuration options](https://jenkins.io/doc/book/pipeline/syntax/#configuration-options). In this exercise we are going to use the `submitter` option to control what Team Master member is allowed to submit the `input` directive. But first you need to add another member to your CloudBees Team Master. Team Masters provide an easy to use authorization model right out-of-the-box. The following roles are available (there is a CLI to add or modify roles):

- **Team Admin:** administrator of the Team Master.
- **Team Member:** read, write and execute permission on the pipelines.
- **Team Guest:** read only.

We want to add a **Team Guest** to our Team Masters and then set that Team member as the `submitter` for our `input` directive. Before you begin, pick a person next to you and share each other's Jenkins account names. You will use that account name when adding a new member to your Team Master below:

1. On your Team Master, navigate to the Team list by clicking on the ***Administration*** link on the top right (this link is available on all Blue Ocean pages accept for the [Pipeline Run Details view](https://jenkins.io/doc/book/blueocean/pipeline-run-details/#pipeline-run-details-view)). <p><img src="img/more/input_submitter_admin_link.png" width=600/>
2. Next, click on the cog icon for your team.  <p><img src="img/more/input_submitter_team_cog.png" width=500/>
3. Click on the ***Members*** link in the left menu and then click on the ***Add a user or group*** link. <p><img src="img/more/input_submitter_members_link.png" width=600/>
4. Select **Team Guest** from the role drop-down, enter the account name for the person next to you in the ***Add user or group*** input (I will use **beedemo-ops**), press your ***enter/return*** key, and then click the **Save changes** button.  <p><img src="img/more/input_submitter_add_team_guest.png" width=500/>
5. Click on the ***Pipelines*** link in the top menu.

Now that we have a new team member, you can add them as a `submitter` for the `input` directive in your `nodejs-app/Jenkinsfile.template` Pipeline script.

1. Use the GitHub file editor to update your `nodejs-app/Jenkinsfile.template` Pipeline script in your forked **custom-marker-pipelines** repository - updating the `input` directive of the **Deploy** `stage` with the following changes (replacing **beedemo-ops** with Jenkins username of your new **Team Guest** member). Also, update the `timeout` duration to give your approver plenty of time to submit the `input`:

```
options {
    timeout(time: 60, unit: 'SECONDS') 
}
input {
    message "Should we deploy?"
    submitter "beedemo-ops"
    submitterParameter "APPROVER"
}
```

2. So, we added one additonal configuration option for our `input` directive: `submitterParameter`. Setting the  `submitterParameter` option will result in a Pipeline environmental variable named `APPROVER` being set with the value being the username of the user that submitted the `input`. In this case it will either be **beedemo-ops** or **SYSTEM** if it timeouts before it is submitted. Update the `echo` step in your `nodejs-app/Jenkinsfile.template` Pipeline script to print the `APPROVER` environmental variable and commit the changes:

```
echo "Continuing with deployment - approved by ${APPROVER}"
```

3. Navigate to the **master** branch of your **helloworld-nodejs** job in Blue Ocean on your Team Master and run the job. If you attempt to approve the `input` you will get an error: <p><img src="img/more/input_submitter_error.png" width=600/>
4. The ***submitter*** needs to navigate to the **master** branch of your **helloworld-nodejs** job on your Team Master to approve the `input` of your **helloworld-nodejs** Pipeline. You can use the *Team switcher* to quickly navigate to another Team Master that you are a member. The *Team switcher* drop-down will appear in the top right of your screen once you have been added as a member to another Team Master. The ***submitter*** needs to switch to the Team where they are a *Team Guest* member by selecting that team from the *Team switcher* drop-down. <p><img src="img/more/input_submitter_team_switcher.png" width=600/>
5. As the ***submitter*** navigate to the **helloworld-nodejs** job on your new team and approve the `input`. Note the output of the `echo` step. <p><img src="img/more/input_submitter_approved_by.png" width=550/>

>**NOTE:** If you select a Pipeline job as a *favorite* you will be able to see things like jobs awaiting `input` submission in the Blue Ocean **Dashboard**. 

<img src="img/more/input_submitter_favorite.png" width=600/>

## Post Actions

What happens if your `input` step times out or if the *approver* clicks the **Abort** button? There is a special [`post` section for Delcarative Pipelines](https://jenkins.io/doc/book/pipeline/syntax/#post) that allows you to define one or more additional steps that are run upon the completion of the entire `pipeline` or an individual `stage` execution and are designed to [handle a variety of conditions](https://jenkins.io/doc/book/pipeline/syntax/#post-conditions) (not only **aborted**) that could occur outside the standard Pipeline flow.

In this example we will add a `post` section to our **Deploy** stage to handle a timeout or disapproval by our submitter (aborted run). 

1. Add the following `post` section just below the `options` directive a the root of your Pipeline using the GitHub editor and commit your changes:

```
pipeline {
  agent { label 'nodejs-app' }
  options { 
    buildDiscarder(logRotator(numToKeepStr: '2'))
  }
  post {
    aborted {
      echo "Why didn't you push my button?"
    }
  }
```

3. Run your pipeline from the **Branches** view of the Blue Ocean Activity View for your pipeline.
4. Let the job timeout or have your `submitter` click the **Abort** button. You will see the following output: <p><img src="img/more/input_post_abort.png" width=550/>

## Restartable Stages

Declarative Pipelines support a feature referred to as [***Restartable Stages***](https://jenkins.io/doc/book/pipeline/running-pipelines/#restart-from-a-stage). You can restart any completed Declarative Pipeline from any top-level `stage` which ran in that Pipeline job run. This allows you to re-run a Pipeline from a `stage` which may have failed due to transient or environmental reasons. All inputs to the Pipeline will be the same. This includes SCM information, build parameters, and the contents of any `stash` step calls in the original Pipeline, if specified. Stages which were skipped due to an earlier failure will not be available to be restarted, but `stages` which were skipped due to a `when` condition not being satisfied will be available.

1. Navigate to the **master** branch of your **helloworld-nodejs** job in Blue Ocean on your Team Master and run the job.
2. 

## Parallel Stages



## Sequential Stages



