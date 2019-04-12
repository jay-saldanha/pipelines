

# Name

Gather training data by querying BigQuery 


# Labels

GCP, BigQuery, Kubeflow, Pipeline


# Summary

A Kubeflow Pipeline component to submit a query to BigQuery and store the result in a Cloud Storage bucket.


# Details


## Intended use

Use this Kubeflow component to:
*   Select training data by submitting a query to BigQuery.
*   Output the training data into a Cloud Storage bucket as CSV files.


## Runtime arguments:


| Argument | Description | Optional | Data type | Accepted values | Default |
|----------|-------------|----------|-----------|-----------------|---------|
| query | The query used by BigQuery to fetch the results. | No | String |  |  |
| project_id | The project ID of the Google Cloud Platform (GCP) project to use to execute the query. | No | GCPProjectID |  |  |
| dataset_id | The ID of the persistent BigQuery dataset to store the results of the query. If the dataset does not exist, the operation will create a new one. | Yes | String |  | None |
| table_id | The ID of the BigQuery table to store the results of the query. If the table ID is absent, the operation will generate a random ID for the table. | Yes | String |  | None |
| output_gcs_path | The path to the Cloud Storage bucket to store the query output. | Yes | GCSPath |  | None |
| dataset_location | The location where the dataset is created. Defaults to US. | Yes | String |  | US |
| job_config | The full configuration specification for the query job. See [QueryJobConfig](https://googleapis.github.io/google-cloud-python/latest/bigquery/generated/google.cloud.bigquery.job.QueryJobConfig.html#google.cloud.bigquery.job.QueryJobConfig) for details. | Yes | Dict | A JSONobject which has the same structure as [QueryJobConfig](https://googleapis.github.io/google-cloud-python/latest/bigquery/generated/google.cloud.bigquery.job.QueryJobConfig.html#google.cloud.bigquery.job.QueryJobConfig) | None |
## Input data schema

The input data is a BigQuery job containing a query that pulls data f rom various sources. 


## Output:

Name | Description
:--- | :----------
output_gcs_path | The path to the Cloud Storage bucket containing the query output in CSV format.

## Cautions & requirements

To use the component, the following requirements must be met:

*   The BigQuery API is enabled.
*   The component is running under a secret [Kubeflow user service account](https://www.kubeflow.org/docs/started/getting-started-gke/#gcp-service-accounts) in a Kubeflow Pipeline cluster.  For example:

    ```
    bigquery_query_op(...).apply(gcp.use_gcp_secret('user-gcp-sa'))
    ```
*   The Kubeflow user service account is a member of the `roles/bigquery.admin` role of the project.
*   The Kubeflow user service account is a member of the `roles/storage.objectCreator `role of the Cloud Storage output bucket.

## Detailed description
This Kubeflow Pipeline component is used to:
*   Submit a query to BigQuery.
    *   The query results are persisted in a dataset table in BigQuery.
    *   An extract job is created in BigQuery to extract the data from the dataset table and output it to a Cloud Storage bucket as CSV files.

    Use the code below as an example of how to run your BigQuery job.

### Sample

Note: The following sample code works in an IPython notebook or directly in Python code.

#### Set sample parameters
```python
# Required Parameters
PROJECT_ID = '<Put your project ID here>'
GCS_WORKING_DIR = 'gs://<Put your GCS path here>' # No ending slash

# Optional Parameters
EXPERIMENT_NAME = 'Bigquery -Query'
COMPONENT_SPEC_URI = 'https://raw.githubusercontent.com/kubeflow/pipelines/master/components/gcp/bigquery/query/component.yaml'
```

#### Install Kubeflow Pipeline SDK
```python
# Install the SDK (Uncomment the code if the SDK is not installed before)
# KFP_PACKAGE = 'https://storage.googleapis.com/ml-pipeline/release/0.1.11/kfp.tar.gz'
# !pip3 install $KFP_PACKAGE --upgrade
```

#### Load component definitions
```python
import kfp.components as comp

bigquery_query_op = comp.load_component_from_url(COMPONENT_SPEC_URI)
display(bigquery_query_op)
```

#### Example pipeline that uses the component

```python
import kfp.dsl as dsl
import kfp.gcp as gcp
import json
@dsl.pipeline(
    name='Bigquery query pipeline',
    description='Bigquery query pipeline'
)
def pipeline(
    query, 
    project_id, 
    dataset_id='', 
    table_id='', 
    output_gcs_path='', 
    dataset_location='US', 
    job_config=''
):
    bigquery_query_op(query, project_id, dataset_id, table_id, output_gcs_path, dataset_location, 
        job_config).apply(gcp.use_gcp_secret('user-gcp-sa'))
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
    'query': 'SELECT * FROM `bigquery-public-data.stackoverflow.posts_questions` LIMIT 10',
    'project_id': PROJECT_ID,
    'output_gcs_path': '{}/bigquery/query/questions.csv'.format(GCS_WORKING_DIR)
}

#Get or create an experiment and submit a pipeline run
import kfp
client = kfp.Client()
experiment = client.create_experiment(EXPERIMENT_NAME)

#Submit a pipeline run
run_name = pipeline_func.__name__ + ' run'
run_result = client.run_pipeline(experiment.id, run_name, pipeline_filename, arguments)
```

## References
*   [Source code](https://github.com/kubeflow/pipelines/blob/master/component_sdk/python/kfp_component/google/bigquery/_query.py)
*   [Sample notebook](https://github.com/kubeflow/pipelines/blob/master/components/gcp/bigquery/query/sample.ipynb)
*   [Bigquery Query REST API](https://cloud.google.com/bigquery/docs/reference/rest/v2/jobs/query)

## License 
By deploying or using this software you agree to comply with the [AI Hub Terms of Service](https://aihub.cloud.google.com/u/0/aihub-tos) and the [Google APIs Terms of Service](https://developers.google.com/terms/). To the extent of a direct conflict of terms, the AI Hub Terms of Service will control.
