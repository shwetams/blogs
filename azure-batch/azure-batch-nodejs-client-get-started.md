# Get started with Azure Batch Nodejs client

Through this article we will run through the steps to setting up an Azure batch job, deploy tasks and monitor them using the [Nodejs SDK](http://azure.github.io/azure-sdk-for-node/azure-batch/latest/). This article assumes that you have a base understanding of Nodejs and have Nodejs setup and installed on your dev machine.

You can install azure-batch SDK for node through npm:

`npm install azure-batch`

This will install the latest version of azure-batch node SDK.

Now, let's understand the batch scenario we want to work with in more detail and we get into mapping it into Azure batch components right after that.

## The scenario
I was working with a customer, helping them process large amount of csv files into JSON. I have a csv to JSON processor Python console app that takes in the storage account details, container name and a blob pattern. It iterates through the blobs in the container that match the pattern downloads them and converts them into JSON, and re-uploads them with a /json pattern. Following figure explains the flow of the processor.

![processcsv.py flow](./media/nodejs-client/processcsvflow.png)

Now, I needed to scale this processor to be able to process a large number of files on a daily basis. The files would be uploaded every four hours or so.

Clearly, Azure batch was a pretty good fit.

However, I also needed a trigger that would deploy this batch job, and after completion of all tasks delete the pools.

I was thinking of using a Nodejs blob trigger function. That could deploy a batch job with the python console app and pass the relevant parameters.

## Azure Batch with Nodejs client

If you haven't gone through the overview of Azure Batch service before, I would recommend you read this [detailed overview](service-fabric-get-started.md) document before proceeding further.

I also know some of you will still skip it :), so I have tried to cover the basics of Azure Batch along with the instructions to create it.


### Step 1: Create an Azure Batch Account

As a first step, let's create an Azure Batch account. You can create it from the [portal](batch-account-create-portal.md) or from commandline ([Powershell](batch-powershell-cmdlets-get-started.md) /[Azure cli](batch-cli-get-started.md)).

Following are the commands to create one through Powershell.

Create a Resource Group, skip this step if you already have one where you want to create the Batch Account:

`New-AzureRmResourceGroup -Name <your resource group name> -Location <location such as westus>`

Then create an Azure Batch account using the New-AzureRmBatchAccount command.

`New-AzureRmBatchAccount -AccountName '<unique name>' -Location '<location>' -ResourceGroupName '<resource group name>'`

Each Batch account has its corresponding access keys, these keys are needed to create further resources in Azure batch account. A good practice for production environment is to use Azure Key Vault to store these keys, and create a Service principal for the application that can access and download the keys from vault.

`$acc_keys = Get-AzureRmBatchAccountKeys -AccountName 'account name'`

You can print the account keys from the variable `$acc_keys` by using the following command. Please copy and store the key for further use below.

`$acc_keys.PrimaryAccountKey`

### Azure Batch Architecture

Now that we have an Azure Batch account, next step will be to setup Batch pools , jobs and tasks. We want to do this programmatically each time files are uploaded into the storage container.

Hence, I created a Azure function app and created a Azure Blob Trigger function. Please refer to the links below on details of how to do this. We will straight jump to the code.

- [Create function app](functions-create-first-azure-function)
- [Create Storage Blob trigger function](functions-bindings-storage-blob.md#storage-blob-trigger)

![Azure Batch Architecture](./media/nodejs-client/azurebatcharchitecture.png)

Also, you can go to "Kudu Console" in the Azure function's Settings tab to run the npm install commands. In this case to install Azure Batch SDK for Nodejs.

### Create Azure Batch Pool

Following code snippet shows creation of Azure Batch pool of VMs. I am creating multiple pools, each for a specific customer.

>[AZURE.NOTE] Technically, I will be creating several Azure Storage Blob trigger functions for each customer. Each function will monitor the corresponding Storage account container.


    var batch = require('azure-batch');
    var accountName = 'your account name';
    var accountKey = 'account key downloaded';
    var customerDetails = {
    "customerid":"customerid1",
    "numVMs":4   // Number of VM nodes to create in a pool
    "vmSize":"STANDARD_F4" // VM size nodes in a pool.
    "folders":["folder1","folder2","folder3"...],
    "storage_acc_key": "####",
    "storage_acc_name": "####"

    }

    // Create the credentials object using the account name and key
    var credentials = new batch.SharedKeyCredentials(accountName,accountKey);

    // Create the Azure Batch client
    var batch_client = new batch.ServiceClient(credentials,'azure batch URI');

    // Creating pool ID
    var poolid = "pool" + customerDetails.customerid;

    // Creating Image reference configuration

    var imgRef = {publisher:"Canonical","offer":"UbuntuServer",sku:"14.04.2-LTS",version:"latest"}

    // Creating the VM configuration
    var vmconfig = {imageReference:imgRef,nodeAgentSKUId:"batch.node.ubuntu 14.04"}

    var vmSize = customerDetails.vmSize
    var numVMs = customerDetails.numVMs

    // Creating the Pool configuration

    var poolConfig = {id:poolid, displayName:poolid,vmSize:vmSize,virtualMachineConfiguration:vmconfig,targetDedicated:numVms,enableAutoScale:false }

    // Creating the Pool for the specific customer

    var pool = batch_client.pool.add(poolConfig,function(error,result){
           console.log(result);
           console.log(error);

       });

We are using Linux VMs, you can get the complete list of VM image options from this [link](batch-linux-nodes.md#list-of-virtual-machine-images).

The Azure Batch URI can be found in the Overview tab of the Azure portal. It will be of the format:

https://accountname.location.batch.azure.com

Please refer to the screenshot below:

![Azure batch uri](./media/nodejs-client/azurebatchuri.PNG)

### Create Azure Batch Job

Once you get a success message, next step is to create a Job. A job is a logical grouping of similar tasks. The example I have taken here, a job could be "Process CSV Files" and each task could be for each customer, that is running the same process for each customer.

Since, I have implemented a function app for every customer, for me a job could have one task corresponding to a customer; I had several folders within the container which I wanted to process , so the code below creates multiple tasks for a job, each task corresponds to look for specific folder pattern that I pass as a parameter to the processcsv.py app.

The idea is to maximize the node utilization in the pool and use Batch's parallelism to execute operations in parallel. This completely depends on the particular use cases.


>[AZURE.NOTE] Storage container is a flat structure, when we say a folder within a container we mean a pattern within a blob name.
For example, let's say I have a file on my local computer:

#### folder1\folder2\filename.txt

My blob storage, the container name will be **folder 1** and the blob name will be **folder2\filename.txt**.

In order to retrieve all files in folder2; I will use the Pattern parameter in the storage SDK. In the example above, pattern will be *folder2*

#### Preparation task

The VM nodes created will be blank Ubuntu nodes, you typically will have your own set of programs that you will need to install. I have created a shell script that installs the latest version of Python and also the Azure Storage SDK for python, along with Azure Python SDK. You can refer the files on github using the links provided at the end of the article.

Following code explains adding a preparation task to a job while creating a job. This task will be run on all the VM nodes created in the pool.


    // Creating a job preparation configuration object:
    // id: is a unique ID for the preparation task
    // commandLine is the command line to execute the app , in this case the shell script
    // resourceFiles: It is an array of objects which provide details of files that need to be downloaded for the task.
        //   blobSource: is the SAS URI
        //   filePath: path to save the filePath
        //   'fileMode':File mode in octal format, only applicable for
             Linux node, default value is 0770
    // waitForSuccess: Set it to true, as false would mean that the tasks can run even if the preparation tasks fails. In our case it is needed to be true.
    // runElevated: Elevated privileges are needed to run the task.

    var job_prep_task_config = {id:"storedatahbasesg",commandLine:"sudo sh startup_prereq.sh > startup.log",resourceFiles:[{'blobSource':'Blob SAS URI','filePath':'startup_prereq.sh'}],waitForSuccess:true,runElevated:true}

     // Setting up Batch pool configuration
        var pool_config = {poolId:poolid}

    // Setting up Job configuration along with preparation task
        var job_config = {id:poolid,displayName:"process csv files",jobPreparationTask:job_prep_task_config,poolInfo:pool_config}

    // Adding Azure batch job to the pool
        var job = batch_client.job.add(job_config,function(error,result){
        console.log(error);
        console.log(result);
        });       

### Creating Azure Batch Tasks for a Job

Now that my job is created along with my preparation task, I will create tasks for that job. In the example we are using, I have multiple sub-folders for a customer. I am going to create multiple task, each corresponding to a folder.

I have accordingly modified the processcsv.py file to accept parameters including the storage keys, folder name(s), year, month, day and hour etc.

>[AZURE.NOTE] It is unsafe to send storage account keys, please do use Azure key vault instead. This code is for demo purposes only

Following is the code



    var folders = customerDetails.folders;
    for(var f=0;f<folders.length;f++)
       {
           var fString = '["' + folders[f] + '"]';

           var date_param = Math.floor((new Date()).getUTCMilliseconds() / 1000)
           var exec_info_config = {startTime: date_param}

           // Setting up the task configuration
           // id: unique ID for the task
           // displayName : User friendly display name
           // commandLine: Command line to execute the task
           // resourceFiles: explained above in the job configuration

           var task_config = {id:memberid+"_"+folders[f] + 'processcsv','displayName':'process csv ' + folders[0],commandLine:'python processcsv.py --year ' + year + ' --month ' + month +' --day ' + day + ' --hour '+ hour + ' --memberObj \'[{"id":"' + customerid +'","storage_acc_name":"' + customerDetails.storage_acc_name +'","storage_acc_key": "' + customerDetails.storage_acc_key +'","root_path": "incremental","folders":' + fString +'}]\'',resourceFiles:[{'blobSource':'blob SAS URI','filePath':'processcsv.py'}]}

           // Adding task to the pool

           var task = batch_client.task.add(poolid,task_config,function(error,result){
               console.log(error);
               console.log(result);

           });            
       }        

The code above will add multiple tasks to the pool. And each of the tasks will be executed on a node in the pool of VMs created. If the number of tasks exceeds the number of VMs in a pool, the tasks will wait till a node is made available. This orchestration is handled by Azure Batch automatically.

The portal has detailed views on the tasks and job statuses. You can also use the list and get functions in the Azure Node SDK. Details are provided in the documentation [link](http://azure.github.io/azure-sdk-for-node/azure-batch/latest/Job.html).

#### Github reference
You can find the reference files processcsv.py and the preparation shell script at:

- processcsv.py
- Preparation shell script
