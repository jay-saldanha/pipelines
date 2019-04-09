# Name
Data preparation by executing an Apache Beam job in Cloud Dataflow

# Labels
GCP, Cloud Dataflow, Apache Beam, Python, Kubeflow

# Summary
A Kubeflow Pipeline component that prepares data by submitting an Apache Beam job (authored in Python) to Cloud Dataflow for execution. The Python Beam code is run with Cloud Dataflow Runner.

# Details
## Intended use

Use this component to run a Python Beam code to submit a Cloud Dataflow job as a step of a Kubeflow pipeline. 

## Runtime arguments
Name | Description | Optional |  Data type| Accepted values | Default |
:--- | :----------| :----------| :----------| :----------| :---------- |
python_file_path |  The path to the Cloud Storage bucket or local directory containing the Python file to be run. |  |  GCSPath |  |  |
project_id |  The ID of the Google Cloud Platform (GCP) project  containing the Cloud Dataflow job.| | GCPProjectID | | |
requirements_file_path |   The path to the Cloud Storage bucket or local directory containing the pip requirements file. | Yes | GCSPath |  | None |
location |  The regional endpoint to which the job request is directed. | Yes |  GCPRegion  |   |   None  |
staging_dir  |   The path to the Cloud Storage directory where the staging files are stored. A random subdirectory will be created under the staging directory to keep the  job information.This is done so that you can resume the job in case of failure. `staging_dir` is passed as the command line arguments (`staging_location` and `temp_location`) of the Beam code. |   Yes  |   GCPPath  |   |   None  |
args |  The list of arguments to pass to the Python file. | No |  List | A list of string arguments | None |
wait_interval |  The number of seconds to wait between calls to get the status of the job. | Yes | Integer  |  | 30 |

## Input data schema

Before you use the component, the following files must be ready in a Cloud Storage bucket:
- A Beam Python code file.
- A  `requirements.txt` file which includes a list of dependent packages.

The Beam Python code should follow the [Beam programming guide](https://beam.apache.org/documentation/programming-guide/) as well as the following additional requirements to be compatible with this component:
- It accepts the command line arguments `--project`, `--temp_location`, `--staging_location`, which are [standard Dataflow Runner options](https://cloud.google.com/dataflow/docs/guides/specifying-exec-params#setting-other-cloud-pipeline-options).
- It enables `info logging` before the start of a Cloud Dataflow job in the Python code. This is important to allow the component to track the status and ID of the job that is created. For example, calling `logging.getLogger().setLevel(logging.INFO)` before any other code.


## Output
Name | Description
:--- | :----------
job_id | The id of the Cloud Dataflow job that is created.

## Cautions & requirements
To use the components, the following requirements must be met:
- Cloud Dataflow API is enabled.
- The component is running under a secret Kubeflow user service account in a Kubeflow Pipeline cluster.  For example:
```
component_op(...).apply(gcp.use_gcp_secret('user-gcp-sa'))
```
The Kubeflow user service account is a member of:
- `roles/dataflow.developer` role of the project.
- `roles/storage.objectViewer` role of the Cloud Storage Objects `python_file_path` and `requirements_file_path`.
- `roles/storage.objectCreator` role of the Cloud Storage Object `staging_dir`. 

## Detailed description
The component does several things during the execution:
- Downloads `python_file_path` and `requirements_file_path` to local files.
- Starts a subprocess to launch the Python program.
- Monitors the logs produced from the subprocess to extract the Cloud Dataflow job information.
- Stores the Cloud Dataflow job information in `staging_dir` so the job can be resumed in case of failure.
- Waits for the job to finish.
The steps to use the component in a pipeline are:
1. Install the Kubeflow Pipelines SDK:
   ```python
   %%capture --no-stderr

   KFP_PACKAGE = 'https://storage.googleapis.com/ml-pipeline/release/0.1.14/kfp.tar.gz'
   !pip3 install $KFP_PACKAGE --upgrade
   ```
2. Load the component using the Kubeflow Pipelines SDK:
   ```python
   import kfp.components as comp

   dataflow_python_op = comp.load_component_from_url(
    'https://raw.githubusercontent.com/kubeflow/pipelines/d2f5cc92a46012b9927209e2aaccab70961582dc/components/gcp/dataflow/launch_python/component.yaml') 
    help(dataflow_python_op)``` 

### Sample
Note: The following sample code works in an IPython notebook or directly in Python code. See the sample code below to learn how to execute the template.
In this sample, we run a wordcount sample code in a Kubeflow Pipeline. The output will be stored in a Cloud Storage bucket. Here is the sample code:
```python
!gsutil cat gs://ml-pipeline-playground/samples/dataflow/wc/wc.py
```


#### Set sample parameters
```python
# Required Parameters
PROJECT_ID = '<Put your project ID here>'
GCS_WORKING_DIR = 'gs://<Put your GCS path here>' # No ending slash

# Optional Parameters
EXPERIMENT_NAME = 'Dataflow - Launch Python'
COMPONENT_SPEC_URI = 'https://raw.githubusercontent.com/kubeflow/pipelines/master/components/gcp/dataflow/launch_python/component.yaml'
``` 

#### Example pipeline that uses the component


```python
import kfp.dsl as dsl
import kfp.gcp as gcp
import json
@dsl.pipeline(
    name='Dataflow launch python pipeline',
    description='Dataflow launch python pipeline'
)
def pipeline(
    python_file_path,
    project_id,
    requirements_file_path = '',
    location = '',
    job_name_prefix = '',
    args = '',
    wait_interval = 30
):
    dataflow_python_op(python_file_path, project_id, requirements_file_path, location, job_name_prefix, args,
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
    'python_file_path': 'gs://ml-pipeline-playground/samples/dataflow/wc/wc.py',
    'project_id': PROJECT_ID,
    'requirements_file_path': 'gs://ml-pipeline-playground/samples/dataflow/wc/requirements.txt',
    'args': json.dumps([
        '--output', '{}/wc/wordcount.out'.format(GCS_WORKING_DIR),
        '--temp_location', '{}/dataflow/wc/tmp'.format(GCS_WORKING_DIR),
        '--staging_location', '{}/dataflow/wc/staging'.format(GCS_WORKING_DIR)
    ])
}

#Get or create an experiment and submit a pipeline run
import kfp
client = kfp.Client()
experiment = client.create_experiment(EXPERIMENT_NAME)

#Submit a pipeline run
run_name = pipeline_func.__name__ + ' run'
run_result = client.run_pipeline(experiment.id, run_name, pipeline_filename, arguments)
```
#### Inspect the output
```python
!gsutil cat $OUTPUT_FILE 
```

## References
- [Component Python code](https://github.com/kubeflow/pipelines/blob/master/component_sdk/python/kfp_component/google/dataflow/_launch_python.py)
- [Component Docker file](https://github.com/kubeflow/pipelines/blob/master/components/gcp/container/Dockerfile)
- [Sample notebook](https://github.com/kubeflow/pipelines/blob/master/components/gcp/dataflow/launch_python/sample.ipynb)
- [Dataflow Python Quickstart](https://cloud.google.com/dataflow/docs/quickstarts/quickstart-python)
