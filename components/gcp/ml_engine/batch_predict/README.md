# Name

Component: Submitting a batch prediction job to AI Platform

# Labels

AI Platform, Kubeflow


# Summary

A Kubeflow pipeline component to submit a batch prediction job against a trained model to AI Platform. <!--can you rephrase this? what do you mean by "against a trained model?-->

# Facets
<!--Make sure the asset has data for the following facets:
Use case
Technique
Input data type
ML workflow

The data must map to the acceptable values for these facets, as documented on the “taxonomy” sheet of go/aihub-facets
https://gitlab.aihub-content-external.com/aihubbot/kfp-components/commit/fe387ab46181b5d4c7425dcb8032cb43e70411c1
--->
Use case:

Technique: 

Input data type:

ML workflow: 


# Details
## Intended use
Use this component to submit a batch prediction job against a trained model to AI Platform.

## Runtime arguments
<!--add missing details in the table -->
 Argument | Description | Optional | Data type | Accepted values | Default |
|:------- |:------------|:---------|:----------|:----------------|:--------|
project_id | The ID of the parent project of the job. |No | -|- |-
model_path | The path to the model. It can be one of these: <ul><li>`projects/[PROJECT_ID]/models/[MODEL_ID]`</li> <li>`projects/[PROJECT_ID]/models/[MODEL_ID]/versions/[VERSION_ID]` </li> <li>A Cloud Storage path to the model file.</li></ul>|No |-|-|-
input_paths | The Cloud Storage location of the input data files. It can contain wildcards.|No  | -|-|-
input_data_format | The format of the input data files. See [DatFormat](https://cloud.google.com/ml-engine/reference/rest/v1/projects.jobs#DataFormat).|No |-|-|-
output_path | The Cloud Storage location of the output file.|No |-|-|-
region | The Compute Engine region in which you run the prediction job.|No |-|-|-
output_data_format | The format of the output data files.|Yes|-|-|JSON
prediction_input | The input parameters to create a prediction job. See [PredictionInput](https://cloud.google.com/ml-engine/reference/rest/v1/projects.jobs#PredictionInput).|-|-|-|-
job_id_prefix | The prefix of the generated job id.|-|-|-|-
wait_interval | The number of seconds to wait in case the operation has a long runtime. |Yes|-|-|30

## Input data schema
<!--add missing details here-->

## Output:
| Name    | Description                 | Type      |
|:------- |:----                        | :---      |
job_id | The ID of the created batch job.|String

## Cautions & requirements
<!--add missing details here -->

## Detailed description

The following sample code works in IPython notebook or directly in Python code.

### Set sample parameters

```python
# Required parameters
PROJECT_ID = '<Put your project ID here>'
GCS_WORKING_DIR = 'gs://<Put your GCS path here>' # No ending slash

# Optional Parameters
EXPERIMENT_NAME = 'CLOUDML - Batch Predict'
COMPONENT_SPEC_URI = 'https://raw.githubusercontent.com/kubeflow/pipelines/master/components/gcp/ml_engine/batch_predict/component.yaml'
```

### Install the Kubeflow pipeline's SDK

```python
# Install the SDK (Uncomment the code if the SDK is not installed before)
# KFP_PACKAGE = 'https://storage.googleapis.com/ml-pipeline/release/0.1.11/kfp.tar.gz'
# !pip3 install $KFP_PACKAGE --upgrade
```

### Load the component's definitions

```python
import kfp.components as comp

mlengine_batch_predict_op = comp.load_component_from_url(COMPONENT_SPEC_URI)
display(mlengine_batch_predict_op)
```

### Example pipeline that uses the component

```python
import kfp.dsl as dsl
import kfp.gcp as gcp
import json
@dsl.pipeline(
    name='CloudML batch predict pipeline',
    description='CloudML batch predict pipeline'
)
def pipeline(
    project_id, 
    model_path, 
    input_paths, 
    input_data_format, 
    output_path, 
    region, 
    output_data_format='', 
    prediction_input='', 
    job_id_prefix='',
    wait_interval='30'):
    task = mlengine_batch_predict_op(project_id, model_path, input_paths, input_data_format, 
    output_path, region, output_data_format, prediction_input, job_id_prefix,
    wait_interval).apply(gcp.use_gcp_secret('user-gcp-sa'))
```

### Compile the pipeline

```python
pipeline_func = pipeline
pipeline_filename = pipeline_func.__name__ + '.pipeline.tar.gz'
import kfp.compiler as compiler
compiler.Compiler().compile(pipeline_func, pipeline_filename)
```

### Submit the pipeline for execution

```python
#Specify values for the pipeline's arguments
arguments = {
    'project_id': PROJECT_ID,
    'model_path': 'gs://ml-pipeline-playground/samples/ml_engine/census/trained_model/',
    'input_paths': '["gs://ml-pipeline-playground/samples/ml_engine/census/test.json"]',
    'input_data_format': 'JSON',
    'output_path': GCS_WORKING_DIR + '/batch_predict/output/',
    'region': 'us-central1',
    'prediction_input': json.dumps({
        'runtimeVersion': '1.10'
    })
}

#Get or create an experiment 
import kfp
client = kfp.Client()
experiment = client.create_experiment(EXPERIMENT_NAME)

#Submit a pipeline run
run_name = pipeline_func.__name__ + ' run'
run_result = client.run_pipeline(experiment.id, run_name, pipeline_filename, arguments)
```

## References
<!--add missing details here-->
[PredictionInput](https://cloud.google.com/ml-engine/reference/rest/v1/projects.jobs#PredictionInput)
[DatFormat](https://cloud.google.com/ml-engine/reference/rest/v1/projects.jobs#DataFormat)
## License
By deploying or using this software you agree to comply with the [AI Hub Terms of Service](https://aihub.cloud.google.com/u/0/aihub-tos) and the [Google APIs Terms of Service](https://developers.google.com/terms/). To the extent of a direct conflict of terms, the AI Hub Terms of Service will control.
