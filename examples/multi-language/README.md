<!--
    Licensed to the Apache Software Foundation (ASF) under one
    or more contributor license agreements.  See the NOTICE file
    distributed with this work for additional information
    regarding copyright ownership.  The ASF licenses this file
    to you under the Apache License, Version 2.0 (the
    "License"); you may not use this file except in compliance
    with the License.  You may obtain a copy of the License at

      http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing,
    software distributed under the License is distributed on an
    "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
    KIND, either express or implied.  See the License for the
    specific language governing permissions and limitations
    under the License.
-->

# Example multi-language pipelines

This project provides examples of Apache Beam
[multi-language pipelines](https://beam.apache.org/documentation/programming-guide/#multi-language-pipelines):

## Using Java transforms from Python

* **python/addprefix** - A Python pipeline that reads a text file and attaches a prefix on the Java side to each input.
* **python/javacount** - A Python pipeline that counts words using the Java `Count.perElement()` transform.
* **python/javadatagenerator** - A Python pipeline that produces a set of strings generated from Java.
                                  This example demonstrates the `JavaExternalTransform` API.

### Instructions for running the pipelines

#### 1) Start the expansion service

1. Download the latest 'beam-examples-multi-language' JAR. Starting with Apache Beam 2.36.0,
   you can find it in [the Maven Central Repository](https://search.maven.org/search?q=g:org.apache.beam).
2. Run the following command, replacing `<version>` and `<port>` with valid values:
  `java -jar beam-examples-multi-language-<version>.jar <port> --javaClassLookupAllowlistFile='*'`

#### 2) Set up a Python virtual environment for Beam

1. See [the Python quickstart](https://beam.apache.org/get-started/quickstart-py/)
   for more information.

#### 3) Execute the Python pipeline

1. In a new shell, run a pipeline in the **python** directory using a Beam runner that supports
   multi-language pipelines.

   The Python files contain details about the actual commands to run.

## Using Python transforms from Java

### Sklearn Mnist Classification

Performs image classification on handwritten digits from the [MNIST](https://en.wikipedia.org/wiki/MNIST_database)
database.

Please see [here](https://github.com/apache/beam/tree/master/sdks/python/apache_beam/examples/inference) for
context and information regarding the corresponding Python pipeline.

Please note that the Java pipeline is
[availalble in the Beam Java examples module](https://github.com/apache/beam/tree/master/examples/java/src/main/java/org/apache/beam/examples/multilanguage/SklearnMnistClassification.java).

#### Setup

* Obtain/generate a csv input file that contains labels and pixels to feed into the model and store it in
GCS. An example input is available
[here](https://storage.googleapis.com/apache-beam-samples/multi-language/mnist/example_input.csv).

* Create a model file that contains the pickled file of a scikit-learn model
trained on MNIST data and store it in GCS. An example model file is available
[here](https://storage.googleapis.com/apache-beam-samples/multi-language/mnist/example_model).
This model was generated by by running the program given
[here](https://python-course.eu/machine-learning/training-and-testing-with-mnist.php)
on the
[example input dataset](https://storage.googleapis.com/apache-beam-samples/multi-language/mnist/example_input.csv).

* Perform Beam runner specific setup according to instructions
[here](https://beam.apache.org/get-started/quickstart-java/#run-a-pipeline).

Following instructions are for running the pipeline with the Dataflow runner. For other portable runners,
please modify the instructions according to the guidelines
[here](https://beam.apache.org/documentation/sdks/java-multi-language-pipelines/#run-with-directrunner)

#### Instructions for running the Java pipeline on released Beam (Beam 2.43.0 and later).

* Checkout the Beam examples Maven archetype for the relevant Beam version.

```
export BEAM_VERSION=<Beam version>

mvn archetype:generate \
    -DarchetypeGroupId=org.apache.beam \
    -DarchetypeArtifactId=beam-sdks-java-maven-archetypes-examples \
    -DarchetypeVersion=$BEAM_VERSION \
    -DgroupId=org.example \
    -DartifactId=multi-language-beam \
    -Dversion="0.1" \
    -Dpackage=org.apache.beam.examples \
    -DinteractiveMode=false
```

* Run the pipeline.

```
export GCP_PROJECT=<GCP project>
export GCP_BUCKET=<GCP bucket>
export GCP_REGION=<GCP region>

mvn compile exec:java -Dexec.mainClass=org.apache.beam.examples.multilanguage.SklearnMnistClassification \
    -Dexec.args="--runner=DataflowRunner --project=$GCP_PROJECT \
                 --region=us-central1 \
                 --gcpTempLocation=gs://$GCP_BUCKET/multi-language-beam/tmp \
                 --output=gs://$GCP_BUCKET/multi-language-beam/output" \
    -Pdataflow-runner
```

* Inspect the output. Each line has data separated by a comma ",". The first item is the actual label of
the digit. The second item is the predicted label of the digit.

```
gsutil cat gs://$GCP_BUCKET/multi-language-beam/output*
```

#### Instructions for running the Java pipeline at HEAD (Beam 2.41.0 and 2.42.0).

* Activate a new virtual environment following
[these instructions](https://beam.apache.org/get-started/quickstart-py/#create-and-activate-a-virtual-environment).

* 2. Install Apache Beam package with gcp support and the `sklearn` package.

```
pip install apache-beam[gcp]
pip install sklearn
```

* Startup the expansion service

```
python -m apache_beam.runners.portability.expansion_service_main -p <PORT> --fully_qualified_name_glob "*"
```

* Make sure that Docker is installed and available on your system.

* In a different shell, build and push Python and Java Docker containers.

```
export DOCKER_ROOT=<Docker root>

./gradlew :sdks:python:container:py38:docker -Pdocker-repository-root=$DOCKER_ROOT -Pdocker-tag=latest

docker push $DOCKER_ROOT/beam_python3.8_sdk:latest

./gradlew :sdks:java:container:java11:docker -Pdocker-repository-root=$DOCKER_ROOT -Pdocker-tag=latest -Pjava11Home=$JAVA_HOME

docker push $DOCKER_ROOT/beam_java11_sdk:latest
```

* Run the pipeline using the following Gradle command (this guide assumes Dataflow runner).
Note that we override both the Java and Python SDK harness containers here.

```
export GCP_PROJECT=<GCP project>
export GCP_BUCKET=<GCP bucket>
export GCP_REGION=<GCP region>
export EXPANSION_SERVICE_PORT=<PORT>

# This removes any existing output.
gsutil rm gs://$GCP_BUCKET/multi-language-beam/output*

./gradlew :examples:multi-language:sklearnMinstClassification --args=" \
--runner=DataflowRunner \
--project=$GCP_PROJECT \
--gcpTempLocation=gs://$GCP_BUCKET/multi-language-beam/tmp \
--output=gs://$GCP_BUCKET/multi-language-beam/output \
--sdkContainerImage=$DOCKER_ROOT/beam_java11_sdk:latest \
--sdkHarnessContainerImageOverrides=.*python.*,$DOCKER_ROOT/beam_python3.8_sdk:latest \
--expansionService=localhost:$EXPANSION_SERVICE_PORT \
--region=${GCP_REGION}"
```

* Inspect the output. Each line has data separated by a comma ",". The first item is the actual label
of the digit. The second item is the predicted label of the digit.

```
gsutil cat gs://$GCP_BUCKET/multi-language-beam/output*
```

### Python Dataframe Wordcount

This example is covered in the [Java multi-language pipelines quickstart](https://beam.apache.org/documentation/sdks/java-multi-language-pipelines/).
The pipeline source code is available at
[PythonDataframeWordCount.java](https://github.com/apache/beam/tree/master/examples/java/src/main/java/org/apache/beam/examples/multilanguage/PythonDataframeWordCount.java).