# Build batch score using ParallelRunStep and store output to blob

## Flow Chart to show the process

![alt text](https://github.com/balakreshnan/mlops/blob/master/images/mlopsstepsmlparallelscorebatch.jpg "mlops Parallel Step")

Code sample below

```
import os
import urllib
import shutil
import azureml
import pandas as pd

from azureml.core import Experiment
from azureml.core import Workspace, Run

from azureml.core.compute import ComputeTarget, AmlCompute
from azureml.core.compute_target import ComputeTargetException
```

```
ws = Workspace.from_config()
```

```
project_folder = './touring-project'
os.makedirs(project_folder, exist_ok=True)

experiment = Experiment(workspace=ws, name='touring-model')
```

```
output_folder = './outputs'
os.makedirs(output_folder, exist_ok=True)
```

```
result_folder = './results'
os.makedirs(result_folder, exist_ok=True)
```

```
from azureml.core.compute import ComputeTarget, AmlCompute
from azureml.core.compute_target import ComputeTargetException

# Choose a name for your CPU cluster
cpu_cluster_name = "touringcluster"

# Verify that cluster does not exist already
try:
    cpu_cluster = ComputeTarget(workspace=ws, name=cpu_cluster_name)
    print('Found existing cluster, use it.')
except ComputeTargetException:
    compute_config = AmlCompute.provisioning_configuration(vm_size='STANDARD_D14_V2',
                                                           max_nodes=4)
    cpu_cluster = ComputeTarget.create(ws, cpu_cluster_name, compute_config)

cpu_cluster.wait_for_completion(show_output=True)
```

```
from azureml.core.runconfig import RunConfiguration
from azureml.core.conda_dependencies import CondaDependencies
from azureml.core.runconfig import DEFAULT_CPU_IMAGE

# Create a new runconfig object
run_amlcompute = RunConfiguration()

# Use the cpu_cluster you created above. 
run_amlcompute.target = cpu_cluster

# Enable Docker
run_amlcompute.environment.docker.enabled = True

# Set Docker base image to the default CPU-based image
run_amlcompute.environment.docker.base_image = DEFAULT_CPU_IMAGE

# Use conda_dependencies.yml to create a conda environment in the Docker image for execution
run_amlcompute.environment.python.user_managed_dependencies = False

# Specify CondaDependencies obj, add necessary packages
run_amlcompute.environment.python.conda_dependencies = CondaDependencies.create(conda_packages=['scikit-learn'])
```

```
%%writefile batch_scoring.py
import io
import pickle
import argparse
import numpy as np
import pandas as pd

import joblib
import os
import urllib
import shutil
import azureml

from azureml.core.model import Model
from sklearn.linear_model import LogisticRegression
from sklearn.ensemble import RandomForestClassifier


def init():
    global touring_model

    model_path = Model.get_model_path("sklearn_touring")

    #model_path = Model.get_model_path(args.model_name)
    with open(model_path, 'rb') as model_file:
        touring_model = joblib.load(model_file)


def run(mini_batch):
    # make inference    
    print(f'run method start: {__file__}, run({mini_batch})')
    resultList = []
    for file in mini_batch:
        input_data = pd.read_parquet(file, engine='pyarrow')
        num_rows, num_cols = input_data.shape
        X = input_data[[col for col in input_data.columns if "encoding" in col]]
        y = input_data['touring_flag']

        X = X[[col for col in X.columns if col not in ["income_premium_range_encoding", "networth_encoding"]]]
        pred = touring_model.predict(X).reshape((num_rows, 1))

    # cleanup output
    #result = input_data.drop(input_data.columns[4:], axis=1)
    result = X
    result['variety'] = pred

    return result
```

```
from azureml.core.datastore import Datastore

batchscore_blob = Datastore.register_azure_blob_container(ws, 
                      datastore_name="inputdatastore", 
                      container_name="containername", 
                      account_name="storageaccoutnname",
                      account_key="xxxxx",
                      overwrite=True)

def_data_store = ws.get_default_datastore()
```

```
from azureml.core.datastore import Datastore

batchscore_blob_out = Datastore.register_azure_blob_container(ws, 
                      datastore_name="input_datastore", 
                      container_name="containername", 
                      account_name="storageaccountname", 
                      account_key="xxxxxxx",
                      overwrite=True)

def_data_store_out = ws.get_default_datastore()
```

```
from azureml.core.dataset import Dataset
from azureml.pipeline.core import PipelineData

input_ds = Dataset.File.from_files((batchscore_blob, "/"))
output_dir = PipelineData(name="scores", 
                          datastore=def_data_store_out, 
                          output_path_on_compute="results")
```

```
from azureml.core import Environment
from azureml.core.conda_dependencies import CondaDependencies
from azureml.core.runconfig import DEFAULT_CPU_IMAGE

cd = CondaDependencies.create(pip_packages=["scikit-learn", "azureml-defaults","pyarrow"])
env = Environment(name="parallelenv")
env.python.conda_dependencies = cd
env.docker.base_image = DEFAULT_CPU_IMAGE
```

```
from azureml.pipeline.core import PipelineParameter
from azureml.pipeline.steps import ParallelRunConfig

parallel_run_config = ParallelRunConfig(
    #source_directory=scripts_folder,
    entry_script="batch_scoring.py",
    mini_batch_size='1',
    error_threshold=2,
    output_action='append_row',
    append_row_file_name="tuoring_outputs.txt",
    environment=env,
    compute_target=cpu_cluster, 
    node_count=3,
    run_invocation_timeout=120)
```

```
from azureml.pipeline.steps import ParallelRunStep

batch_score_step = ParallelRunStep(
    name="parallel-step-test",
    inputs=[input_ds.as_named_input("input_ds")],
    output=output_dir,
    #models=[model],
    parallel_run_config=parallel_run_config,
    #arguments=['--model_name', 'sklearn_touring'],
    allow_reuse=True
)
```

```
from azureml.core import Experiment
from azureml.pipeline.core import Pipeline

pipeline = Pipeline(workspace=ws, steps=[batch_score_step])
pipeline_run = Experiment(ws, 'batch_scoring').submit(pipeline)
pipeline_run.wait_for_completion(show_output=True)
```

```
from azureml.widgets import RunDetails
RunDetails(pipeline_run).show()
```

```

End of pipeline run.