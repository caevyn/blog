---
title: batch jobs with aws lambda and docker
description: How to use lambda and docker on ecs to run batch jobs. We will look at how to generate a static blog in a container everytime a github commit is made, uploading it to S3 when done.
date: "May 17th, 2015"
created: 2015-05-17
---
I've been working a lot with scheduling batch jobs at work, which isn't the most exciting thing in the world; however, I've been spending a lot of my own time playing with AWS. It turns out AWS is a convenient place to run batch jobs. There are plenty of batch scenarios where a file turns up, triggers a program to do some work, which in turn might produce some output. A popular example of this sort of processing is a static blog generator, so we will use that as an example in this post. 

Amazon's new elastic container service allows you to deploy and run Docker containers across a cluster of EC2 instances. I'm sure the most popular use of ECS will be as a convenient place to deploy web apps and services; however, the task abstraction at its core maps very well to running batch jobs.

Another new AWS service is Lambda, which allows you to run nodejs functions in reaction to certain events. It seems like a very useful service for processing many kinds of events, but there are some constraints. You are limited to 60 seconds of processing time, you don't have full control of the underlying instance and it only supports nodejs (although you can launch non-node processes from the node code if the dependencies are on the AMI). 
Lambda on its own would probably be perfect for generating a static site using a nodejs static site generator, but I've become quite fond of Elixir and want to use [Benny Hallet's Obelisk](https://github.com/BennyHallett/obelisk) for a blog generator. To do this I'll need the Erlang VM, Elixir, an Obelisk blog to generate, and somewhere to put the output. I also want to trigger a rebuild every time I push to Github.

The following diagram gives a better idea how these piece can fit together.
![Blog generation diagram](http://img.maltmurphy.com/hipsterbatchgliffy.png)

Github already has a convenient SNS integration which will publish a notification on push. AWS Lambda can subscribe to an SNS topic to trigger the nodejs function, which can in turn use the nodejs AWS SDK to kick off our ECS task. The ECS task consists of a Docker container which just runs a build script. It grabs the latest blog code, runs Obelisk to generate the blog, and publishes it to an S3 bucket serving a website via Route 53.

There are certainly simpler ways to generate and host a blog, so think of this as a generic batch pipeline where the jobs might require different run-times or resources. Docker gives you isolation and ECS will take care of scheduling your task so that it gets allocated the required resources in your cluster. Since ECS is just EC2 under the hood, you can take advantage of auto scaling groups, VPC security, etc. The combination of Lambda and ECS could be applied to many other batch scenarios.

### SNS Topic and Github Integration

Create a new SNS topic and add an email subscription so we can test that it works. You can use the UI or AWS cli.
    
    $ aws sns create-topic --name BlogDeploy
    $ aws sns subscribe --topic-arn <your topic arn> --protocol email --notification-endpoint myemail@gmail.com

It is also worth creating an IAM user just for this with permission to publish to the SNS topic but nothing else. Copy down the Id and secret key as you will need them to paste into Github. Edit the topic policy to give the new user access by adding the following statement to the array of statements.
    
    {
      "Sid": "__console_pub_0",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<account>:user/<username>"
      },
      "Action": "SNS:Publish",
      "Resource": <your topic arn>
    }

In your Github repo, go to Settings -> WebHooks and Services and add an AmazonSNS service. You just need to provide the arn and region of the topic and the Id and secret key of the IAM user created earlier. Pushing a change to the repo should now publish an SNS message and subsequent email.

### S3 bucket and Route 53

Set up an S3 bucket to host the site. I won't go into detail here, AWS has good docs around it [here](http://docs.aws.amazon.com/AmazonS3/latest/dev/WebsiteHosting.html). It is important to name the bucket the same as your domain. E.g. maltmurphy.com. Again, I suggest creating an IAM user just for publishing to the bucket. Add a bucket policy that allows that user to write to that bucket. Setting up route 53 for my domain was also straight forward thanks to the [good docs](http://docs.aws.amazon.com/AmazonS3/latest/dev/website-hosting-custom-domain-walkthrough.html).

The key statements on the bucket policy are:

       {
		"Sid": "Stmt1430825277885",
		"Effect": "Allow",
		"Principal": {
			"AWS": "arn:aws:iam::<acc number>:user/s3guy"
		},
		"Action": "s3:ListBucket",
		"Resource": "arn:aws:s3:::maltmurphy.com"
	},
	{
		"Sid": "Stmt1430825277884",
		"Effect": "Allow",
		"Principal": {
			"AWS": "arn:aws:iam::<acc number>:user/s3guy"
		},
		"Action": "s3:*",
		"Resource": "arn:aws:s3:::maltmurphy.com/*"
	}

### Docker container

The docker container I use for the blog can be found [here](https://github.com/caevyn/obelisk-builder). It inherits from an existing Elixir base image, adds the aws-cli and a shell script. When you run the container the following script runs:

    #!/bin/bash
    git clone https://github.com/caevyn/blog.git
    cd ./blog
    mix do deps.get, deps.compile
    mix obelisk build
    aws s3 sync build s3://maltmurphy.com/ --delete
 
It simply clones the repo, uses Mix (The awesome Elixir command line tool) to fetch dependencies and compile. Obelisk is executed as a mix task and 'mix obelisk build' runs the static site generation. The aws cli is used to copy to an S3 bucket. The aws cli requires some environment variables for aws credentials to be passed in. This can be done via the ECS task but we will get to that in a moment.

Build the Docker container and give it a docker hub friendly name, i.e. username/obelisk-builder. We will publish to the docker hub to make configuring ECS easier. Test the container and make sure it can connect to your S3 bucket, using the credentials of your S3 IAM user.
            
      docker run -i -t -e "AWS_SECRET_ACCESS_KEY=key" -e "AWS_ACCESS_KEY_ID=id" username/obelisk-builder
 
 Once it works, push the image to the docker hub:
     
     docker push username/obelisk-builder

### ECS

Now we have a Docker image published to the hub we can set up an ECS task. When you first visit ECS you will step through a wizard where you can create a task on a default cluster. In step 1, choose 'custom task definition'. In the task definition, put in the name of the image you published to the docker hub, allocate some resources and set the AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY environment variables for the S3 user's credentials. In step 3, choose 'run tasks once' and set the desired number of tasks to 1. Continue on and configure the cluster so that there is at least one instance. You will also be asked to create an IAM role that allows access to the ECS service. The ECS task definition can be found in this [Gist](https://gist.github.com/caevyn/3c95cdd6b9cd0c197cba). 

Run the task and hopefully everything works. If it doesn't, there are a couple of things that can help troubleshoot. SSH into your container instance and you can query the task info as it is running with the following:
    
     curl http://localhost:51678/v1/tasks | python -mjson.tool

The 'docker logs' command is also very handy to see what exactly went on during the task run.

### Lambda

All that is left to configure is our Lambda job. It will subscribe to the SNS topic and kick off the ECS task using the node aws-sdk.

    console.log('Loading function');
    var aws = require('aws-sdk');
     
    exports.handler = function(event, context) {
        //console.log('From SNS:', event.Records[0].Sns.Message);
        var ecs = new aws.ECS();
        var params = {
          taskDefinition: 'arn:aws:ecs:us-west-2:<acc number>:task-definition/build-blog:3',
          count: 1,
          startedBy: 'lambda'
        };
        ecs.runTask(params, function(err, data) {
          if (err) console.log(err, err.stack);
          else    { console.log(data); context.succeed("yew!");}
        });
    }; 
     
     
Create an IAM role with the following policy to use so that the Lambda function has permission to run your task and nothing else:

    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "logs:*"
          ],
          "Resource": "arn:aws:logs:*:*:*"
        },
        {
          "Effect": "Allow",
          "Action": [
            "ecs:RunTask"
          ],
          "Resource": [
            "arn:aws:ecs:us-west-2:<acc number>:task-definition/build-blog:*"
          ]
        }
      ]
    }

You can test your Lambda function in the UI, when you run it your ECS task should start.
The final piece is to add a subscription to the SNS topic for the Lambda job. Go back to SNS and add in the subscription through the UI, alternatively use the cli like we did for the email subscription.

If you push to your Git repo, you should now see an update to your blog soon after.

### Final Thoughts

As I mentioned earlier, there are easier ways to generate a blog. However, the combination of Lambda and ECS tasks is a nice way to run batch jobs in reaction to events, especially if you have a variety of run-times which can be easily wrapped up via Docker. Uploading a file to S3, processing some data with your favourite language and producing some reports or other output describes a lot of batch workloads and this could be a nice way to manage them. The thing that continues to impress me about AWS is how the new services compliment the existing ones and everything works well together.
