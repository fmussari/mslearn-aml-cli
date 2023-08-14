## Lab 01. Create an Azure Machine Learning workspace and assets with the CLI (v2)

```azurecli
az extension list
az extension list -o table
az extension list --query "[].{name: name, version: version}" -o table
```

### Create an Azure resource group and set as default
```azurecli
az group create --name "rg-mlops-labs" --location "eastus"
az configure --defaults group="rg-mlops-labs"
```

### Create an Azure Machine Learning workspace and set as default
No need to specify Resource Group since it was set as default.
```azurecli
az ml workspace create --name "mlw-mlops-labs"
az configure --defaults workspace="mlw-mlops-labs"
```

### Create a compute instance in your workspace:  
No need to specify Resource Group and ML Workspace since they were set as defaults.

```azurecli
az ml compute create --name "ci71974" --size STANDARD_DS11_V2 --type ComputeInstance
```

### Create an environment
```azurecli
az ml environment create --file ./Allfiles/Labs/01/basic-env.yml
```

### Create a dataset
```azurecli
az ml data create --file ./Allfiles/Labs/01/data-local-path.yml
```
This should be equivalent ([reference](https://learn.microsoft.com/en-us/cli/azure/ml/data?view=azure-cli-latest#az-ml-data-create)):
```azurecli
az ml data create --name diabetes-data --version 1 --path ./Allfiles/Labs/01/data -d "Dataset pointing to diabetes data stored as CSV on local computer. Data is uploaded to default datastore."
```
### Clean up resources
```azurecli
az ml compute stop --name "ci71974" --no-wait
az ml workspace delete -n mlw-mlops-labs -g rg-mlops-labs
```

## Lab 02. Run a basic Python training job

### Start the instance again

```azurecli
az ml compute start --name "ci71974"
```
Or create if deleted:
```azurecli
az ml compute create --name "ci71974" --size STANDARD_DS11_V2 --type ComputeInstance
```

### Train a model
```azurecli
az ml job create --file ./Allfiles/Labs/02/basic-job/basic-job.yml --web
```

### Train a model with dataset from datastore
```azurecli
az ml job create --file ./Allfiles/Labs/02/input-data-job/data-job.yml --web
```

### Clean up resources
```azurecli
az ml compute stop --name "ci71974" --no-wait
```

## Lab 03. Run a sweep job to tune hyperparameters

### Prerequisites
To train multiple models in parallel, you’ll use a compute cluster to train the models. 

```azurecli
az ml compute create --name "aml-cluster-71974" --size STANDARD_DS11_V2 --max-instances 2 --type AmlCompute
```

### Run a sweep job

- **mslearn-aml-cli/Allfiles/Labs/02/sweep-job** -> **sweep-job.yml**

```azurecli
az ml job create --file ./mslearn-aml-cli/Allfiles/Labs/02/sweep-job/sweep-job.yml
```

### Clean up resources
```azurecli
az ml compute delete --name "aml-cluster-71974" --no-wait --yes
```

## Lab 04. Track Azure ML jobs with MLflow

### Start the instance again
```azurecli
az ml compute start --name "ci71974"
```

### Enable autologging

- Note that you’ll run the **mlflow-autolog.py**
```azurecli
az ml job create --file ./mslearn-aml-cli/Allfiles/Labs/03/mlflow-job/mlflow-job.yml
```
### Use logging functions to track custom metrics

- Note that you’ll run the **custom-mlflow.py**
```azurecli
az ml job create --file ./mslearn-aml-cli/Allfiles/Labs/03/mlflow-job/custom-mlflow-job.yml
```

### Extra from *[Manage models with MLflow](https://learn.microsoft.com/en-us/training/modules/use-mlflow-azure-machine-learning-jobs-submitted-cli-v2/3-manage-models-mlflow)*

```azurecli
az ml model list
```

The output in the shell will show you the summary information of the job you submitted. It will show you the inputs you defined in the **mlflow-job.yml** file, and the details that Azure Machine Learning adds, like the name.  
  
To download the output files using the CLI (2), first, set the current directory of the shell to where you want to download all job-related files to. Then, to actually download the files for a specific job, you use the following command:

```azurecli
az ml job download --name <name>
```

To register a model, use the `ml model create` command.
(By looking at the documentation it seems that `--local-path` is not the correct argument but `--path` instead)
```azurecli
az ml model create --name churn-example --version 1 --local-path <name>/model/
```
  
How would it be to register the model without downloading it? Maybe like this:

```azurecli
az ml model create --name my-model --version 1 --path runs:/<name>/model/ --type mlflow_model
```

### Clean up resources
```azurecli
az ml compute delete --name "ci71974" --no-wait --yes
```

## Lab 05. Deploy a model to a managed online endpoint

### Deploy a model

#### create the endpoint
...the name of the endpoint (setted in **create-endpoint.yml**) must be unique in the Azure region.

```azurecli
az ml online-endpoint create --name diabetes-mlflow -f ./mslearn-aml-cli/Allfiles/Labs/04/mlflow-endpoint/create-endpoint.yml
```
#### deploy the model
```azurecli
az ml online-deployment create --name mlflow-deployment --endpoint diabetes-mlflow -f ./mslearn-aml-cli/Allfiles/Labs/04/mlflow-endpoint/mlflow-deployment.yml --all-traffic
```

### Extra from [*Deploy your model to a managed endpoint*](https://learn.microsoft.com/en-us/training/modules/deploy-azure-machine-learning-model-managed-endpoint-cli-v2/3-deploy-model-managed-endpoint)

For example, if after testing you want to reroute all traffic to the green deployment, which uses the newest version of the model, use the following command:

```azurecli
az ml online-endpoint update --name diabetes-mlflow --traffic "blue=0 green=100"
```

### Test the endpoint

```azurecli
az ml online-endpoint invoke --name diabetes-mlflow --request-file ./mslearn-aml-cli/Allfiles/Labs/04/mlflow-endpoint/sample-data.json
```

### Clean up resources

```azurecli
az ml online-endpoint delete --name diabetes-mlflow --yes --no-wait
```

## Lab 06. Run a pipeline with components

### Start compute instance (if needed)

```azurecli
az ml compute start --name "ci71974"
``` 

### Run a pipeline

```azurecli
az ml job create --file ./mslearn-aml-cli/Allfiles/Labs/05/job.yml
``` 

### Create components
To reuse the pipeline’s components, you can create the component in the Azure Machine Learning workspace. 

```azurecli
az ml component create --file ./mslearn-aml-cli/Allfiles/Labs/05/summary-stats.yml
az ml component create --file ./mslearn-aml-cli/Allfiles/Labs/05/fix-missing-data.yml
az ml component create --file ./mslearn-aml-cli/Allfiles/Labs/05/normalize-data.yml
az ml component create --file ./mslearn-aml-cli/Allfiles/Labs/05/train-decision-tree.yml
az ml component create --file ./mslearn-aml-cli/Allfiles/Labs/05/train-logistic-regression.yml
``` 
Obtain a list of all existing components:
```azurecli
az ml component list -g <resorce-group> -w <workspace-name>
```

### Create a new pipeline with the Designer

### Clean up resources

```azurecli
az ml compute stop --name "testdev-vm" --no-wait
az ml workspace delete
```

Or better yet, delete the entire resorce group:
```azurecli

```
