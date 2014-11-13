# Part 4: A CI Pipeline for Automated Deployments

### **Goal: build a CI pipeline to deploy a dummy application in an automated, reliable, and repeatable manner.**

We'll be buidling upon the last workshop to create a CI pipeline that tests, packages, publishes, and deploys a dummy application every time you commit to the application's repository. To this end, we'll be touching on some new concepts and tools:

- IAM Roles
- S3
- Cloudinit
- The Phoenix Server philosophy
- Go CD pipelines

I will discuss each of these when they become relevant.

#### Disclaimer
In the interest of building an end to end deployment pipeline in a single workshop, we're going to have to take some pretty serious shortcuts. What you will have built by the end of this workshop will _never_ suffice in the wild. However, it will be enough for you to grasp the essence of CI pipelining and automated deployments.

#### Tear down your infrastructure when you're done
We'll be provisioning three medium EC2 instances which cost around 9 cents an hour each. So don't forget to tear down your infrastructure when you're done.


## Set yourself up
I'll assume that you've done the previous workshops and have Ansible and the AWS cli set up on your machine.

You'll want a good overview of what you're doing throughout this workshop, so I would recommend opening the following AWS web concole services in seperate browser tabs so that you can move back and fourth easily:

- Cloudformation
- EC2
- S3
- IAM

In the past few workshops, you've most likely been running configuration and provisioning from scripts in a local clone of this repository, for this workshop we need to change things up a little. If you haven't already done so, fork this repository into your own github account, then clone it locally. You'll be working from your cloned/forked directory here on out, and can remove the old one if you like.

We're doing this because you're going to have to be able to push code up to github, and you don't have the permissions to push to my repository. Have a look at your remotes if you want to make sure that you're properly set up (`git remote -v` from within the repository you just cloned). You should see something like this:
    
    origin	git@github.com:YOUR_GITHUB_NAME/Devops-101.git (fetch)
    origin	git@github.com:YOUR_GITHUB_NAME/Devops-101.git (push)



## Build your infrastructure
We'll be going down a similar route as the last workshop. We'll use Cloudformation to create a stack of resources, and then Ansible to configure a Go server and Go agent on two seperate EC2 instances. The following commands will require some time and patience, so execute them and read on while they complete.

Let's get to it:

- go to the `part-four/infrastructure/provisioning/` directory
- provision the infrastructure:

        aws cloudformation create-stack --stack-name infrastructure --template-body "file://./infrastructure-template.json" --capabilities CAPABILITY_IAM
  
- go to your Cloudformation browser tab, you should see your infrastructure stack being created, or equivalently through the AWS cli:

        aws cloudformation describe-stacks
  
While Cloudformation creates your infrastructure, let's take a look at the stack template in `infrastructure-template.json`. The template is very similar to what we had in the previous workshop, but I've added a few resources:

|Resource|Description|
|:--|:--|
|Bucket| This is used to create an S3 bucket which we'll be using to store the packaged application|
|InstanceProfile|This resource gets added to an EC2 Instance resource and lets us associate an IAM role to an EC2 instance|
|Role|An EC2 instance assumes an IAM role, the role has policies associated to it, which determine what permissions the EC2 instance has when assuming that role|
|Policy|A description of the permissions that we wish to give to a role, we then associate that role to an EC2 instance, such that the instance has permission do certain AWS related tasks|

We'll get to S3 and buckets a little later, what's most important here is the role resource, and it's supporting resources.

#### Understanding IAM roles
Remember when we installed the AWS cli? Remember how we had to create that AWS config file with some AWS credentials so that we could access our AWS account from the command line and do fun things like Cloudformation? Well, those credentials - obviously - are what let us do everything we wish to do with AWS. 

If you cast your mind way back, you'll recall that we've given ourselves full administration access. If you go to the IAM service tab and look at your user, you'll see that you're a part of the `Administrators` group. If you go to that group, you'll see that it has a policy called something like `AdministratorAccess-XXXXXXXXXX`. If you click `show` you'll see something like this:

```javascript
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    }
  ]
}
```
What you're looking at is the policy that allows you to do whatever you want with AWS. Whenever you use the AWS cli, your credentials are used to pull up this policy, and then your desired AWS command is checked against what you can and cannot do.


Now, what if we want one of our EC2 instances to be able to do things like launch a Cloudformation stack or access S3? Without it's own credentials, it won't be able to do anything. Well, we could simply create a new user and put the right credentials on the instance, right?

Wrong.

It's really not a good idea to be throwing credentials around like candy. What we really want is to be able to give an EC2 instance a temporary set of credentials that are easy to distribute, rotate, and revoke. This is where IAM roles come in. You assign a role to an instance, and you assign policies to that role, much like the policy above, but with much stricter permissions of course. 

So you can think of it a bit like this: an IAM role is to a machine what an IAM user is to a human. See [here](http://docs.aws.amazon.com/IAM/latest/UserGuide/WorkingWithRoles.html) for a more in depth discussion on IAM roles and the problems it solves.

Now that you have a basic understanding of roles, look closely at the template, you'll see that by using the role, policy, and instanceProfile resources, we've given a bunch of permissions to our CI slave instance. We're doing this because we want our CI slave to be able to use the AWS cli to carry out tasks that we will discuss soon enough.


## Configure your CI environment
By now, your infrastructure stack should be built. Like we did in the last workshop, we'll need to go through an irritating manual step.

- check the outputs from your stack creation:

        aws cloudformation describe-stacks
        
- if the stack is built you'll see something like this:

    ```    
    OUTPUTS	CIMasterPublicIp	XX.XX.XX.XX
    OUTPUTS	CISlavePublicIp	    XX.XX.XX.XX
    OUTPUTS	CIMasterPrivateIp	XX.XX.XX.XX
    ```
    
- grab those values and put them in the `part-four/infrastructure/configuration/inventory` file
- go to the `part-four/infrastructure/configuration/` directory
- launch Ansible to configure your CI master and CI slave instances:
    
    ```
    ansible-playbook playbook.yml -u ubuntu -i inventory --private-key="~/.ssh/main.pem"
    ```
    
This will take a little while. In the meantime, it's worth looking over `playbook.yml` to refresh your memory on what we're doing to get CI up and running. Not much has changed since the last workshop, with the exception of a few extra packages being installed on the CI slave, like Leiningen, Ruby, and the AWS cli.

When Ansible has completed, ssh to it with port forwarding such that we can access the Go web console:

```
ssh -L 8153:localhost:8153 ubuntu@YOUR_CI_MASTER_PUBLIC_IP -i ~/.ssh/main.pem
```

Now open `http://localhost:8153/` in your browser to access the Go web console.


## Create your first pipeline
Now we're ready to get to the meat of this workshop. We're going to build this pipeline out incrementally. But first, let's think about the overall picture of what we want to achieve:

|Pipeline Stage|Description|
|:--|:--|
|Pull down the repository| Although not technically a stage, we'll be pulling application code down from a github repository|
|Test| Run the tests in the application code with Leiningen|
|Package| Package the application as a standalone jar with Leiningen|
|Publish| Publish the jar to S3 where we'll later fetch it to run it|
|Deploy| We'll be creating a new app server and deploying the application to it every time we run the pipeline|


#### Pull down the repository
There are a few steps involved in pulling the repository onto the CI slave:

- firstly, on the Go web console, go to the `PIPELINES` tab, you will be prompted to add a pipeline
- use `dummyApplication` as the pipeline name, then select `Next`
- select `Git` as the `Material Type` 
- go to **your** version of this repository on github, and on the right hand pane, you should see `SSH clone URL` or `HTTPS clone URL`, copy the **HTTPS URL**, it should look something like this: `https://github.com/YOUR_GITHUB_NAME/devops-101.git`
- now, back on the Go web console, put that URL into the relevant field
- `Check Connection` and then select `next`

#### Create the test stage
We're now ready to create the first stage. Fill in the fields as follows:

|Field| Value|
|:--|:--|
|Stage Name| test|
|Job Name| test|
|Task Type| more|
|Command| `lein`|
|Arguments| `test`|
|Working Directory| part-four/application|

Now press `Finish` and you'll see the beginnings of you pipeline. But we're not quite done. On the left you'll see a pipeline structure with `test` as a sub label under `dummyApplication`, click on the `test` label, this should bring up the `Stage Settings` panel. Select `Clean Working Directory` and press `Save`.

Let's explore how Go organises the structure of a pipeline:

|Component| Description|
|:--|:--|
|Pipeline| This is the top level, in our case, our pipeline is called `dummyApplication`, a pipeline can have one or more stages|
|Stages| Stages within a pipeline execute sequentially, a stage can have one or more jobs|
|Jobs| Jobs within a stage execute in *parallel*, so be careful with these, each job can have one or more tasks|
|Tasks| Tasks within a job execute sequentially, these are the bottom level guys|

Play around with Go and try to perceive this pipeline structure. when you're ready, lets run the pipeline for the first time.

#### Run the thing
Before you can run the pipeline, you'll need to make sure that the agent is enabled:

- go to the `AGENTS` tab, you should see the `ci-slave` agent
- select it and hit `ENABLE`

Now go back to the `PIPELINES` tab and hit the pause button to unpause the pipeline. Within a few moments it should start to run:

- click the yellow (or green if it's complete) bar
- on the left hand `JOBS` panel you should see `test`, click on it
- now go to the `Console` tab if you're not already there

Within a minute or so you should see a bunch of output showing the jobs being carried out on the Go agent. Have a read through it and see if you can discern what's going on.


So what just happened? 

1. the Go server dispatched the stage to the Go agent running on the CI slave instance
2. the Go agent pulled down the repository
3. the Go agent began the `test` job
4. the job passed, therefore the stage passed, therefore the pipeline passed
5. and the world rejoiced

As a side note, the `test` job uses Leiningen, which is a project management tool for Clojure (which is what our dummy application is written in). All you need to know about Leiningen is that we can use it to run our tests and build our application. You don't need to know much more, but if you like, you can learn about it [here](http://leiningen.org/).

#### Fail fast
Let's play around with our pipeline a little. First, let's confirm that when we commit a change and push it up to github, the pipeline is triggered automatically. Secondly, let's commit a change that makes our tests fail so that we can confirm that the pipeline fails:

- open `part-four/application/test/application/core_test.clj` in your favourite editor
- on line 7, change the 1 to a 2, this will make the test fail
- commit and push the change

In the Go web console, navigate to the `PIPELINES` tab and wait patiently for Go to pick up the changes to your repository. This can take up to a minute sometimes. You shouldn't have to trigger it manually.


When it finally gets triggered, you'll see that it fails on test. This is what we wanted to confirm. Now go and fix that failing test, commit it, push it, and watch the pipeline go green again.

#### Create the package stage
Now, let's create the second stage:

- go to the `PIPELINES` tab, hit the cog icon on the top right hand of the pipeline panel
- go to the `Stages` tab
- then `Add new stage`
- fill in the fields as follows:


    |Field| Value|
    |:--|:--|
    |Stage Name| package|
    |Job Name| package|
    |Task Type| more|
    |Command| `lein`|
    |Arguments| `uberjar`|
    |Working Directory| part-four/application|
- press `Save`
- once again, go to the `Stage Settings` under the `package` stage and select `Clean Working Directory`
- don't forget to `Save`

This task is straight forward. We're packaging the application with the Leiningen `uberjar` command. This command generates two jars in the `target/uberjar` directory relative to the application's root directory. 

#### Renaming the jars
The names of the jars produced by the above task will be something like `application-0.1.0-SNAPSHOT-standalone.jar` and `application-0.1.0-SNAPSHOT.jar`. Every time we run a Go pipeline, it increments the pipeline's counter. Wouldn't it be nice to stamp that counter onto the name of our jars so that we know which pipeline run produced them? Well, Go gives us some environment variables that we can use, including $GO_PIPELINE_COUNTER. Let's create a task that changes the jar names to something like `application-0.1.0-$GO_PIPELINE_COUNTER-standalone.jar` and `application-0.1.0-$GO_PIPELINE_COUNTER.jar`:

- go to the `PIPELINES` tab, hit the cog icon on the top right hand of the pipeline panel
- go to the `package` stage's `Stage Settings` panel and select the `Jobs` tab
- select the `package` job and then `Add new task`
- select `more` for the type of task
- fill in the fields as follows:

    |Field| Value|
    |:--|:--|
    |Command| `sh`|
    |Arguments (line 1)| `-c`|
    |Arguments (line 2)| `rename "s/-SNAPSHOT/-$GO_PIPELINE_COUNTER/" application-*.jar`|
    |Working Directory| part-four/application/target/uberjar|
    
    **Note:** be very careful to put the `-c` and the `rename` command on separate lines.

So there we go, we've got two nicely named jars. But they aren't of much use to us just sitting there on the Go agent. What we really want is to make them available to later stages. We'll achieve that by "artifacterizing" them.

#### Dealing with artifacts
Typically, you have several stages that need to be executed sequentially in a single pipeline run. If you have multiple Go agents, any agent can be assigned the next stage, you're not guaranteed to have the same agent executing every stage in a single pipeline run. This means that when a stage produces some kind of output, and we require that output as input to some later stage, we need to take that output and throw it over to the Go server, such that it can orchestrate where it will be needed next. These outputs are called artifacts, and you'll see a lot of these going around in the wild.

In our case it's rather simple:

1. the `package` stage produces two artifacts (the jars)
2. we want to send those artifacts up to the Go server for safe keeping
3. when we need them in a later stage, we can simply fetch the artifacts from the Go server

So let's do it:

- go to the `PIPELINES` tab, hit the cog icon on the top right hand of the pipeline panel
- select the `package` stage
- then select the `package` job under the `Jobs` tab
- go to the `Artifacts` tab
- in `Source` put `part-four/application/target/uberjar/application-*jar`, that wildcard will pick up both the jars
- in `Destination` put `packages` (be careful to include the trailing 's' here)
- don't forget to `Save`
- now go to the `PIPELINES` tab and run the pipeline

When the pipeline has completed successfully, the two jars that were produced on the Go agent will have been transferred up to the Go server. If you want to verify this, go to your terminal where your ssh connection to the Go server will still be open, and navigate to `/var/lib/go-server/artifacts/pipelines/dummyApplication/`. In this directory you should see the corresponding numbers of pipeline runs, if you dig down into that directory you should find the `packages` directory which houses the two jars you just produced and renamed.
 
#### Create the publish stage
You may have assumed that we would be ready to deploy the application at this point. But there is one more stage we need to consider before doing so, and that's the publish stage. It's always a good idea to keep the outputs of our pipelines somewhere safe, because you never know when you'll need them. Now, we're already sending the artifacts to the Go server after the package stage, so why do more? The short answer is that we shouldn't treat the Go server as an artifact repository, that's not what it's made for. We need something a little more suited to the purpose. 

That's were S3 comes in. S3 is AWS' general purpose file storage solution. There are much better tools for hosting our artifacts out there, but for now, S3 will do. We're simply going to be using it to store every jar that's produced by our pipeline.

You should know how to create a stage by now, here are the fields you need to fill in:

|Field| Value|
|:--|:--|
|Stage Name| publish|
|Job Name| publish|
|Task Type| more|
|Command| `sh`|
|Arguments (line 1)| `-c`|
|Arguments (line 2)| `aws s3 cp packages/ s3://devops-part-four/ --recursive --exclude "*" --include "application-*-$GO_PIPELINE_COUNTER*jar"`|

**Note:** be very careful to put the `-c` and the `aws` command on separate lines. You'll also need to select `Clean Working Directory` in the stage settings, you should know how to do that by now.

So this task uses the AWS cli's S3 tool to upload our jars to an S3 bucket called "devops-part-four" which was provisioned as part of our infrastructure stack. It should now be clear why we wanted to give our CI slave instance an IAM role that allows it to run AWS S3 commands.

There's one last thing we need to do before we go and rerun this pipeline. We need to pull the jars down from the Go server so as to upload them to S3:

- go to the `PIPELINES` tab, hit the cog icon on the top right hand of the pipeline panel
- select the `publish` stage
- then select the `publish` job under the `Jobs` tab
- select `Add new task` and select `Fetch Artifact` for the type of task
- fill in the fields as follows, but be careful, some fields don't need to be filled:

    |Field| Value|
    |:--|:--|
    |Stage| package|
    |Job| package|
    |Source| packages|
    
     **Note:** be careful to include the trailing 's' in 'packages' in the `Source` field.
- don't forget to `Save`
- you need to switch the order of the two tasks by clicking the arrow icon in the `order` column, ensure that the `Fetch Artifact` task comes first

This special task goes and grabs the artifact produced by the `package` stage's `package` job. In particular, it goes and grabs the `packages` directory within which we can find the two jars. Now go and run the pipeline. When it's complete, navigate to your S3 browser tab and take a look in the `devops-part-four` bucket. You should see the jars.

#### Create the deploy stage
We're finally in a position to deploy our application. But first, Let's think about what steps we need to take.

1. **Provision an EC2 instance:**     
   **TODO** Ensure that it has access to S3 
   
   
2. **Configure the EC2 instance:**      
   **TODO** Ensure that it has Java installed
   
3. **Get the uberjar onto the EC2 instance:**     
   **TODO** You need the jar from S3
   
4. **Run the application:**    
   **TODO** Make a user and run the jar  
   
5. **Delete the old EC2 instance:**   
   **TODO** The first time you run your pipeline, this won't be a problem, but any time after that, you'll need to blast              away the old one after you build the new one. Why build one every time? We'll discuss that in just a moment. The main point to take away here is that you need to remove the old app server.
   

The first thing that may strike you as odd is that we're redeploying an entire EC2 instance just for a single little jar. Yes, that's true, it's a pretty big undertaking. But what I'm trying to demonstrate here is the Phoenix Server pattern. In the wild, it's not uncommon to host a single application per Ec2 instance.

**TODO**
- How IAM roles are being used here to orchestrate this
- Why use the Phoenix server pattern
- How Cloudinit is being used here


### Clean up:

**TODO** How to clean up

**TODO**
- Fix up commit message
- Fix part three image






