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

2. **Commit & Run** your updated Pipeline and note the input prompt during the ```Deploy``` stage.  This input prompt is also available in the Console log and classic Stage View.<p><img src="img/2-input-basic.png" width=550/>

>NOTE: Jenkins will wait indefinitely for a user response. Let's fix that by setting a timeout.

3. Set a `timeout` for the `stage` using the `stage` `options` directive by updating the **Deploy** `stage` to match the following:

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