# Advanced Pipelines with CloudBees Core

## Shared Libraries

In this exercise we are going to add a *step* to our Pipeline from a [**Pipeline Shared Library**](https://jenkins.io/doc/book/pipeline/shared-libraries/), providing functionality to set default values based on default Jenkins environmental variables. But first, you will fork the Pipeline Shared Library for this exercise from https://github.com/cloudbees-cd-acceleration-workshop/pipeline-library into the GitHub Organization you created in **[Setup - Create a GitHub Organization](./Setup.md#create-a-github-organization)**.

Once you have forked the ***pipeline-library*** repository into your GitHub Organization you will need to configure it as a Pipeline Shared Library configuration for your Team master. Pipeline Shared Libraries may be configured at the Jenkins Master level or the Jenkins folder level. The GitHub Oranization project that you created earlier is actually a special type of folder, so we will add the ***pipeline-library*** to that folder.

1. In the ***Github Organization** folder Jenkins project you started to create in the previous exercise scroll down to the **Pipeline Libraries** section and click the **Add** button. <p><img src="img/intro/advanced_shared_lib_add.png" width=550/>
2. Enter `cd-accel` for the Library **Name**, `master` for the **Default version**.
3. Next for the **Retrieval method** seleclt **Modern SCM**.
4. Then, for the **Source Code Management** select **GitHub**.
5. Select the GitHub **Credentials** you created earlier, enter your GitHub Organization name as the **Owner**, select **pipeline-library** for the **Repository** and the click the **Save** button. <p><img src="img/intro/shared_lib_config.png" width=600/>

If you navigate back to your fork of the **pipeline-library** repository you will notice.

Now that the **Pipeline Shared Library** is configured for your Team Master we will use it in the `nodejs-app/Jenkinsfile.template` Pipeline in your forked **customer-marker-pipelines** repository.

1. Add the following line after the top level the `agent` directive:

```
  libraries {
    lib("SharedLibs")
  }
```

2. Then add the following stage after the stage you created in **Exercise 3.1**:

```
      stage('Shared Lib') {
         steps {
             helloWorld("Jenkins")
         }
      }
```

>The `helloWorld` function we are calling can be seen at: https://github.com/PipelineHandsOn/shared-libraries/blob/master/vars/helloWorld.groovy


## Cross Team Collaboration
In this exercise we are going to set-up two Pipeline jobs (using the Jenkins classic UI) that demonstrate CloudBee's Cross Team Collaboration feature. We will need two separate Pipelines - one that publishes an event - and another that is triggered by an event.

### Master Events

For the first part of **Cross Team Collaboration** we will create an event that is only published on your master.

#### Publish Event

First you have to publish an event from a Pipeline - any other Pipeline may set a trigger to listen for this event. Create a Pipeline job named `notify-event` with the following content, but replace `<username>Event` with your username so my event would be `beedemoEvent`:

```
pipeline {
    agent none
    stages {
        stage('Publish Event') {
            steps {
                publishEvent simpleEvent('<username>Event')
            }
        }
    }
}
```

#### Event Trigger

Next, create a Pipeline job name `notify-trigger` and set a `trigger` to listen for the event you created above with the following content, again don't forget to edit `<username>Event`:

```
pipeline {
    agent none
    triggers {
        eventTrigger simpleMatch('<username>Event')
    }
    stages {
        stage('Event Trigger') {
            when {
                expression { 
                    return currentBuild.rawBuild.getCause(com.cloudbees.jenkins.plugins.pipeline.events.EventTriggerCause)
                }
            }
            steps {
                echo 'triggered by published event'
            }
        }
    }
}
```

After creating both of these Pipeline jobs you will need to run the **Event Trigger** job once so that the trigger is registered (similar to what was necessary for job parameters). Once that is complete, click on **Build Now** to run the **Publish Event** job. Once that job has completed, the **Event Trigger** job will be triggered after a few seconds. The logs will show that the job was triggered by an `Event Trigger` and the `when` expression will be true.

>**NOTE**:  If your *trigger* job does not fire, you may need to enable the **Cross Team Collaboration** feature on your master.  Navigate to the top level of your master, select **Manage Jenkins** and then **Configure Notification**.  Next, select **Enable** and choose **Local only** and then click **Save**.

### Cross-Master Events

For the second part of **Cross Team Collaboration** we will create an event that will be published **across Team Masters** via CloudBees Operations Center. The Cross Team Collaboration feature has a configurable router for routing events and we will change the router used for this exercise. You will need to select a partner to work with - one person will be the notifier and the other person will update their **event-trigger** job to be triggered when the notifier's job is run.

1. First you need to update the **Notification Router Implementation** to use the **Operations Center Messaging** router by clicking on the **Manage Jenkins** link - on the left side at the root of your Team Master (classic ui).
2. Next, scroll down and click on **Configure Notification** link.
3. Under **Notification Router Implementation** select the **Operations Center Messaging** option as opposed to the currently selected **Local only** option.
4. Now, the trigger job of the second partner needs to be updated to listen for the notifier's `event` string - ask your partner what their event string is - and then run the job once to register the trigger.
5. Next, the notifier will run their `notify-event` job and you will see the `notify-trigger` job get triggered.

## Advanced Jenkins Kubernetes Agents

In this exercise we will explore the [Jenkins Kubernetes plugin](https://github.com/jenkinsci/kubernetes-plugin) and will create a new pipeline job to use the `podTemplate` and `container` directives in your Declarative Pipeline. The plugin creates a [Kubernetes Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) for each agent requested by a Jenkins job, with at least one Docker container running as the JNLP agent, and stops the pod and all containers after the build is complete (or after a set amount of time as is the case here).

>NOTE: The **Jenksin Kubernetes Plugin** is automatically installed and configured to provision dynamic ephemeral agents by CloudBees Jenkins Enterprise on Kubernetes. 

1. Create a "New Item" and give it a name like "Advanced K8s Example" - choose the Pipeline type click OK.
2. Copy and paste the following code into the **Pipeline Script** text box near the bottom of the page:

```
pipeline {
  agent {
    kubernetes {
      label 'kubernetes'
      containerTemplate {
        name 'go'
        image 'golang:1.10.1-alpine'
        ttyEnabled true
        command 'cat'
      }
    }
  }
  stages {
    stage('golang in k8s') {
        steps {
            container('go') {
                sh 'go version'
            }
        }
    }
  }
}
```
**Note:** Notice the use of the **container** directive.  This tells Jenkins which container in a Pod to use for the steps in the stage.  In this exercise a single container (golang) was explicitly defined in the pipeline, however a second container is implicitly created to handle the JNLP communiction between Jenkins and the Pod.

```
pipeline {
  agent {
    kubernetes {
      label 'kubernetes'
      containerTemplate {
        name 'go'
        image 'golang:1.10.1-alpine'
        ttyEnabled true
        command 'cat'
      }
    }
  }
  stages {
    stage('golang in k8s') {
        steps {
            container('go') {
                sh 'go version'
            }
            container('jnlp') {
                sh 'java -version'
            }
        }
     }
  }
}
```
**Note:** In the above example you were able to execute a java command in the implicitly defined **jnlp** container in the Pod.  The JNLP container is a part of every Pod created by the Jenkins Kubernetes plugin.

