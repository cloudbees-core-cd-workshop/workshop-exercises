# More Declarative Syntax with CloudBees Core

## Interactive Input

For this exercise we are going to add a new stage after the **Build and Push Image** stage that will demonstrate how to pause a Pipeline job and prompt for interactive input. 

>NOTE: The declarative `input` directive blocks the `stage` from executing and acquiring an agent.

1. Use the GitHub file editor to update the `nodejs-app/Jenkinsfile.template` file in your forked **customer-marker-pipelines** repository - adding the following `stage` to your Pipeline after the ***Build and Push Image*** `stage`:

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

2. **Commit & Run** your updated Pipeline and note the input prompt during the `Deploy` stage.  This `input` prompt is also available in the Console log and classic Stage View.<p><img src="img/more/input_basic.png" width=550/>

3. Your Team Master will wait indefinitely for a user response. Let's fix that by setting a timeout. Earlier we used `options` at the global `pipeline` level to  Set a `timeout` for the `stage` using the `stage` `options` directive by updating the **Deploy** `stage` to match the following:

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

4. **Commit & Run** your pipeline and wait at least 30 seconds. Your pipeline should be automatically aborted.<p><img src="img/2-input-timeout.png" width=550/>

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

1. 

## Parallel Stages

