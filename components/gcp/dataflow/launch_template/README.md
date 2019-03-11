
# Submitting a job to the Google Cloud Dataflow service using a dataflow template <!-- this is the title-->

## Labels
GCP, BigQuery

## Summary

<!--- why the user wants to use this
- key technologies used -->


## Details

### Intended Use

A Kubeflow Pipeline component to submit a job from a dataflow template to Google Cloud Dataflow service.

### Runtime Parameters: <!--- input the missing details-->
Argument | Description | Optional | Data type | Accepted values | Default |
:--- | :---- | :--- | :---- | :--- | :---- | 
project_id | Required. The ID of the Cloud Platform project that the job belongs to.| | | | |
gcs_path | Required. A Cloud Storage path to the template from which to create the job. Must be valid Cloud Storage URL, beginning with 'gs://'.| | | | |
launch_parameters | Parameters to provide to the template being launched. Schema defined in https://cloud.google.com/dataflow/docs/reference/rest/v1b3/LaunchTemplateParameters. `jobName` will be replaced by generated name. | | | | |
location | Optional. The regional endpoint to which to direct the request. | | | | |
job_name_prefix |  Optional. The prefix of the genrated job name. If not provided, the method will generated a random name.
validate_only | If true, the request is validated but not actually executed. Defaults to false. | | | | |
wait_interval | Optional wait interval between calls to get job status. Defaults to 30. | | | | |

### Input data schema
<!--Description of the inputs (data, parameters, scripts, etc.) the pipeline ingests.
* P0: High-level description of inputs
* P1: Fields & definitions
* P1: Sample data (text/images/etc.)
* P1: Data formatting requirements. Size and other limitations.
* P2: Link to sample datasets if available
Make sure the customer is clear on what to provide.
Is it one file or several? What format(s)?
If the input is identified by a path to a location (like a local directory, or Cloud Storage bucket), what exactly should be in that location? A file with a particular name, format, etc.?
If it is a script, what is the expected language?
If input data is specified in runtime parameters, make sure the arguments mentioned here and in the runtime parameter table are identical.-->

### Output:
<!--Description of what a deployed pipeline generates.
* P0: High-level description of output
* P1: Fields & definitions
* P1: Sample data
* P1: Side-effects (e.g. data transformed along the way)
What is the output, exactly? Trained models, model metrics, predictions, transformed data?
Where can the customer find the output? If the output location is specified in runtime parameters, make sure the arguments mentioned here and in the runtime parameter table are identical.
How many files will there be, and what formats will they be in?
Anything else the customer should know about the expected file names, sizes, or other metadata?  -->

Name | Description
:--- | :----------
job_id | The id of the created dataflow job.

### Caution and requirements
<!--P1: Data requirements (you need at least X samples…)
* P1: Cluster requirements
* P1: Resource access (external/3rd party services it needs access to)
* P1: Other prerequisites (e.g. access tokens for 3rd party solutions)
* P1: Known issues & limitations
* P1: Ethical considerations
* P2: Data assumptions/limitations (examples: implications of models being trained will be enriched with Google-proprietary data, face detection was trained on biased data)-->

If there are any known issues identified, see if there’s any troubleshooting information that can be provided so the customer can resolve them.
Does this asset use any services or APIs? Mention that dependency so users can properly configure their environment.

### Performance and Metrics

<!--Generated metrics and other indicators to evaluate the impact of deploying the pipeline
* P2: Latency
* P2: Accuracy
* P2: Cost
* P2: Benchmarks 
If possible, describe the environment in which the above measurements were made so the customer can compare apples to apples. -->


### Detailed Description

<!--Description of the inner workings and the components of the pipeline (especially if multi-step pipeline).
* P0: How to run it locally and how to use it in a pipeline
* P1: Description of each step/container in the pipeline (example)
* P1: High-level purpose of each step (e.g. preprocessing, training, serving)
* P1: Static DAG (alpha: screenshot manually inserted by publisher)
* P2: Input/Output & inner workings of each step
* P2: Links to docker files and source used to create each container
* P2: Links to code (Python, etc.) 
If open source: Link to source code, location of container, type of registry, etc.
If a download: Describe the artifact (file type, name, size)
Most important is that how to use the pipeline or pipeline component is crystal clear. If there are multiple ways to use it (like locally and then deployed in the cloud), make sure each way has a separate, complete procedure describing how to use it (install/run/deploy/etc).
For pipelines, make sure that the version of Kubeflow that was used to create the pipeline is stated.
If the asset uses a particular algorithm, training approach, or other defining feature, make sure there’s a sentence or two about what it is and why it is being used (what’s the value compared to other approaches?).
Any information about technologies or environment that is good to know but not key enough to go in the summary or intended use sections should go here. For example, whether or not a Tensorflow model uses eager execution.--->


#### Sample

Note: the sample code below works in both IPython notebook or python code directly.

#### Set sample parameters


```python
# Required Parameters
PROJECT_ID = '<Please put your project ID here>'
GCS_WORKING_DIR = 'gs://<Please put your GCS path here>' # No ending slash

# Optional Parameters
EXPERIMENT_NAME = 'Dataflow - Launch Template'
COMPONENT_SPEC_URI = 'https://raw.githubusercontent.com/kubeflow/pipelines/master/components/gcp/dataflow/launch_template/component.yaml'
```

#### Install KFP SDK


```python
# Install the SDK (Uncomment the code if the SDK is not installed before)
# KFP_PACKAGE = 'https://storage.googleapis.com/ml-pipeline/release/0.1.11/kfp.tar.gz'
# !pip3 install $KFP_PACKAGE --upgrade
```

#### Load component definitions


```python
import kfp.components as comp

dataflow_template_op = comp.load_component_from_url(COMPONENT_SPEC_URI)
display(dataflow_template_op)
```

#### Here is an illustrative pipeline that uses the component


```python
import kfp.dsl as dsl
import kfp.gcp as gcp
import json
@dsl.pipeline(
    name='Dataflow launch template pipeline',
    description='Dataflow launch template pipeline'
)
def pipeline(
    project_id, 
    gcs_path, 
    launch_parameters, 
    location='', 
    job_name_prefix='', 
    validate_only='', 
    wait_interval = 30
):
    dataflow_template_op(project_id, gcs_path, launch_parameters, location, job_name_prefix, validate_only, 
        wait_interval).apply(gcp.use_gcp_secret('user-gcp-sa'))
```

#### Compile the pipeline


```python
pipeline_func = pipeline
pipeline_filename = pipeline_func.__name__ + '.pipeline.tar.gz'
import kfp.compiler as compiler
compiler.Compiler().compile(pipeline_func, pipeline_filename)
```

#### Submit the pipeline for execution


```python
#Specify pipeline argument values
arguments = {
    'project_id': PROJECT_ID,
    'gcs_path': 'gs://dataflow-templates/latest/Word_Count',
    'launch_parameters': json.dumps({
       'parameters': {
           'inputFile': 'gs://dataflow-samples/shakespeare/kinglear.txt',
           'output': '{}/dataflow/launch-template/'.format(GCS_WORKING_DIR)
       }
    })
}

#Get or create an experiment and submit a pipeline run
import kfp
client = kfp.Client()
experiment = client.create_experiment(EXPERIMENT_NAME)

#Submit a pipeline run
run_name = pipeline_func.__name__ + ' run'
run_result = client.run_pipeline(experiment.id, run_name, pipeline_filename, arguments)
```
### References

<!--* P2: Links to papers
* P2: Links to publisher website -->
