# Parallel and Sequential Stages with CloudBees Core

The ability to define stages to run in parallel is an important feature of Jenkins Pipeline jobs. The Declarative Pipeline syntax has extended support for [parallel stages](https://jenkins.io/doc/book/pipeline/syntax/#parallel), [sequential stages](https://jenkins.io/doc/book/pipeline/syntax/#sequential-stages) and nested stages - and all of these features are nicely visualized in the Blue Ocean UI. In this section of exercise, we will use sequential stages and Pipeline parallelization to speed up the execution of the tests we will had for our **helloworld-nodejs** app.

>**Starting Here or Catching Up?**
>
>If you are starting with this set of exercises or just need to catch up, you may get the the correct version of the  **nodejs-app/Jenkinsfile.template** Pipeline script for starting these exercises [from this branch](https://github.com/cloudbees-cd-acceleration-workshop/custom-marker-pipelines/blob/after-approvals/nodejs-app/Jenkinsfile.template).

## Kubernetes Pod Templates Defined in Pipeline Script

So far we have been using the **nodejs-app** [Kubernetes *Pod Template* defined for us on **CloudBees Jenkins Operations Center (CJCO)**](https://go.cloudbees.com/docs/cloudbees-core/cloud-admin-guide/agents/#_globally_editing_pod_templates_in_operations_center). However, for the next set of changes to the **nodejs-app/Jenkinsfile.template** Pipeline script we will need an additional Docker *container* for executing tests and we also want to use a different vesion of the **node** Docker image than the one provided by the CJOC *Kubernetes Shared Cloud* which is `node:8.12.0-alpine`. So we will update the **nodejs-app/Jenkinsfile.template** Pipeline script with an *inline* Kubernetes Pod Template definition.

1. Open the GitHub editor for the **nodejs-app/Jenkinsfile.template** Pipeline script in the **master** branch of your forked **customer-marker-pipelines** repository.
2. Replace the `agent` section of the **Test** `stage` with the following:

```
      agent {
        kubernetes {
          label 'nodejs-app-inline'
          yaml """
kind: Pod
metadata:
  name: nodejs-app
spec:
  containers:
  - name: nodejs
    image: node:10.10.1-alpine
    command:
    - cat
    tty: true
  - name: testcafe
    image: 946759952272.dkr.ecr.us-east-1.amazonaws.com/kypseli/testcafe:alpha-1
    command:
    - cat
    tty: true
          """
        }
      }
```

3. The [Kubernetes plugin allows you to use standard Kubernetes Pod yaml configuration](https://github.com/jenkinsci/kubernetes-plugin#using-yaml-to-define-pod-templates) to define Pod Templates directly in your Pipeline script. Commit the changes and then navigate to the **master** branch of your **helloworld-nodejs** job in Blue Ocean on your Team Master and run the job. The job will queue indefinitely, but why? 
4. The answer is provided by the [CloudBees Kube Agent Management plugin](https://go.cloudbees.com/docs/cloudbees-core/cloud-admin-guide/agents/#monitoring-kubernetes-agents). Exit to the classic UI on your Team Master and navigate up to the **helloworld-nodejs** Multibranch folder. On the bottom left of of the sreen there is a dedicated widget that provides information about the ongoing provisioning of Kubernetes agents. It also highlights failures, allowing you to determine the root cause of a provisioning failure. Click on the link for the failed or pending pod template. <p><img src="img/parallel/pipeline_pod_template_failure.png" width=400/>
4. You will see that the **nodejs** container has an error - it looks like there is not a **node** Docker image available with that tag. If you goto [Docker Hub and look at the tags availalbe for the **node** image](https://hub.docker.com/r/library/node/tags/) you will see there is a **10.10.0-alpine** but not **10.10.1-alpine**: <p><img src="img/parallel/pipeline_pod_template_containers_error.png" width=850/> 
5. Abort the current run and open the GitHub editor for the **nodejs-app/Jenkinsfile.template** Pipeline script in the **master** branch of your forked **customer-marker-pipelines** repository. Update the `image` for the **nodejs** `container` to be `node:10.10.0-alpine`.
6. Commit the changes and then navigate to the **master** branch of your **helloworld-nodejs** job in Blue Ocean on your Team Master and run the job. The job will run successfully. Also, note the output of the `sh 'node --version'` step - it is `v10.10.0` instead of `v8.12.0`: <p><img src="img/parallel/pipeline_pod_template_node_version.png" width=850/>

## Tests

So far, we have a **Test** `stage` that doesn't really do anything. We are going to change that by executing a [Testcafe](http://devexpress.github.io/testcafe/) driven test for the **helloworld-nodejs** app in our Pipeline.

1. Open the GitHub editor for the **nodejs-app/Jenkinsfile.template** Pipeline script in the **master** branch of your forked **customer-marker-pipelines** repository.
2. Update the `steps` section of the **Test** `stage` to match the following:

```groovy
      steps {
        checkout scm
        container('nodejs') {
          sh '''
            npm install express
            npm install pug --save
            node ./hello.js &
          '''
        }
        container('testcafe') {
          sh '/opt/testcafe/docker/testcafe-docker.sh "chromium --no-sandbox,firefox" tests/*.js -r xunit:res.xml'
        }
      }
```
3. Notice how we now have 2 `container` blocks - with both containers being provided by our inline Pod Template. Also notice the `xunit:res.xml` part of the **testcafe** `sh` step. **Testcafe**  provides JUnit compatible output and it is useful to have Jenkins record that output for reporting and visualization. We will use the [`junit` step from the the JUnit plugin to captured and display test results](https://jenkins.io/doc/book/pipeline/jenkinsfile/#test) in Jenkins. We will add the `always` condition block to our `post` section of the `test` `stage` - because we want to capture both successful tests and failures:

```groovy
      post {
        success {
          stash name: 'app', includes: '*.js, public/**, views/*, Dockerfile'
        }
        always {
          junit 'res.xml'
        }
      }
```

4. Commit those changes and run the **helloworld-nodejs** **master** branch job and it will fail. It failed because the **Testcafe** test did not pass. We can see the exact error under the ["Tests" tab of the Blue Ocean Pipeline Run Details view](https://jenkins.io/doc/book/blueocean/pipeline-run-details/#tests) for this run: <p><img src="img/parallel/test_failure.png" width=850/>
5. So it appears that we have a slight typo in our **helloworld-nodejs** app. Use the GitHub editor to open the `hello.js` file on the **master** branch of your forked copy of the **helloworld-nodejs** repository, fix the misspelling of **Worlld** to **World** and then commit the changes. 
6. Navigate to the **master** branch of your **helloworld-nodejs** job in Blue Ocean on your Team Master and your job should already be running as a GitHub webhook triggered it when you commited the changes for the `hello.js` file. The test will be successful and the job will complete successfully: <p><img src="img/parallel/test_success.png" width=850/>

## Parallel Stages

The example in the section above runs tests across two different browsers - Chromium and Firefox - linearlly. In practice, if the tests took 30 minutes to complete, the "Test" stage would take 60 minutes to complete! Of course these tests are rather simple

Fortunately, Pipeline has built-in functionality for executing portions of Scripted Pipeline in parallel, implemented in the aptly named parallel step.

Refactoring the example above to use the parallel step:

```groovy
    stage('Test') {
      parallel {
        stage('Chrome') {
          agent {
            kubernetes {
              label 'nodejs-app-inline'
              yaml """
    kind: Pod
    metadata:
      name: nodejs-app
    spec:
      containers:
      - name: nodejs
        image: node:10.9.0-alpine
        command:
        - cat
        tty: true
      - name: testcafe
        image: 946759952272.dkr.ecr.us-east-1.amazonaws.com/kypseli/testcafe:alpha-1
        command:
        - cat
        tty: true
              """
            }
          }          
          steps {
            checkout scm
            container('nodejs') {
              sh '''
                npm install express
                npm install pug --save
                node ./hello.js &
              '''
            }
            container('testcafe') {
              sh '/opt/testcafe/docker/testcafe-docker.sh "chromium --no-sandbox" tests/*.js -r xunit:res.xml'
            }
          }
          post {
            success {
              stash name: 'app', includes: '*.js, public/**, views/*, Dockerfile'
            }
            always {
              junit 'res.xml'
            }
          } 
        }
        stage('Firefox') {
          agent {
            kubernetes {
              label 'nodejs-app-inline'
              yaml """
    kind: Pod
    metadata:
      name: nodejs-app
    spec:
      containers:
      - name: nodejs
        image: node:10.9.0-alpine
        command:
        - cat
        tty: true
      - name: testcafe
        image: 946759952272.dkr.ecr.us-east-1.amazonaws.com/kypseli/testcafe:alpha-1
        command:
        - cat
        tty: true
              """
            }
          }
          steps {
            checkout scm
            container('nodejs') {
              sh '''
                npm install express
                npm install pug --save
                node ./hello.js &
              '''
            }
            container('testcafe') {
              sh '/opt/testcafe/docker/testcafe-docker.sh firefox tests/*.js -r xunit:res.xml'
            }
          }
          post {
            always {
              junit 'res.xml'
            }
          }          
        }
      }
    }
```

## Sequential Stages

Running in parallel does not make a lot sense for our **helloworld-nodejs** app. With as fast as these browser tests our it doesn't really make sense to set-up the node.js app twice. But nested sequential stages may nice

1. Update agent to have two **testcafe** containers:

```
      agent {
        kubernetes {
          label 'nodejs-app-inline'
          yaml """
kind: Pod
metadata:
  name: nodejs-app
spec:
  containers:
  - name: nodejs
    image: node:10.9.0-alpine
    command:
    - cat
    tty: true
  - name: testcafe-chrome
    image: 946759952272.dkr.ecr.us-east-1.amazonaws.com/kypseli/testcafe:alpha-1
    command:
    - cat
    tty: true
  - name: testcafe-firefox
    image: 946759952272.dkr.ecr.us-east-1.amazonaws.com/kypseli/testcafe:alpha-1
    command:
    - cat
    tty: true
          """
        }
      }
```

2. Use sequential stages to run the node.js setup just once and then the browser tests in parallel:

```groovy
    stage('Test') {
      agent {
        kubernetes {
          label 'nodejs-app-inline'
          yaml """
kind: Pod
metadata:
  name: nodejs-app
spec:
  containers:
  - name: nodejs
    image: node:10.9.0-alpine
    command:
    - cat
    tty: true
  - name: testcafe-chrome
    image: 946759952272.dkr.ecr.us-east-1.amazonaws.com/kypseli/testcafe:alpha-1
    command:
    - cat
    tty: true
  - name: testcafe-firefox
    image: 946759952272.dkr.ecr.us-east-1.amazonaws.com/kypseli/testcafe:alpha-1
    command:
    - cat
    tty: true
          """
        }
      }
      stages {
        stage('Node Setup') {
          steps {
            checkout scm
            container('nodejs') {
              sh '''
                npm install express
                npm install pug --save
                node ./hello.js &
              '''
            }
          }
        }
        stage('Chrome') {
          steps {
            container('testcafe-chrome') {
              sh '/opt/testcafe/docker/testcafe-docker.sh "chromium --no-sandbox" tests/*.js -r xunit:res.xml'
            }
          }
        }
        stage('Firefox') {
          steps {
            container('testcafe-firefox') {
              sh '/opt/testcafe/docker/testcafe-docker.sh firefox tests/*.js -r xunit:res.xml'
            }
          }
        }
      }
      post {
        success {
          stash name: 'app', includes: '*.js, public/**, views/*, Dockerfile'
        }
        always {
          junit 'res.xml'
        }
      }    
    }
```
3. Navigate to the **master** branch of your **helloworld-nodejs** job in Blue Ocean on your Team Master and run the job. <p><img src="img/parallel/sequential_nested_success.png" width=850/>

## Parallel Stages with Scripted Syntax

What we really want to do above is

```groovy
    stage('Test') {
      agent {
        kubernetes {
          label 'nodejs-app-inline'
          yaml """
kind: Pod
metadata:
  name: nodejs-app
spec:
  containers:
  - name: nodejs
    image: node:10.9.0-alpine
    command:
    - cat
    tty: true
  - name: testcafe-chrome
    image: 946759952272.dkr.ecr.us-east-1.amazonaws.com/kypseli/testcafe:alpha-1
    command:
    - cat
    tty: true
  - name: testcafe-firefox
    image: 946759952272.dkr.ecr.us-east-1.amazonaws.com/kypseli/testcafe:alpha-1
    command:
    - cat
    tty: true
    ports:
    - name: firefox1
      containerPort: 1339
    - name: firefox2
      containerPort: 1340
          """
        }
      }
      steps {
        checkout scm
        container('nodejs') {
          sh '''
            npm install express
            npm install pug --save
            node ./hello.js &
          '''
        }
        script {
          parallel chrome: {
            container('testcafe-chrome') {
              sh '/opt/testcafe/docker/testcafe-docker.sh "chromium --no-sandbox" tests/*.js -r xunit:res-chrome.xml'
            }
          }, 
          firefox: {
              container('testcafe-firefox') {
                sh '/opt/testcafe/docker/testcafe-docker.sh firefox --ports 1339,1340 tests/*.js -r xunit:res-firefox.xml'
              }
          }
        }
      }
      post {
        success {
          stash name: 'app', includes: '*.js, public/**, views/*, Dockerfile'
        }
        always {
          junit 'res*.xml'
        }
      }    
    }
```

2. Despite a bit of wackiness in Blue Ocean, the final output once the job completes actually looks nice:  <p><img src="img/parallel/parallel_scipted_success.png" width=850/>
3. Another issue is that we lose the build logs in Blue Ocean for the **nodejs** steps. 

```groovy
    stage('Test') {
      agent {
        kubernetes {
          label 'nodejs-app-inline'
          yaml """
kind: Pod
metadata:
  name: nodejs-app
spec:
  containers:
  - name: nodejs
    image: node:10.9.0-alpine
    command:
    - cat
    tty: true
  - name: testcafe-chrome
    image: 946759952272.dkr.ecr.us-east-1.amazonaws.com/kypseli/testcafe:alpha-1
    command:
    - cat
    tty: true
  - name: testcafe-firefox
    image: 946759952272.dkr.ecr.us-east-1.amazonaws.com/kypseli/testcafe:alpha-1
    command:
    - cat
    tty: true
    ports:
    - name: firefox1
      containerPort: 1339
    - name: firefox2
      containerPort: 1340
          """
        }
      }
      stages {
        stage('App Setup') {
          steps {
            checkout scm
            container('nodejs') {
              sh '''
                npm install express
                npm install pug --save
                node ./hello.js &
              '''
            }
          }
        }
        stage('Browser Tests') {
          steps {
            script {
              parallel chrome: {
                container('testcafe-chrome') {
                  sh '/opt/testcafe/docker/testcafe-docker.sh "chromium --no-sandbox" tests/*.js -r xunit:res-chrome.xml'
                }
              }, 
              firefox: {
                  container('testcafe-firefox') {
                    sh '/opt/testcafe/docker/testcafe-docker.sh firefox --ports 1339,1340 tests/*.js -r xunit:res-firefox.xml'
                  }
              }
            }
          }
        }
      }
      post {
        success {
          stash name: 'app', includes: '*.js, public/**, views/*, Dockerfile'
        }
        always {
          junit 'res*.xml'
        }
      }    
    }
```
2. Now we have logs for the **App Setup** nested `stage` and our job runs a bit faster than before:  <p><img src="img/parallel/sequential_with_scripted_parallel.png" width=850/>