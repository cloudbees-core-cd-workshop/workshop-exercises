# Parallel and Sequential Stages with CloudBees Core

## Kubernetes Pod Templates Defined in Pipeline Script

So far we have been using the **nodejs-app** Kubernetes Pod Template defined for us on **CloudBees Jenkins Operations Center (CJCO)**. However, for the next stage we will need an additional container for executing tests and we also want to use a different vesion of the **node** Docker image than the one provided by the CJOC Kubernetes Shared Cloud: `node:8.12.0-alpine`. So we will update the **nodejs-app/Jenkinsfile.template** Pipeline script with an *inline* Kubernetes Pod Template definition.

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
    image: node:10.9.0-alpine
    command:
    - cat
    tty: true
  - name: testcafe
    image: beedemo/testcafe@sha256:7cae1a73327d2ef2db61a4fe523bd8ee1697c104e928c1de05f207e0220c890c
    command:
    - cat
    tty: true
          """
        }
      }
```

3. Commit the changes and then navigate to the **master** branch of your **helloworld-nodejs** job in Blue Ocean on your Team Master and run the job. Note the output of the `sh 'node --version'` step - it is `v10.9.0` instead of `v8.12.0`: <p><img src="img/parallel/pipeline_pod_template.png" width=850/>

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
          sh '/opt/testcafe/docker/testcafe-docker.sh "chromium --no-sandbox" tests/*.js -r xunit:res.xml'
        }
      }
```
3. Notice how we now have 2 `container` blocks - with both containers being provided by our inline Pod Template. Also notice the `xunit:res.xml` part of the **testcafe** `sh` step. **Testcafe**  provides JUnit compatible output and it is useful to have Jenkins record that output for reporting and visualization. We will use the `junit` step from the the JUnit plugin to captured and display test results in Jenkins. We will add the `always` condition block to our `post` section of the `test` `stage` - because we want to capture both successful tests and failures:

```groovy
      post {
        success {
          stash name: 'app', includes: '*.js, public/**, views/*, Dockerfile'
        }
        always {[^1]
          junit 'res.xml'
        }
      }
```

4. Commit those changes and run the **helloworld-nodejs** **master** branch job and it will fail. It failed because the **Testcafe** test did not pass. We can see the exact error under the ["Tests" tab of the Blue Ocean Pipeline Run Details view](https://jenkins.io/doc/book/blueocean/pipeline-run-details/#tests) for this run: <p><img src="img/parallel/test_failure.png" width=850/>
5. So it appears that we have a slight typo in our **helloworld-nodejs** app. Use the GitHub editor to open the `hello.js` file on the **master** branch of your forked copy of the **helloworld-nodejs** repository, fix the misspelling of **Worlld** to **World** and then commmit the changes. 
6. Navigate to the **master** branch of your **helloworld-nodejs** job in Blue Ocean on your Team Master and your job should already be running as a GitHub webhook triggered it when you commited the changes for the `hello.js` file. The test will be successful and the job will complete successfully: <p><img src="img/parallel/test_success.png" width=850/>

## Parallel Stages



## Sequential Stages