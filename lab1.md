# AWS Container Immersion Day: Lab 1

![containers_logo](/assets/containers_logo.png)

## Overview of lab
This lab introduces the basics of working with [docker](https://www.docker.com/) and [Amazon ECR](https://aws.amazon.com/ecr/).

This includes:
* Creating a cloud9 Environment
* Creating 2 ECR repositories
* Preparing two microservices container images
* Pushing the images to ECR

__Verify that the region is us-west-2 (OREGON)__
![lab_region](/assets/lab_region.png)

## Login to AWS Workshop Portal (at an AWS Event)
Connect to the portal by using CTRL+click or CMD+click the button or browsing to https://dashboard.eventengine.run/ in another tab.

You will need the Participant Hash provided upon entry, and your email address to track your unique session.

Once you have completed the step above, you can head straight to Create a Workspace.

## Create a Cloud9 Workspace
In the AWS console go to the `Cloud9` service

![cloud9_service](/assets/services_cloud9.png)

Click `Create environment`
![cloud9_welcome](/assets/cloud9_welcome.png)
Use `ecs-lab1` as the environment name and Click Next step until you can click `Create Environment`.


From the Cloud9 workspace, expand the terminal by clicking window icon.

![cloud9_cmd](/assets/cloud9_1.png)

## Prepping the Docker images

At this point, we're going to pretend that we're the developers of both the web and api microservices, and we will get the latest from our source repo. In this case we will just be using the plain old curl, but just pretend you're using git:

```bash
cd ~/environment
curl -O https://s3-us-west-2.amazonaws.com/apn-bootcamps/microservice-ecs-2017/ecs-lab-code-20170524.tar.gz

tar -xvf ecs-lab-code-20170524.tar.gz
```
You may see a number of errors in your terminal that look like this:
```
aws-microservices-ecs-bootcamp-v2/.git/objects/._pack
tar: Ignoring unknown extended header keyword `SCHILY.dev'
tar: Ignoring unknown extended header keyword `SCHILY.ino'
tar: Ignoring unknown extended header keyword `SCHILY.nlink'
```
These are entirely innocuous and can be ignored.

Our first step is to build and test our containers locally. If you've never worked with Docker before, there are a few basic commands that we'll use in this workshop, but you can find a more thorough list in the [Docker "Getting Started"](https://docs.docker.com/get-started/) documentation.
To build your first container, go to the web directory. This folder contains our web Python Flask microservice:

```sh
cd aws-microservices-ecs-bootcamp-v2/web
```

Let's have a look at the `Dockerfile`:
```sh
cat  Dockerfile
```
```Dockerfile
#   Copyright 2017 Amazon.com, Inc. or its affiliates.
#
#   Licensed under the Apache License, Version 2.0 (the "License");
#   you may not use this file except in compliance with the License.
#   You may obtain a copy of the License at
#
#       http://www.apache.org/licenses/LICENSE-2.0
#
#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.

FROM ubuntu:latest
MAINTAINER widha@amazon.com
RUN apt-get update -y && apt-get install -y apt-transport-https curl ca-certificates wget unzip python-pip python-dev build-essential && apt-get clean && apt-get autoremove && rm -rf /var/lib/apt/lists/*
RUN wget https://s3.dualstack.us-east-2.amazonaws.com/aws-xray-assets.us-east-2/xray-daemon/aws-xray-daemon-linux-2.x.zip && unzip aws-xray-daemon-linux-2.x.zip && cp ./xray /usr/bin/xray
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
ENTRYPOINT ["python"]

EXPOSE 3000

CMD ["xray", "--log-file /var/log/xray-daemon.log"] &
CMD ["app.py"]
```

To build the container you need
```sh
docker build -t ecs-lab-web .
```

This should output steps that look something like this:
```
Sending build context to Docker daemon 4.096 kB
Sending build context to Docker daemon 
Step 0 : FROM ubuntu:latest
 ---> 6aa0b6d7eb90
Step 1 : MAINTAINER widha@amazon.com
 ---> Using cache
 ---> 3f2b91d4e7a9
```

To view the image that was just built:
```sh
docker images | grep ecs-lab
```
To run your container:
```sh
docker run -d -p 3000:3000 ecs-lab-web
```
This command runs the image in daemon mode and maps the docker container port 3000 with the host (in this case our workstation) port 3000. We're doing this so that we can run both microservices on a single host without port conflicts.

To check if your container is running:
```
docker ps
```
This should return a list of all the currently running containers. In this example, it should just return a single container, the one that we just started:
```
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                    NAMES
7b0d04f4502c        ecs-lab-web         "python app.py"     9 seconds ago       Up 9 seconds        0.0.0.0:3000->3000/tcp   eloquent_noether
```

To test the actual container output:
```sh
curl -s localhost:3000/web
```
This should return:
```html
<html>
  <head>
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css" integrity="sha384-BVYiiSIFeK1dGmJRAkycuHAHRg32OmUcww7on3RYdg4Va+PmSTsz/K68vbdEjh4u" crossorigin="anonymous">
    <script src="https://cdnjs.cloudflare.com/ajax/libs/fetch/2.0.3/fetch.min.js" type="text/javascript"></script>
    <style>h1 {text-align: center;} p {text-align:center;}#status {margin-top:30px;text-align:center} #api-response {visibility: hidden;text-align:center;font-family:Menlo,Monaco,Consolas,"Courier New",monospace;border: 1px solid #aaa; border-radius:4px;-webkit-border-radius:4px;-moz-border-radius:4px;border-radius:4px; padding:10px; font-size:10px;}</style>
  </head>
  <body>
  <h1>Hi! I'm Web</h1>
  <p class="lead">I'm served via Python + Flask.</p>
  <p><a class="btn btn-lg btn-success" href="#" role="button" id="btn-call">Let's call API</a></p>
  <div id="status"><span id="api-response"></span></div>
  <script>
    function checkStatus(response) {
      if (response.status == 200 && response.status < 300) {
        return response;
      } else {
        var error = new Error(response.statusText);
        error.response = response;
        throw error;
      }
    }
    function call() {
      var status = document.getElementById('api-response');
      
      status.style.visibility = 'visible';
      status.innerHTML = 'calling API ... ';

      fetch('/api')
      .then(checkStatus)
      .then(function(response) { return response.json()})
      .then(function(data) {
        status.innerHTML = 'successful calling /API. \<br />\<br />Response: \<pre>' + JSON.stringify(data) + '\</pre>';
      })
      .catch(function(error) {
        status.innerHTML = 'error calling /API ' + error;
      })
    }
    document.getElementById('btn-call').onclick = call;
  </script>
  </body>
</html>
```

Repeat the same steps with the api microservice. Change directory to /api and repeat the same steps above
```sh
cd ../api 
docker build -t ecs-lab-api .
docker run -d -p 8000:8000 ecs-lab-api
curl localhost:8000/api
```
The API container should return:
```json
{ "response" : "hi!  i'm ALSO served via Python + Flask.  i'm an API." }
```
__We now have two working microservices containers.__

## Creating container registries with ECR

Once images are built, it’s useful to share them and this is done by pushing the images to a container registry. 

Let’s create two repositories in Amazon EC2 Container Registry (ECR):

Navigate to the ECR console, and select `Get Started`.

![ecr_service](/assets/service_ecr.png)

Name your first repository `ecs-lab-web`

![ecr_web](/assets/ecr_1.png)


Once you've created the repository click it, then `click View Push Commands`.

__Take note of these, as you'll need them in the next step.__

The push commands should like something like this:

![ecr_web](/assets/ecr_2.png)

__Once you've created the `ecs-lab-web` repository, repeat the process for the `ecs-lab-api` repository.__

## Pushing our tested images to ECR
Now that we've tested our images locally, we need to tag and push them to ECR. This will allow us to use them in Task Definitions that can be deployed to an ECS cluster.

You'll need your push commands that you saw during registry creation. You can find them again by going back to the repository (ECS Console > Repositories > Select the Repository you want to see the commands for > View Push Commands).

_Note: why `:latest`? This is the actual image tag. In most production environments, you'd tag images for different schemes, for example, you might tag the most up-to-date image with :latest, and all other versions of the same container with a commit SHA from a CI job. If you push an image without a specific tag, it will default to :latest, and untag the previous image with that tag. For more information on Docker tags, see the Docker documentation. 

You can see your pushed images by viewing the repository in the ECS Console._ Alternatively, you can use the CLI:
```sh
aws ecr list-images --repository-name=ecs-lab-web
```
```json
{
    "imageIds": [
        {
            "imageTag": "latest", 
            "imageDigest": "sha256:5d366779e6bd5cbe43cf776663da4138f086bffa6dea3acd5f63b3c77e7e7b39"
        }
    ]
}
```

```sh
aws ecr list-images --repository-name=ecs-lab-api
```
```json
{
    "imageIds": [
        {
            "imageTag": "latest", 
            "imageDigest": "sha256:433f6b2cd0690bd0f4d250cccf2c24554eacd3912f7b8a08afb4ee91737c4fa0"
        }
    ]
}
```

You have successfully completed Lab 1
