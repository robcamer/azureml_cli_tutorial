# Tutorial
## Create a new workspace associated with a pre-existing resource group
```console
az ml workspace create -w <workspace name> -g <resource group>
```

## Attach current working directory to that workspace\
```console
az ml folder attach -w marinecorpsamlws -g marinecorps
```
## Create a low priority AzureML compute target 
```console
az ml computetarget create amlcompute ^
--max-nodes 1 ^
--name <desired name of compute target> ^
--vm-size STANDARD_D2_V2 ^
--resource-group <name of your resource group> ^
--vm-priority lowpriority ^
--workspace <name of your workspace>
```
After running this command, you should see the following output:
```console
Provisionig compute resources...
Resource creation submitted successfully
```

You can check that it has been created by running the following ocmmand:
```console
az ml computetarget list -w <workspace name> -g <resource group name>
```

## Create a python script run
First, we will need to write a script that we want to execute on the remote compute target that we just created.
You can write your own script using vscode or you IDE of choice and save it at the root directory from which you are running your CLI commands.
You can also just use the [train.py](train.py) script included in this repo.

When you attached the current working directory to a workspace, a .azureml folder should have been autogenerated. 
In this folder, you should find a [conda_dependencies.yml](.azureml/conda_dependencies.yml) file. You can add dependencies to this as needed.
In this folder, you should also find two .runconfig files. These should serve you as examples for creating your own .runconfig file that specifies the remote compute target that you created earlier. Feel free to use the [aml_compute.runconfig](.azureml/aml_compute.runconfig) file in this repo. 

Now, run the command in the CLI to kick off the script run on your remote compute target
```console 
az ml run submit-script -c <name of your .runconfig file*> -e <name of experiment*> <path to the script that you want to run>
```
*Note: if you provide an experiment name that does not already exist in the workspace, a new experiment will be created
*Note: when passing the .runconfig parameter, do not include the file extension. For example, if using the runconfig file provided, your CLI command would look like this:
```console
az ml run submit-script -c aml_compute -e ...
``` 

## Navigate to the portal to see your experiment running
You can now see your experiment running in ms.portal.azure.com under your workspace
Once the run is complete, you should see a model.pkl file in the outputs folder (if you used the provided train.py script)

## Register your model 
Register the model from the script run in the model registry for your workspace
```console
az ml model register --name <name_you_want_to_give_model> --experiment-name <name of your experiment> --run-id <run_id_from_portal> --asset-path <path to your model in the experiment*>
```
*Note: in this case, --asset-path should be outputs/model.pkl (if you used the train.py script provided)

## Check to see if the model was registered correctly
You can check in the portal under the "Models" tab in your workspace on by using the following CLI command:
```console 
az ml model list
```

## Deploy the registered model
Before you can deploy the model as a webservice, you need to create:
* A scoring script:
    * This script must have an init() function that loads the model and a run() function that takes in json data as a parameter and returns the prediction in json format.
    * You can create your own scoring script or use the [score.py](score.py) file provided in this repo. 
* An inferenceconfig.json file:
    * That specifies the name of your scoring script and any dependencies
    * You can create your own inferenceconfig.json file or use the [inferenceconfig.json](.azureml/inferenceconfig.json) file provided in this repo.
* A deploymentconfig.json file:
    * That specifies the metadata of your deployment
    * You can create your own deploymentconfig.json file or use the [deploymentconfig.json](.azureml/deploymentconfig.json) file provided in this repo.
* An AKS cluster to use for deployment:
```console
az ml computetarget create aks ^
--name <name for your aks cluster> ^
--location <location you want it to reside in> ^
--resource-group <name of your resource group> ^
--workspace-name <the workspace you are working in>
```

Once you have all of these things, run this CLI command:
```console
az ml model deploy  ^
--name <name that you want to give your webservice> ^
--model <name of registered model>:<model version number> ^
--inference-config-file <path to inferenceconfig.json> ^
--deploy-config-file <path to deploymentconfig.json> ^
--computetarget <name of your aks cluster>
```

The output from this should look like this:

<img src="./media/image.png" width="500" height="200"/>

Once you have your scoring uri, you can validate your deployed service via a POST request
* To get keys for sendings requests, run the following command:
    ```console
    az ml service get-keys --name <name of service>
    ```
* Use the following data for a test (if you used the iris data set):
    ```json 
    {"data": [[5.1, 3.5, 1.4, 0.2],
              [4.9, 3.0, 1.4, 0.2],
              [4.7, 3.2, 1.3, 0.2], 
              [6.5, 3.0, 5.2, 2.0],
              [6.2, 3.4, 5.4, 2.3]]
    }
    ```
    And expect to get the following output:
    ```number
    [
        0,
        0,
        0,
        2,
        2
    ]
    ```

You can also check the status of your deployed model with the following CLI command:
```console
az ml service show --name <name of service>
```

