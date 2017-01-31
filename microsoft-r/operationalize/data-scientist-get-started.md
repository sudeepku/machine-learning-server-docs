---

# required metadata
title: "Operationalization: Get Started for Data Scientists | Microsoft R Docs"
description: "Operationalization of R Analytics with Microsoft R Server"
keywords: ""
author: "j-martens"
manager: "jhubbard"
ms.date: "1/25/2017"
ms.topic: "get-started-article"
ms.prod: "microsoft-r"
ms.service: ""
ms.assetid: ""

# optional metadata
ROBOTS: ""
audience: ""
ms.devlang: ""
ms.reviewer: ""
ms.suite: ""
ms.tgt_pltfrm: ""
ms.technology: 
  - deployr
  - r-server
ms.custom: ""
---

# Get Started for Data Scientists

Now that you've learned about [R Server's operationalization feature](about.md), we can dig into how data scientists can deploy,  consume, and share web services in order to operationalize their R analytics.

Data scientists work locally with [Microsoft R Client](../r-client-get-started.md) in their preferred R IDE and favorite version control tools to build scripts and models. Using the `mrsdeploy` package that ships with Microsoft R Client and R Server, the data scientist can develop, test, and ultimately deploy these R analytics as web services in your production environment. 

An R Server web service is an R code execution on the [operationalization compute node](configuration-initial.md). Each web service is uniquely defined by a `name` and `version`. You can use the functions in [the `mrsdeploy` package](../mrsdeploy/mrsdeploy.md) to gain access a service's lifecycle from an R script. This package is installed with Microsoft R Client as well as Microsoft R Server.  The `mrsdeploy` package provides functions for publishing and managing a web service backed by an R code block or script that you provide. The package also provides functions for establishing a [remote execution](remote-execution.md) session in a console application.  [Learn more about this package](../mrsdeploy/mrsdeploy.md). Similarly, a set of [RESTful APIs](https://microsoft.github.io/deployr-api-docs/9.0.1/#services-management-apis) are available to provide direct programmatic access to a service's lifecycle directly. 

Once deployed, the web service can be: 
+ Consumed directly in R by another data scientist, for testing purposes for example 
+ [Integrated into an application by an application developer](app-developer-get-started.md)  using the  Swagger-based .JSON file produced when the web service was published. 

![Operationalization Engine](../media/o16n/data-scientist-easy-deploy.png) 

## What you'll need

You'll develop your R analytics locally with R Client, deploy them to Microsoft R Server as web services, and then consume or share them.

**On the local client**, you'll need to [install R Client](../r-client-get-started.md) first.  You'll also need to [configure the R IDE](https://msdn.microsoft.com/en-us/microsoft-r/r-client-get-started#step-2-configure-your-ide) of your choice, such as R Tools for Visual Studio, to run Microsoft R Client.  Once you have this set up, you can develop your R analytics in your local R IDE using the functions in [the `mrsdeploy` package](../mrsdeploy/mrsdeploy.md) that was installed with Microsoft R Client (and R Server). 

**On the remote server**, you'll need the connection details and access to an instance of [Microsoft R Server](../rserver.md) with its [operationalization feature configured](configuration-initial.md). Once R Server is configured for operationalization, you'll be able to connect to it from your local machine, deploy your models and other analytics to Microsoft R Server as web services, and finally consume or share those services. Please contact your administrator for any missing connection details.

## How to deploy a model as a service

This example walks you through the deployment of a simple model as a web service hosted in R Server. 

>[!IMPORTANT]
> This example assumes you have configured an R Integrated Development Environment (IDE) to work with [Microsoft R Client](../r-client-get-started.md). It also assumes you have [authenticated access](security-authentication.md) to an instance of Microsoft R Server with its [operationalization feature configured](configuration-initial.md).


We'll use the following script in our example:

   ```R
   ##             MODEL DEPLOYMENT EXAMPLE                 ##

   ##########################################################
   #       Create & Test a Logistic Regression Model        #
   ##########################################################
    
   # Logistic regression vehicle transmission to estimate 
   # the probability of a vehicle being fitted with a 
   # manual transmission base on horsepower (hp) and weight (wt)

   # Create glm model with `mtcars` dataset
   carsModel <- glm(formula = am ~ hp + wt, data = mtcars, family = binomial)

   # Wrap prediction in a function
   manualTransmission <- function(hp, wt) {
     newdata <- data.frame(hp = hp, wt = wt)
     predict(carsModel, newdata, type = "response")
   }
   
   # test function by printing results
   print(manualTransmission(120, 2.8)) # 0.6418125

   ##########################################################
   #            Log into Microsoft R Server                 #
   ##########################################################
   
   # Authenticate with Azure AD using `mrsdeploy` pkg function
   remoteLoginAAD(
       "https://rserver.contoso.com:12800", 
       authuri = "https://login.windows.net", 
       tenantid = "microsoft.com", 
       clientid = "5599bff3-2ec2-4975-9068-28acf86a3b6f", 
       resource = "b9667d00-1c06-4b9d-a94f-06ecb71822b0", 
       session = FALSE 
   )

   ##########################################################
   #             Publish Model as a Service                 #
   ##########################################################

   # Publish as service using `publishService()` function from 
   # `mrsdeploy` package. Name service `mtService` and provide
   # unique version number. Assign service to the variable `api`
   api <- publishService(
     mtService,
     code = manualTransmission,
     model = carsModel,
     inputs = list(hp = "numeric", wt = "numeric"),
     outputs = list(answer = "numeric"),
     v = "v1.0.0"
   )

   ##########################################################
   #                 Consume Service in R                   #
   ##########################################################
   
   # Print capabilities that define the service holdings (inputs/outputs)
   print(api$getCapabilities())
   
   # Consume service by calling function, `manualTransmission`
   # contained in this service
   result <- api$manualTransmission(120, 2.8)

   # Print response output named `answer`
   print(result$output("answer")) # 0.6418125   

   ##########################################################
   #         Get Service-specific Swagger File in R         #
   ##########################################################
   
   # During this authenticated session, download the  
   # Swagger-based JSON file that defines this service
   swagger <- api$swagger()
   cat(swagger, file = "swagger.json", append = FALSE)

   # Authenticated users can get the Swagger-based JSON file in R
   # by name at a later time using the function `getServices` from 
   # `mrsdeploy` package such as:
   # api <- getServices("mtService", "v1.0.0")
   # swagger <- api$swagger()
   # cat(swagger, file = "swagger.json", append = FALSE)

   # Share Swagger-based JSON with those who need to consume it
   ```

Now let's dive into this example down. Let's start by creating the model locally, then publish it, and then share it with other authenticated users.

### (1) Model locally

1. Launch your R IDE. 

   <!--@In our example, we are using R Tools | Visual Studio (RTVS), but any popular R IDE should work.-->
   
   <!--@SCREEN [RTVS with R Client installed]-->

1. Run the R code to create the model.  

   In our example, the GLM model, `carsModel`, is creating using the dataset `mtcars`, which is a built-in data frame in R. This model estimates the probability of a vehicle being fitted with a manual transmission base on horsepower (hp) and weight (wt)

   ```R
   # Create glm model with `mtcars` dataset
   carsModel <- glm(formula = am ~ hp + wt, data = mtcars, family = binomial)

   # Wrap prediction in a function
   manualTransmission <- function(hp, wt) {
     newdata <- data.frame(hp = hp, wt = wt)
     predict(carsModel, newdata, type = "response")
   }
   
   # test function by printing results
   print(manualTransmission(120, 2.8)) # 0.6418125
   ```

1. Examine the results of the locally executed code. 

   <!--In this example, the results show @.-->
   
   <!--@SCREEN [RTVS SHOWING RESULTS]-->

<br> 

### (2) Publish model as a web service

1. From your local R IDE, log into Microsoft R Server **with your credentials** using the appropriate authentication function from [the `mrsdeploy` package](../mrsdeploy/mrsdeploy.md) (`remoteLogin`, `remoteLoginAAD`, or `remoteLoginAD`).  Ask your administrator for authentication details if you do not have any.

   In our example, we used Azure Active Directory for authentication.

   ```R   
   remoteLoginAAD(
       "https://rserver.contoso.com:12800", 
       authuri = "https://login.windows.net", 
       tenantid = "microsoft.com", 
       clientid = "5599bff3-2ec2-4975-9068-28acf86a3b6f", 
       resource = "b9667d00-1c06-4b9d-a94f-06ecb71822b0", 
       session = FALSE 
   )
   ``` 

   <!--@SCREEN [RTVS PROOF OF SUCCESSFUL CONNECTION TO MRS.]-->

   Now, you are successfully connected to the remote R Server.

1. Publish the model as a web service to R Server using the `publishService()` function from the `mrsdeploy` package.

   To publish it, you'll need to specify:
   + A name for the service (required)
   + The code (required) and necessary model assets 
   + Any inputs 
   + The output that application developers will need to integrate in their applications. 
   + A unique version number for the service, if you'll be sharing it with others.   

   <!--@ SHOULD WE COVER VERSIONING AT THIS POINT? EACH TIME YOU TRY THIS CODE, USE A UNIQUE VERSION NUMBER. -->

   <!-- @ DO WE ADD A WARNING NOTE HERE SAYING THAT YOU CANNOT PUBLISH FROM A REMOTE SESSION AND POINT TO THAT REMOTE EXECUTION DOCUMENTATION HERE?-->
   <br>

   Learn more about managing and publishing web services using this package [in this article](../mrsdeploy/mrsdeploy-websrv-vignette.md).

   In our example, we executed these commands to publish a web service called `mtService` using the model called `carsModel` and a function called `manualTransmission`. As an input, it takes a list of vehicle horsepower and vehicle weight represented as R numerica. As an output, a percentage as an R numeric for the probability each vehicle has of being fitted with a manual transmission. We then download the Swagger-based JSON file that was generated when the service was published to our local file system.

   `getCapabilities()` will return the service name, version, descriptions, inputs, outputs, and the name of the function to be consumed.

   ```R
   # Print capabilities that define the service holdings (inputs/outputs)
   print(api$getCapabilities())
   
   # Consume service by calling function, `manualTransmission`
   # contained in this service
   result <- api$manualTransmission(120, 2.8)

   # Print response output named `answer`
   print(result$output("answer")) # 0.6418125   

   # During this authenticated session, you can download the  
   # Swagger-based JSON file that defines this service, or do it
   # later with api <- getServices("<name>", "<version>")
   swagger <- api$swagger()
   cat(swagger, file = "swagger.json", append = FALSE) 
   ``` 

   In our example, we observe the same results as we did when it was locally executed.

   >[!NOTE]
   >As long as the package versions are the same on R Server as they are locally, you should get the same results.

1. During the authenticated session in which you published the service, download the Swagger-based JSON file specific to this service so that you or other authenticated users can test and consume the service. This Swagger file is generated when the service was published. It will be downloaded to the local file system. 

   See part 3 below for code to get this file after the session ends.

   ```R
   swagger <- api$swagger()
   cat(swagger, file = "swagger.json", append = FALSE) 
   ``` 
   
<br> 

<a name="share"></a>

### (3) Share service with others 

>[!IMPORTANT]
> Anyone who wishes to consume the service must have [authenticated access](security-authentication.md) to an instance of Microsoft R Server with its [operationalization feature configured](configuration-initial.md).

When the web service is published, a Swagger-based JSON file is generated automatically to define the service. You can now hand off this file to share the predictive web service with **other authenticated users of R Server**, such as:

Data scientists can also explore and consume Web services directly in R using some of the functions in the `mrsdeploy` package installed with Microsoft R Server and R Client. 

#### Collaborate with data scientists

Other data scientist may want to explore, test, and consume Web services directly in R using the functions in the `mrsdeploy` package installed with Microsoft R Server and R Client. 

As the owner of the service, you can share the name and version number for the service with fellow data scientists. 

After authenticating, other data scientists can use the `getServices` function in R to call the service. Then they can get details about the service and start consuming it.

```R
api <- getServices("mtService", "v1.0.0")

# Print capabilities to see what service can do
print(api$getCapabilities())
     
# Start interacting with the service, for example:
# Calling the function, `manualTransmission`
# contained in this service
result <- api$manualTransmission(120, 2.8)
```

#### Collaborate with application developers

Application developers can integrate web services into their applications using each service-specific Swagger-based JSON file along with the required inputs. 
   
The application developers can get the Swagger-based JSON file in one of two ways:

+ You can also send them the Swagger file you've downloaded yourself, which can sometimes be a faster approach.  Get the Swagger with:
   ```
   api <- getServices("<name>", "<version>")
   swagger <- api$swagger()
   cat(swagger, file = "swagger.json", append = FALSE) 
   ```

+ Or, they can request the file  **as an authenticated user with an [active bearer token](app-developer-get-started.md#authentication) in the request header** (since all API calls must be authenticated).   

  The URL is formed as follows:
  ```
  GET /api/<service-name>/<service-version>/swagger.json
  ```

Once the application developer has this Swagger-based JSON file, he or she can create client libraries for integration. Read "[Application Developer Get Started Guide](app-developer-get-started.md)".  

<!--## Example: Deploy an R script as a service-->

## How to execute R code remotely

You can use Microsoft R Client to run your R code locally and from R Client you can connect remotely to R Server to run your code there. You can easily switch between the local context and the remote context using `pause()` and `resume()` functions.  Learn more in this article, [Remote Execution in Microsoft R Server](remote-execution.md).

Requirements for remote execution include:

+ You must configure an R Integrated Development Environment (IDE) to work with [Microsoft R Client](../r-client-get-started.md). 
+ You must also have [authenticated access](security-authentication.md) to an instance of Microsoft R Server with its [operationalization feature configured](configuration-initial.md).
