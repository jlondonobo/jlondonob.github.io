---
title: Building a quick and cost-effective data pipeline with AWS SDK for pandas and AWS Lambda
description: A guide on building quick and cost-effective pipelines in Python with AWS SDK for pandas and AWS Lambda.
date: "2024-01-21"
draft: false
---

Modern data pipelines are generally built on a dedicated server. Services like Airflow, Prefect, Mage, and Dagster manage data transfer and scheduling by constantly running on a server. This works fine for complex setups if there's a large team to develop and maintain such infrastructure, and costs are not a problem. However, if you're a solo developer or a small company, you might want to set up a simpler solution that you can deploy for virtually zero cost in a short period of time.

This article provides a guide on building a data pipeline that's virtually free and which can be deployed to production in less than an hour. To build such a pipeline you'll leverage the ease of use of **[AWS SDK for pandas](https://aws-sdk-pandas.readthedocs.io/en/stable/index.html)**(formerly AWS Data Wrangler) and the cost-effective runtime environment from **AWS Lambda**.

The article is divided in 2 sections:

- [Introduction to the setup](): Introduces the tools you'll use to build the pipeline. If you want to learn by doing, consider skipping this section.
- [Building the pipeline](): Goes on a detailed explanation of setting up the pipeline.

## TL;DR

Using **AWS SDK for pandas** and **Lambda functions** together allows you to **quickly** and **cost-effectively** build data pipelines. However, AWS SDK for pandas and its dependencies exceed the size limits for a **.zip Lambda deployment package**.

**Lambda layers** effectively address this issue. [AWS SDK for pandas managed layers](https://aws-sdk-pandas.readthedocs.io/en/stable/layers.html) offer a lightweight layer that contains both AWS SDK for pandas and pandas.

For your pipeline to function effectively, set a trigger and increase the function's **memory** and **timeout** settings.

For the complete code, see [aws-sdk-lambda-finance-demo](https://github.com/jlondonobo/aws-sdk-lambda-finance-demo).

## Introduction to the setup

### Why use AWS SDK for pandas and Lambda functions to build a data pipeline

Using **AWS SDK for pandas** and **Lambda functions** together allows you to **quickly** and **cost-effectively** build data pipelines. This setup is ideal for small or personal projects where speed and costs are limiting factors.

**AWS SDK for pandas** streamlines interactions with **AWS data services** like Athena, Glue, RedShift, and S3. It eliminates the need for complex `boto3` coding, simplifying ETL tasks such as reading and writing data from and to data lakes, warehouses, and databases.

**Lambda functions** provide a temporary runtime environment, eliminating the need for an EC2 instance setup and maintenance. Lambda functions also offer the following benefits

- **Event driven**: They can run on a schedule (using cron), respond to file uploads, or HTTP requests.
- **Cost-effective**: Running a 30-second job on a 1 GB runtime twice daily costs about 3 cents, with a generous free tier.
- **Quick to deploy**: Allows for rapid development and deployment of pipelines, particularly effective with a [CI/CD pipeline]().

### A solution for using AWS SDK for pandas in Lambda functions

AWS SDK for pandas and its dependencies exceed the size limits for a **.zip Lambda deployment package**. AWS Lambda sets a maximum size for deployment packages of **50 MB for compressed files** and **250 MB for uncompressed files** (uploaded to S3). However, the AWS SDK for pandas is 90 MB compressed and 268 MB uncompressed.

**Lambda layers** effectively address this issue. While alternatives like **container image deployment packages** exist, Lambda layers keep you within the **.zip deployment package** framework, offering advantages like the following:

- **Ease of use**: Creating and deploying zip files is straightforward, especially for smaller Lambda functions.
- **AWS Integration**: Zip deployments integrate smoothly with AWS services like CodeBuild and the AWS SDK.

I suggest using [AWS SDK for pandas managed layers](https://aws-sdk-pandas.readthedocs.io/en/stable/layers.html) for simplicity and efficiency. They come pre-equipped with the following essential dependencies:

- AWS SDK for pandas
- Pandas
- Other minimal dependencies

AWS SDK for pandas keeps layers updated for all active Python runtimes, regions, and architectures. For a detailed list of available layers, visit [AWS Lambda Managed Layers](https://aws-sdk-pandas.readthedocs.io/en/stable/layers.html).

## End-to-end pipeline

This section offers a step-by-step guide on building a data pipeline with AWS SDK for pandas and AWS lambda with an application for finance. Despite its focus on finance, the pipeline can be customized to call any API.

The pipeline involves the following steps:

1. Call a simulated financial API.
2. Store the API's response in AWS Glue.

By the end of this tutorial, your deployment package should look like this:

```
aws_sdk_lambda_finance_demo
├── api
│   ├── __init__.py
│   └── finance.py
└── lambda_function.py
```

This deployment package **doesn't use external dependencies**. However, many real world applications do. If you'd like to add external dependencies to your deployment package, see [Creating a .zip deployment package with dependencies](https://docs.aws.amazon.com/lambda/latest/dg/python-package.html#python-package-create-dependencies).

For the complete code, see [aws-sdk-lambda-finance-demo](https://github.com/jlondonobo/aws-sdk-lambda-finance-demo).

Note: Implementing this pipeline might result in AWS charges if your free-tier trial has expired. Ensure to deactivate any services you enable for this tutorial.

### Step 1: Write the lambda function

The first step to building the pipeline is to actually write it. Write your own pipeline or use the following sample to get started:

```python
# aws_sdk_lambda_finance_demo/lambda_function.py
import awswrangler as wr
import pandas as pd
from api import finance


def create_database_if_not_exists(database_name):
    # Check if the database exists
    databases = wr.catalog.get_databases()
    if not any(db["Name"] == database_name for db in databases):
        wr.catalog.create_database(database_name)


def lambda_handler(event, context):
    # Fetch data from the API
    daily_stocks = finance.call_api()

    # Convert the response to a DataFrame
    stocks = pd.DataFrame(daily_stocks)

    # Define your database name
    database_name = "market_tracker"

    # Create the database if it doesn't exist
    create_database_if_not_exists(database_name)

    # Define the S3 path where you want to store the data
    s3_path = "s3://acme/glue/market_tracker/daily_stocks"
    table_name = "daily_stocks"

    # Write the DataFrame to the Glue table, appending to the existing table
    wr.s3.to_parquet(
        df=stocks,
        path=s3_path,
        dataset=True,
        database=database_name,
        table=table_name,
        mode="append"
    )

```

Note the absence of credentials in the `lambda_function.py` file. If you're following along with the sample files, ensure the **AWS CLI** is set up on your system . If you haven't configured the AWS CLI, see [Setting up the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html) .

```python
# aws_sdk_lambda_finance_demo/api/finance.py
import random
import datetime

def call_api():
    # Fixed set of three tickers
    tickers = ["AAPL", "MSFT", "TSLA"]

    # Generate data for each ticker
    data_collection = []
    for ticker in tickers:
        data = {
            "date": datetime.date.today().isoformat(),
            "ticker": ticker,
            "stock_price": round(random.uniform(100, 500), 2),
            "trading_volume": random.randint(1000, 10000),
        }
        data_collection.append(data)
    return data_collection
```

### Step 2: Create the lambda function on the AWS console

Once you wrote the pipeline, go ahead and create a Lambda function on the AWS console. To create a function, take the following steps:

1. Go to **AWS Lambda console**.
2. Locate the **Functions tab**.
3. Click on **Create function**.
4. Select **Author from scratch**.
5. Name the function.
6. Select a **Python runtime**.
7. Click on **Create function**.

Before choosing a runtime, make sure there's a [managed layer](https://aws-sdk-pandas.readthedocs.io/en/stable/layers.html) for it. AWS SDK for pandas' managed layers do not support all runtimes. If you select a runtime for which managed layers are not available, you won't be able to attach the layer.

![[images/01_create_function.png]]

![[images/02_function_setup.png]]

#### Step 3: Update the function's configuration and execution role

The default state of Lambda functions is not ideal for running data pipelines. Memory is capped at 128 MB, the timeout set to 3 seconds and the function has no permissions to interact with other AWS services. With this configuration, any pipeline that uses more than 128 MB of memory, takes longer than 3 seconds or interacts with other AWS services will be cancelled.

Consequently, for the function to run you must update its runtime's configuration. Here are recommended values for memory and timeout:

- **Memory**: AWS SDK for pandas [suggest assigning at least 512 MB of memory](https://aws-sdk-pandas.readthedocs.io/en/stable/install.html#aws-lambda-layer) . Tune it up if you expect to move large amounts of data. I wouldn't recommend tuning it down, as it can make even small functions fail unexpectedly.
- **Timeout**: Set the timeout parameter to at least 1.5x-2x the average execution time of your pipeline. If your function's execution times vary significantly, set a higher value. For example, if your pipeline usually takes 30 seconds to complete, you should set the timeout parameter to at least 45-60 seconds.

If your function interacts with other AWS resources like Glue, S3 or Redshift, you must update the function's **execution role**. Customizing the execution role of a Lambda function is outside the scope of this tutorial. However, the following tips can hep you build your own

- [IAM Roles Tutorial](https://www.youtube.com/watch?v=lAkadozQwdo) by [Stephane Maarek](https://www.youtube.com/@StephaneMaarek).
- [Example Lambda execution role](https://github.com/jlondonobo/aws-sdk-lambda-finance-demo/blob/main/role_example.json).
- Tip: Describe the pipeline to ChatGPT and ask it to generate an AWS JSON policy.

![[images/03_start_config.png]]

![[images/04_edit_config.png]]

#### Step 4: Add the AWS SDK for pandas managed layer

Once you correctly configured the function, add the AWS SDK for pandas managed function. To add the function, take the following steps:

1. Locate the **Code** tab.
2. Click on the **Add layer** button, located in the **Layers** panel.
3. Select the **AWSSDKPandas layer** and its **version** from the dropdown menus.
4. Click on **Add**.

![[images/05_layers_start.png]]
![[images/06_layers_config.png]]

If you successfully added the layer, the **"Layers" diagram element** will show a (1) next to it.
![[images/07_layers_confirmation.png]]

### Step 5: Set up a cron trigger

In the final step of the configuration, you'll set up the scheduler. Namely, an EventBridge **cron trigger event**. This trigger will run the function at pre-specified times.

Cron statements are minute-precise. They have control of running a program at a minute level, but not at a second level. If you need second-level control, start the function a minute earlier, and write the starting logic in the code.

To set up a cron trigger, take the following steps: 2. Click on the **Add trigger**. 3. Select **EventrBridge trigger**. 4. Select **Create a new rule**. 5. Name the trigger and add a description. 6. Select **Scheduled expression**. 7. Define the function's execution schedule using cron notation. 8. Click on **Add**.
![[images/08_add_trigger.png]]

![[images/09_configure_trigger.png]]
EventBridge cron statements differ from traditional Unix-style cron statements. If you're unfamiliar with its syntax or want a refresher, ask ChatGPT to generate the statements for you. Take the following differences into consideration:

- EventBridge statements include a year field.
- EventBridge statements can use the ? wildcard.
- EventBridge statements are in UTC.

### Step 6: Upload the deployment package

The last step of the setup is to upload the _deployment package_. To do so, take the following steps:

1. Compress the `api/` and `lambda_function.py` into a .zip file.
2. Locate the **Code** tab.
3. Click the **Upload from** dropdown.
4. Select **.zip file**.
5. Drag and drop your deployment package.
6. Click on **save**.

![[images/10_upload_zip.png]]
![[images/11_drag_zip.png]]

### Step 7: Test the pipeline
The pipeline is now online. Test it by taking the following steps:

1. Go to the **Test** tab and hit the **Test** button.
2. Go to the **AWS Athena console** and query the table you just created.

![[images/12_test_function.png]]
