# Cortana Intelligence Suite Product Detection from Images Solution

## Table of Contents
- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Architecture](#architecture)
- [Set Up Resources](#set-up-resources)
   - [Create an Azure Resource Group](#arg)
   - [Azure Storage](#storage)
   - [Create Azure DocumentDB](#docdb)
   - [Provision the Microsoft Data Science Virtual Machine](#dsvm)
   - [Create Web App](#webapp)
- [Manage Historic Data](#manage-historic-data)
- [Train a Model on DSVM](#train-a-model-on-dsvm)
   - [Configure the DSVM](#configure-the-dsvm)
   - [Download Data](#download-data)
   - [Train the Model](#train-the-model)
   - [Save the Model](#save-the-model)
- [Deploy a Web Service](#deploy-a-web-service)
- [Monitor Model Performance](#monitor-model-performance)
   - [PowerBI Desktop](#powerbi-desktop)
   - [Publish the Report](#publish-the-report)
- [Retrain Models](#retrain-models)
- [Additional Resources](#additional-resources)


## Introduction

The objective of this Guide is to demonstrate data pipelines for retailers to detect products from images. Retailers can use the detections to detect which product a customer has picked up from the shelf. This information can also help stores manage product stockings. This tutorial shows how to set up the prediction service as well as retrain models as new data become available.

The end-to-end solution is implemented in the cloud using Microsoft Azure. The flow in this techincal guide is organized to reflect what we expect customers will do to develop their own solutions. Thus this deployment guide will walk you through the following steps:

- Create Azure Resource Group and add DocumentDB, Azure Storage Blob, Data Science Virtual Machine, Web App, Application Insights to the resource group
- Install Python and related packages to copy data from on-prem to DocumentDB and Azure Storage Blob
- Log onto the Data Science Virtual Machine to download data, train a model, save the model and performance information, and deploy a web service
- Monitor model performace from PowerBI
- Consume the web service

[Return to Top](#cortana-intelligence-suite-product-detection-from-images-solution)

## Prerequisites

The steps described in this guide require the following prerequites:

- **An Azure subscription**: To obtain one, see [Get Azure free trial](https://azure.microsoft.com/en-us/free/)
- A **[Microsoft Power BI](https://powerbi.microsoft.com/en-us/)** account
- An installed copy of **[Power BI Desktop](https://powerbi.microsoft.com/en-us/desktop/?gated=0&number=0)**

[Return to Top](#cortana-intelligence-suite-product-detection-from-images-solution)

## Architecture

[Figure 1][pic1] illustrates the Azure architecture for this solution.

[![Figure 1][pic1]][pic1] Figure 1

This figure describes two workflows - the numbers in orange circles are for the model training workflow and those in pink are for the consumption workflow. In the model training workflow, the historical data - including boths images and annotations - will be loaded from you local computer onto DocumentDB and Azure Storage Blob, with images being stored at Azure Storage Blob and Annotation information at DocumentDB. The model training process is completed on Data Science Virtual Machine (DSVM) after downloading data form DocumentDB and Azure Storage Blob onto the DSVM. The model will be saved on Azure Storage Blob and performance on DocumentDB. Then the model can be deployed as a web service. New data can be annotated and used for model retraining.

In the consumption workflow, the test-website sends images to the web service which identifies products, saves the images and scores, and returns the scored image back to the users. Application Insights will be used to monitor the web service.

[Return to Top](#cortana-intelligence-suite-product-detection-from-images-solution)

## Set Up Resources

The first thing we'll do is to create an Azure Resource Group and add to it DocumentDB, Azure Storage Blob, DSVM, and Web App.

### Choose a Unique String

You will need a unique string to identify your deployment because some Azure services (e.g. Azure Storage)  require a unique name for each instance across the service. We suggest you use only letters and numbers in this string and the length should not be greater than 9.

We suggest you use "[UI]image[N]"  where [UI] is the user's initials and [N] is a random integer that you choose. Characters must be entered in lowercase. Please create a memo file and write down "unique:[unique]" with "[unique]" replaced with your actual unique string.

<a name="arg"></a>
### Create an Azure Resource Group

1. Log into the [Azure Management Portal](https://ms.portal.azure.com).
2. Click **Resource groups** button on upper left (hover over the buttons to view their names or ), then click **+Add** button to add a resource group.
3. Enter your **unique string** for the resource group and choose your subscription.
4. For **Resource Group Location**, you can choose one that's closest to you.

Please save the information to your memo file in the form of the following table. Replace the content in [] with its actual value.

| **Azure Resource Group** |                     |
|------------------------|---------------------|
| resource group name    |[unique]|
| region              |[region]||


In this tutorial, all resources will be generated in the resource group you just created. You can easily access these resources from the resource group overview page, which can be accessed as follows:

1. Log into the [Azure Management Portal](https://ms.portal.azure.com).
2. Click the **Resource groups** button on the upper-left of the screen.
3. Choose the subscription your resource group resides in.
4. Search for (or directly select) your resource group in the list of resource groups.

<a name="storage"></a>
### Create an Azure Storage Account

In this step as well as all remaining steps, if any entry or item is not mentioned in the instruction, please leave it as the default value.

We'll call Azure Storage Blob and Blob interchangeablly.

1. Go to the [Azure Portal](https://ms.portal.azure.com) and navigate to the resource group you just created.
2. In ***Overview*** panel, click **+Add** to add a new resource. Enter **Storage Account** and hit "Enter" key to search.
3. Click on **Storage account - blob, file, table, queue** offered by Microsoft (in the "Storage" category).
4. Click **Create** at the bottom of the description panel.
5. Enter your **unique string** for "Name."
6. Make sure the selected resource group is the one you just created. If not, choose the resource group you created for this solution.
7. Leaving the default values for all other fields, click the **Create** button at the bottom.
8. Go back to your resource group overview and wait until the storage account is deployed. To check the deployment status, click on the Notifications icon on the top right corner as shown in the following figure.

[![Figure 2][pic2]][pic2]

Ater a successful deployment you should see a figure like below.

[![Figure 3][pic3]][pic3]

Click on "Refresh" from the resource group's **Overview** pane so that the new resource shows up. Get the primary key for the Azure Storage Account which will be used in the Python scripts to upload and download data by following these steps:

1. Click the created storage account. In the new panel, click on **Access keys**.
1. In the new panel, click the "Click to copy" icon next to `key1`, and paste the key into your memo.

| **Azure Storage Account** |                     |
|------------------------|---------------------|
| Storage Account        |[unique string]|
| Access Key     |[key]             ||

<a name="docdb"></a>
### Create Azure DocumentDB

1. Go to the [Azure Portal](https://ms.portal.azure.com) and navigate to the resource group you just created.
2. In ***Overview*** panel, click **+Add** to add a new resource. Enter **documentdb** and hit "Enter" key to search.
3. Click on **NoSQL (DocumentDB)** offered by Microsoft (in the "Storage" category).
4. Click **Create** at the bottom of the description panel.
5. Enter your **unique string** for "ID."
6. Make sure **DocumentDB** is selected for "NoSQL API."
7. Make sure the selected resource group is the one you just created. If not, choose the resource group you created for this solution.
8. Leaving the default values for all other fields, click the **Create** button at the bottom.
9. Go back to your resource group overview and wait until DocumentDB is deployed as reported in the Notifications area.

Get the primary key for the DocumentDB account which will be used in the Python scripts to upload and download data by following these steps:

1. Click the created DocumentDB. In the new panel, click on **keys.**
1. In the new panel, click the "Click to copy" icon next to `URI` and paste the key into your memo. Repeat the same process for `PRIMARY KEY.`

| **DocumentDB Account** |                     |
|------------------------|---------------------|
| URI        |[uri]|
| Access Key     |[key]             ||

<a name="dsvm"></a>
### Provision the Microsoft Data Science Virtual Machine

1. Go to the [Azure Portal](https://ms.portal.azure.com) and navigate to the resource group you just created.
2. In ***Overview*** panel, click **+Add** to add a new resource. Enter **Data Science Virtual Machine** and hit "Enter" key to search.
3. Click on **Data Science Virtual Machine** offered by Microsoft (in the "Compute" category).
4. Click **Create** at the bottom of the description panel.
5. Enter your **unique string** for "Name."
6. Enter a user name and save it to your memo file.
7. Enter your password and confirm it. Then save it to your memo file.
8. Make sure the selected resource group is the one you just created. If not, choose the resource group you created for this solution.
9. Leaving the default values for all other fields, click the **OK** button at the bottom.
10. On the ***Size*** panel that shows up, choose a size and click the **Select** button.
11. On the ***Settings*** panel that shows up, use the default values for all fields and click the **OK** button.
12. On the ***Summary*** panel that shows up, double check it and click the **OK** button.
13. On the ***Buy*** panel that shows up, click the **Purchase** button. You can ignore the warning message `The highlighted Marketplace purchase(s) are not covered by your Azure credits, and will be billed separately.`
14. Go back to your resource group overview and wait until DSVM is deployed as reported in the Notifications area.

Save your credentials to the memo file.

| **DSVM** |                     |
|------------------------|---------------------|
| username        |[username]|
| password     |[password]  ||

Once the VM is created, you can remote desktop into it using the account credentials that you provided. To get access link, follow these steps:

1. From the Resource Group click the created DSVM (type "Virtual machine").
2. In ***Overview*** panel, click on the **Connect** button at the top left corner to download the ".rdp" file. We'll use this file later.

<a name="webapp"></a>
### Create Web App

1. Go to the [Azure Portal](https://ms.portal.azure.com) and navigate to the resource group you just created.
2. In ***Overview*** panel, click **+Add** to add a new resource. Enter **Web App** and hit "Enter" key to search.
3. Click on **Web App** offered by Microsoft (in the "Web + Mobile" category).
4. Click **Create** at the bottom of the description panel.
5. Enter your **unique string** for "App name."
6. Make sure the selected resource group is the one you just created. If not, choose the resource group you created for this solution.
7. Click on "App Service plan/Location" and then **+Create New**.
8. Enter your **unique string** for "App Service plan."
9. Click on "Pricing tier" and choose "S1 Standard", then click on **Select.**
10. Click **OK** to confirm the App Service plan.
11. Click **On** for "Application Insights."
12. Leaving the default values for all other fields, click the **Create** button at the bottom.
13. Go back to your resource group overview and wait until the App Service is deployed as reported in the Notifications area. Click **Refresh** in the **Overview** pane. 
14. Click on the newly deployed App Service (Type "App Service") and you'll see the "Overview" page.
15. In the left panel click on **Application settings** (in **SETTINGS** group) then selecct "64-bit" for "Platform" and "On" for "Always On." Save the settings by clicking on **Save**.
16. In the left panel click on **Extensions** (in **DEVELOPMENT TOOLS** group) then click **+Add**. In the window that shows up select **Choose Extension** then **Python 3.5.3 x64**. 
17. Click **OK** to accept legal terms and **OK** again to install the web app extension. Make sure the extension is installed successfully as reported in the "Notifications" area.
18. In  the left panel click on **Deployment options** (in **DEPLOYMENT** group) then **Choose Source** followed by **Local Git Repository**. Leave **Performance Test** as "Not Configured" and click on **OK** to set up deployment source.
19. If this is the first time you deploy application to Azure, you'll need to set your deployment credentials. To do this, click **Deployment credentials** (in **DEPLOYMENT** group) and select your username and password.
20. Click **Overview** from the left panel and locate the value for "URL", hover over it and wait for the **Click to copy** button to appear. Click on it to copy the value. Paste it to your memo file.
21. Click **Overview** from the left panel and locate the value for "Git clone url", hover over it and wait for the **Click to copy** button to appear. Click on it to copy the value. Paste it to your memo file.

Save your credentials to the memo file.

| **Azure App Service** |                     |
|------------------------|---------------------|
| username        |[username]
| password     |[password]  |
| URL                     |[URL]  |
| Git clone url     |[Git clone url]  |

Now that the web app has been prepared, we'll deploy a web service to it after training a model.

[Return to Top](#cortana-intelligence-suite-product-detection-from-images-solution)

## Manage Historic Data

As described in the architecture, we assume that there are some data on you local machine and we need to save those images onto DocumentDB and Azure Storage Blob. The demo data are saved in the folder /data_management. To download the content of this folder, you can use either of these two approaches:

- Download the entir repository by clicking on the **Clone or download** button of the GitHub repository and then **Download ZIP**. this approach downloads the entire repositor and takes longer. Unzip the downloaded file.
- Open the [DownGit](https://minhaskamal.github.io/DownGit/#/home) site and enter `https://github.com/Azure/cortana-intelligence-product-detection-from-images/tree/master/technical_deployment/data_management` then click on **Download** to download the folder. This approach just downloads the data we need and is faster. Unzip the downloaded file.


Open the "technical_deployment/data_management/config.py" file from a text editor and provide values for the following fields using the informaiton from your memo file and leave the other fields as is:

- `blob_account_name`
- `blob_account_key`
- `documentdb_host`
- `documentdb_key`

If you already have Python installed, you can keep using your version. Otherwise, download Python 3.5.3 from [Python Software Foundation](https://www.python.org/downloads/) and install it. Then go to the command window and type the following:

````bash
cd <path-to-data_management-folder>
pip install -r requirements.txt
python 1_upload_historical_data.py
````
After successfulling running the above code, you can verify that the data has been uploaded successfully. To check the contents in Azure Storage Blob, go to the [Azure Portal](https://ms.portal.azure.com) and navigate to the resource group you just created. Then click on the Azure Storage Blob you created (type "Storage account") and click on "Blobs" in the **Overview** panel. Two containenumpy-1.12.1+mkl-cp35-cp35m-win_amd64rs have been created: *images* and *models*. Click on *images* to view the images that have been uploaded.

To check the contents in DocumentDB, navigate to the resource group you just created. Then click on the DocumentDB you created (type "NoSQL (DocumentDB) account") and click on "Document Explorer" in the left panel. Then select "image_collection" from the drop-down menu in the right panel and you will see the documents that have been created. Click on any ID to check the contents of that document. In each document the attribute "azureBlobUrl" points to the corresponding image saved on the Blob.

[Return to Top](#cortana-intelligence-suite-product-detection-from-images-solution)

## Train a Model on DSVM

### Configure the DSVM

Double click the ".rdp" file that you downloaded in step [Provision the Microsoft Data Science Virtual Machine](#dsvm). Enter your credentials for the DSVM to log on to it. From this point forward we'll do everything on DSVM.

Open a web browser on the DSVM and open the [GitHub repo](https://github.com/Azure/cortana-intelligence-product-detection-from-images). Download the repository by clicking on the **Clone or download** button of the GitHub repository and then **Download ZIP**. Unzip the downloaded file.

Go to the [Unofficial Windows Binaries for Python Extension Packages site](http://www.lfd.uci.edu/~gohlke/pythonlibs/) to download the following python-wheel and save it in the folder "technical_deployment/train_model/resources/python35_64bit_requirements."
```
numpy-1.12.1+mkl-cp35-cp35m-win_amd64
```

Locate the folder for "technical_deployment/train_model/resources/python35_64bit_requirements" , then open a command window and type the following:

````bash
cd <path-to-resources/python35_64bit_requirements>
conda create -n cntk-py35 python=3.5
activate cntk-py35
pip install -r requirements.txt
````

Running these commands successfully will allow us to create a virtual environment and activate it. They also install the packages we will need for the rest of the tutorial.

### Download Data

In this step we'll download image from Blob and labels from DocumentDB. Just like on the local computer, we open the "config.py" script in "technical_deployment/data_management" folder from a text editor and provide values for the following fields using the informaiton from your memo file:

- `blob_account_name`
- `blob_account_key`
- `documentdb_host`
- `documentdb_key`

In addition, use today's date as value for `model_version` in the format of `yyyymmdd`. Leave the other fields as is.

Check the folder "technical_deployment/train_model/data/grocery" and confirm that it is empty. Open a command window and type the following to download data.

````bash
cd <path-to-data_management-folder>
activate cntk-py35
python 2_download_data_train.py
````

You can verify that the data have been downloaded successfully by exploring the folders in "technical_deployment/train_model/data/grocery"

### Train the Model

Download the file *AlexNet.model* from [here](https://www.cntk.ai/Models/AlexNet/AlexNet.model) and place it into the subfolder *technical_deployment/train_model/resources/cntk*.

Open a command window and type the following to train the model. It takes about 1 minutes to run "4_trainSVM.py" and the other scripts completes within seconds.

You can learn more about the models from the detailed descriptions in the "train_model" folder's README file. Notice however the prerequisites as described there are not relevant here.


````bash
cd <path-to-train_model-folder>
activate cntk-py35
python 1_computeRois.py
python 2_cntkGenerateInputs.py
python 3_runCntk.py
python 4_trainSvm.py
python 5_evaluateResults.py
python 5_visualizeResults.py
````

### Save the Model

Once you're satisfied with the trained model, you can save its parameters to Blob for future reference. You can also save the performance metrics to DocumentDB. To do these, open a command window and run the following commands.

```bash
cd <path-to-technical_deployment\data_management-folder>
activate cntk-py35
python 3_upload_model.py
python 4_upload_performance.py
```

After you run these commands successfully, the model parameters will be saved in a folder named after your model_version in the **models** container of your Blob account and the performance metrics will be saved in the collection **performance_collection** within the database **detection_db** of your DocumentDB account.

[Return to Top](#cortana-intelligence-suite-product-detection-from-images-solution)

## Deploy a Web Service

1. Go to the [CNTK release page](https://github.com/Microsoft/CNTK/releases) and download "CNTK for Windows v.2.0 Beta 11 CPU only", rename this to "cntk.zip" and put this in the "technical_deployment/web_service" foler.
2. Go to the [Unofficial Windows Binaries for Python Extension Packages site](http://www.lfd.uci.edu/~gohlke/pythonlibs/) to download the following python-wheel and save it in the folder "technical_deployment/web_service/Wheels."
```
numpy-1.12.1+mkl-cp35-cp35m-win_amd64
```
3. Open a command window and type the following to copy files.
```bash
cd <path-to-data_management-folder>
activate cntk-py35
python 5_copy_web_service_data.py
```
4. Now go to folder "technical_deployment/web_service" and deploy the web service by running the following commands. Make sure to replace the following fields with your own values: "your.email@address.com" with your email,"your name" with your name, and "your Git url" with [Git clone url] that you saved in your memo file. Enter your username and password for the Web App when prompted. This process takes about 3 minutes for uploading data and 10 minutes for deploying web service.
```bash
cd <path-to-web_service-folder>
git init
git config --global user.email "your.email@address.com"
git oonfig --global user.name "your name"
git add -A
git commit -m "Initialize web service"
git remote add azure "your Git url"
git push azure master
```
5. Once the deployment is successful, you can open the URL (not the Git clone url) that you saved in your memo file. Upload an image for scoring from the "technical_deployment/data_management/score_images" folder. You should get a scored image within a minute.

[Return to Top](#cortana-intelligence-suite-product-detection-from-images-solution)

## Monitor Model Performance

### PowerBI Desktop

Power BI is used to monitor model performance and provide intelligence on the units for which the images are taken. The instructions that follow describe how you can use the provided Power BI desktop file (image-detection-monitoring.pbix) to visualize your data.

1. If you have not already done so, download and install the [Power BI Desktop application](https://powerbi.microsoft.com/en-us/desktop).
1. Download the Power BI template file `image-detection-monitoring.pbix`, which is available in the  GitHub repository's [power_bi folder](https://github.com/Azure/cortana-intelligence-product-detection-from-images/tree/master/technical_deployment/power_bi), by left-clicking on the file and clicking on "Download" on the page that follows.
1. Double click the downloaded ".pbix" file to open it in Power BI Desktop.
1. The template file connects to a database used in development. You'll need to change some parameters so that it links to your own database. To do this, follow these steps:
    1. Click on "Edit Queries" as shown in the following figure.
       
        [![Figure 4][pic4]][pic4] 

    1. Select a Query from the Queries panel (e.g., average_precision) and click on "Advanced Editors" as shown in the following figure.

        [![Figure 5][pic5]][pic5] 

    1. In the pop-up window for Advanced Editor, replace all "lzimage01" values with your "unique string" (DocumentDB database name). This process is shown in the following two figures, which assumes that the unique string is "flimage01". (You should use the name of your own database.) Click "Done" after making the changes.
    
	Before
        [![Figure 6][pic6]][pic6]
        
	After        
        [![Figure 7][pic7]][pic7] 

    1. With the same Query (e.g., average_precision) selected, click on "Edit Credentials" and enter your your credentials for accessing your database (recorded in the DocumentDB memo table). Then click on "Connect" as shown in the following figures. 

        Click on "Edit Credentials"
        [![Figure 8][pic8]][pic8]
        
        Enter account key and click on "Connect"
        [![Figure 9][pic9]][pic9]

    1. The data for your table should be displayed if the connection information was correct, as in the following figure. It's OK if your values don't match those in the screenshot.

        [![Figure 10][pic10]][pic10]

    1. Update the other Queries by replacing "lzimage01" with the name of your database. This is NOT necessary for queries "roc_combined", "roc_thresh_precision", "roc_thresh_recall", and "roc_combined_vertical."
    1. Click on the "Close & Apply" ribbon after all Queries have been updated.
    
You should now see multiple tabs in Power BI Desktop. The "Precision vs Recall" shows that the performance of the model for the test dataset is good. The "Average Precision" tab shows average precision by class. The "Objects" tab provides summary on objects detected through the web service. So you'll need to use the web service at least once in order to have information on this tab. More information about them can be found in the descriptions in each tab.

### Publish the Report

Now we can publish the report into Power BI online to easily share with others: 
 
1. Click on "Publish" as shown below. Sign in with your Power BI credentials and choose a destination (e.g., My Workspace). 
[![Figure 11][pic11]][pic11]

1. After the report is successfully published, you should see a window like the following. Click on "Got it."
[![Figure 12][pic12]][pic12]

1. Sign into [Power BI](www.powerbi.microsoft.com) and click on the "image-detection-monitoring" report (under Reports) to open it. 
1. We'll share the Churn Rate Overview tab from the report to create a dashboard. To do this, click on the "MyDashboard" tab and select "Pin Live Page" as shown in the following figure. 
[![Figure 13][pic13]][pic13]

1. Pin the page to a new dashboard named "Product Detection Dashboard" as shown in the following figure.
[![Figure 14][pic14]][pic14]

1. Pin the remaining tabs using the same approach.
1. Locate the newly created "Product Detection Dashboard" under Dashboards group. You can share it with others by clicking on the Share button, as shown in the following figure.
[![Figure 15][pic15]][pic15]


[Return to Top](#cortana-intelligence-suite-product-detection-from-images-solution)

## Retrain Models

At this point, you have a working solution that can used to scores images and monitor model performance. As you gather more data, model performance can be improved by retraining the model using the new data. To do that we will following these steps:

- Download the images from DocumentDB and Blob to the DSVM. You can use a specific new image or a group of them. In this demo we'll be downloading all images that have been processed by the web service
- Add manual annotations
- Save the manual annotations
- Retrain the model using historic data and the newly annotated images
- Save and deploy the retrained model

The downloaded images will be saved in a folder named "livestream" under the folder "technical_deployment/train_model/data/grocery." This folder will be generated by the script. To download the images, open a command window and run the following commands:

````bash
cd <path-to-data_management-folder>
activate cntk-py35
python 6_annotation_download_data.py
````

Open the folder "technical_deployment/train_model/data/grocery/livestream" to make sure that the images have been downloaded successfully. If you want to view a scored image, you can modify the "visualize_local.py" script under the "technical_deployment\train_model" folder so that "sub_dir" and "file_name" point to the image you want to view. Then run the following commands:

````bash
cd <path-to-train_model-folder>
activate cntk-py35
python A_visualize_local.py
````

Two scripts are provided for making annotations, one for drawing object boundaries and the other for adding labels to the boundaries. Delete from the "technical_deployment/train_model/data/grocery/livestream" folder the existing files that end with ".bboxes.tsv" or ".bboxes.labels.tsv" to allow the use of the annotation scripts.

Open the script "A1_annotateImages.py" and change the value of "imagesToAnnotateDir" to be the "livestream" folder. Do the same for the script "A2_annotateBboxLabels.py."

Run the following commands to draw boundaries, pressing 'n' when you are done with one image, 'u' when you want to undo (i.e. remove) the last rectangle, and 'q' when you want to quit the annotation tool. The script will quit once all images have been annotated.

````bash
cd <path-to-train_model-folder>
activate cntk-py35
python A1_annotateImages.py
````

Continuing in the above command window, run the following command to add labels.

````bash
python A2_annotateBboxLabels.py
````

Once you are satistified with the annotations, save them to DocumentDB by running the following commands.

````bash
cd <path-to-technical_deployment\data_management-folder>
activate cntk-py35
python 7_annotation_update.py
````

To use the additional annotations for training, copy the images (.jpg), the boxes (.bboxes.tsv), and the labels (.bboxes.labels.tsv) from the "livestream" folder to the "positive" folder. Now you can repeat the same process for training models (section [Train the Model](#train-the-model)), saving models (section [Save the Model](#save-the-model)), deploy web service (section [Deploy a Web Service](#deploy-a-web-service)), and monitoring model performance (section [Monitor Model Performance](#monitor-model-performance)).

If you have deleted the DSVM, you can start by configuring the DSVM (section [Configure the DSVM](#configure-the-dsvm)).

If you have the same DSVM but have deleted the data from previous training, you can start by downloading data (section [Download Data](#download-data)).

[Return to Top](#cortana-intelligence-suite-product-detection-from-images-solution)

## Additional Resources

This GitHub repo [ObjectDetectionUsingCntk ](https://github.com/Azure/ObjectDetectionUsingCntk) offers more advanced details of the model used in this solution.

More examples on deploying web services for CNTK based models can be found in this GitHub repo [Azure-WebApp-w-CNTK ](https://github.com/ilkarman/Azure-WebApp-w-CNTK).

[Return to Top](#cortana-intelligence-suite-product-detection-from-images-solution)

[pic1]: https://cloud.githubusercontent.com/assets/9322661/24459697/2caf4612-146a-11e7-97e7-3b628cd7f760.PNG
[pic2]: https://cloud.githubusercontent.com/assets/9322661/24463987/738fb9a2-1476-11e7-8273-17107a76cbd5.png
[pic3]: https://cloud.githubusercontent.com/assets/9322661/24463988/739306f2-1476-11e7-811f-f8debc7a59b9.png
[pic4]: https://cloud.githubusercontent.com/assets/9322661/24659322/58ed5b92-191a-11e7-848c-7ed77042ee61.png
[pic5]: https://cloud.githubusercontent.com/assets/9322661/24659323/58edc5b4-191a-11e7-878d-9c5cd102772e.PNG
[pic6]: https://cloud.githubusercontent.com/assets/9322661/24659330/590c1348-191a-11e7-8703-ea845659a7e6.PNG
[pic7]: https://cloud.githubusercontent.com/assets/9322661/24659325/58ff2e3a-191a-11e7-9d0b-d9f7aa36974c.PNG
[pic8]: https://cloud.githubusercontent.com/assets/9322661/24659321/58e97fae-191a-11e7-857b-428d1839360c.PNG
[pic9]: https://cloud.githubusercontent.com/assets/9322661/24659329/590bdb08-191a-11e7-959e-445b84d79ccb.PNG
[pic10]: https://cloud.githubusercontent.com/assets/9322661/24659324/58fc23b6-191a-11e7-9c34-194394496827.PNG
[pic11]: https://cloud.githubusercontent.com/assets/9322661/24659327/59069346-191a-11e7-9ff3-9486e328ad1c.png
[pic12]: https://cloud.githubusercontent.com/assets/9322661/24659332/591cf866-191a-11e7-8083-e9a365b5bed7.PNG
[pic13]: https://cloud.githubusercontent.com/assets/9322661/24659333/59219e3e-191a-11e7-9c19-f27b3e8872fb.PNG
[pic14]: https://cloud.githubusercontent.com/assets/9322661/24659331/5918502c-191a-11e7-85f3-cade658d4d46.PNG
[pic15]: https://cloud.githubusercontent.com/assets/9322661/24659326/58ffb6ca-191a-11e7-8057-4d14186c8ba7.PNG