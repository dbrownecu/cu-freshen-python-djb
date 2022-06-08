<!--
#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
-->

removed ibmcloudant>=0.1.1 to core/requirements_common.txt
added cloudant to core/requirements_common.txt
added opencv-python

This repository contains sources files needed to build the Python runtimes for Apache OpenWhisk. The build system will produce a series of docker images for each runtime version. These images are used in the platform to execute Python actions.

The following Python runtime versions (with kind & image labels) are generated by the build system:

- Python 3.7 (python:3.7 & openwhisk/action-python-v3.7)
- Python 3.9 (python:3.9 & openwhisk/action-python-v3.9)
- Python 3.6 AI (python:3.6 & openwhisk/action-python-v3.6-ai)

This README documents the build, customization and testing of these runtime images.

To learn more about using Python actions to build serverless applications, check out the main project documentation [here](https://github.com/apache/openwhisk/blob/master/docs/actions-python.md).

## Build Runtimes

There are two options to build the Python runtime:

- Building locally: [tutorial](tutorials/local_build.md)
- Using OpenWhisk Actions.

### Building Python Runtime using OpenWhisk Actions

Pre-requisites

- [Gradle](https://gradle.org/)
- [Docker](https://www.docker.com/)
- [OpenWhisk CLI wsk](https://github.com/apache/openwhisk-cli/releases)

The runtimes are built using Gradle. The file [settings.gradle](settings.gradle) lists the images that are built by default.

To build all those images, run the following command.

```
./gradlew distDocker
```

You can optionally build a specific image by modifying the gradle command. For example:
```
./gradlew core:python3Action:distDocker
```

The build will produce Docker images such as `action-python-v3.7`
and will also tag the same image with the `whisk/` prefix. The latter
is a convenience, which if you're testing with a local OpenWhisk
stack, allows you to skip pushing the image to Docker Hub.

The image will need to be pushed to Docker Hub if you want to test it
with a hosted OpenWhisk installation.

### Using Gradle to push to a Docker Registry

The Gradle build parameters `dockerImagePrefix` and `dockerRegistry`
can be configured for your Docker Registry. Make sure you are logged
in first with the `docker` CLI.

- Use the `docker` CLI to login. The following assumes you will substitute `$DOCKER_USER` with an appropriate value.
  ```
  docker login --username $DOCKER_USER
  ```

- Now build, tag and push the image accordingly.
  ```
  ./gradlew distDocker -PdockerImagePrefix=$DOCKER_USER -PdockerRegistry=docker.io
  ```

### Using Your Image as an OpenWhisk Action

You can now use this image as an OpenWhisk action. For example, to use
the image `action-python-v3.7` as an action runtime, you would run
the following command.

```
wsk action update myAction myAction.py --docker $DOCKER_USER/action-python-v3.7
```

## Test Runtimes

There are suites of tests that are generic for all runtimes, and some that are specific to a runtime version.
To run all tests, there are two steps.

First, you need to create an OpenWhisk snapshot release. Do this from your OpenWhisk home directory.
```
./gradlew install
```

Now you can build and run the tests in this repository.
```
./gradlew tests:test
```

Gradle allows you to selectively run tests. For example, the following
command runs tests which match the given pattern and excludes all
others.
```
./gradlew :tests:test --tests Python*Tests
```

## Python 3 AI Runtime
This action runtime enables developers to create AI Services with OpenWhisk. It comes with preinstalled libraries useful for running Machine Learning and Deep Learning inferences. [Read more about this runtime here](./core/python36AiAction).

## Using additional python libraries

If you need more libraries for your Python action,  you can include a virtualenv in the zip file of the action.

The requirement is that the zip file must have a subfolder named `virtualenv` with a script `virtualenv\bin\activate_this.py` working in an Linux AMD64 environment. It will be executed at start time to use your extra libraries.

### Using requirements.txt

Python virtual environments are typically built by installing dependencies listed in a `requirements.txt` file. If you have an action that requires additional libraries, you can include a `requirements.txt` file.

You have to create a folder `myaction` with at least two files:

```
__main__.py
requirements.txt
```

Then zip your action and deploy to OpenWhisk, the requirements will be installed for you at init time, creating a suitable virtualenv.

Keep in mind that resolving requirements involves downloading and install software, so your action timeout limit may need to be adjusted accordingly. Instead, you should consider using precompilation to resolve the requirements at build time.

### Precompilation of a virtualenv

The action containers can actually generate a virtualenv for you, provided you have a requirements.txt.


If you have an action in the format described before (with a `requirements.txt`) you can build the zip file with the included files with:

```
zip -j -r myaction | docker run -i action-python-v3.7 -compile main > myaction.zip
```

You may use `v3.9` or `v3.6-ai` as well according to your Python version needs.

The resulting action includes a virtualenv already built for you and that is fast to deploy and start as all the dependencies are already resolved. Note that there is a limit on the size of the zip file and this approach will not work for installing large libraries like Pandas or Numpy, instead use the provide "v.3.6-ai"  runtime instead which provides these libraries already for you.
