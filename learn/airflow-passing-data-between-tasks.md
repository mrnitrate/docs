---
title: "Pass data between tasks"
sidebar_label: "Pass data between tasks"
id: airflow-passing-data-between-tasks
---

<head>
  <meta name="description" content="Learn more about the most common methods to implement data sharing between your Airflow tasks, including an in-depth explanation of XCom." />
  <meta name="og:description" content="Learn more about the most common methods to implement data sharing between your Airflow tasks, including an in-depth explanation of XCom." />
</head>

Sharing data between tasks is a very common use case in Airflow. If you've been writing DAGs, you probably know that breaking them up into smaller tasks is a best practice for debugging and recovering quickly from failures. What do you do when one of your downstream tasks requires metadata about an upstream task, or processes the results of the task immediately before it?

There are a few methods you can use to implement data sharing between your Airflow tasks. In this guide, you'll walk through the two most commonly used methods, learn when to use them, and use some example DAGs to understand how they can be implemented.

:::info

All code in this guide can be found in [the Github repo](https://github.com/astronomer/airflow-guide-passing-data-between-tasks).

:::

## Assumed knowledge

To get the most out of this guide, you should have an understanding of:

- Airflow operators. See [Operators 101](what-is-an-operator.md).
- DAG writing best practices. See [DAG writing best practices in Apache Airflow](dag-best-practices.md).

## Best practices

Before you dive into the specifics, there are a couple of important concepts to understand before you write DAGs that pass data between tasks.

### Ensure idempotency

An important concept for any data pipeline, including an Airflow DAG, is [idempotency](https://en.wikipedia.org/wiki/Idempotence). This is the property whereby an operation can be applied multiple times without changing the result. This concept is often associated with your entire DAG. If you execute the same DAGRun multiple times, you will get the same result. However, this concept also applies to tasks within your DAG. If every task in your DAG is idempotent, your full DAG is idempotent as well.

When designing a DAG that passes data between tasks, it's important that you ensure that each task is idempotent. This helps with recovery and ensures no data is lost if a failure occurs.

### Consider the size of your data

Knowing the size of the data you are passing between Airflow tasks is important when deciding which implementation method to use. As you'll learn, XComs are one method of passing data between tasks, but they are only appropriate for small amounts of data. Large data sets require a method making use of intermediate storage and possibly utilizing an external processing framework.

## XCom

The first method for passing data between Airflow tasks is to use XCom, which is a key Airflow feature for sharing task data.

### What is XCom

[XCom](https://airflow.apache.org/docs/apache-airflow/stable/concepts.html?highlight=xcom#concepts-xcom) is a built-in Airflow feature. XComs allow tasks to exchange task metadata or small amounts of data. They are defined by a key, value, and timestamp.

XComs can be "pushed", meaning sent by a task, or "pulled", meaning received by a task. When an XCom is pushed, it is stored in the Airflow metadata database and made available to all other tasks. Any time a task returns a value (for example, when your Python callable for your [PythonOperator](https://registry.astronomer.io/providers/apache-airflow/modules/pythonoperator) has a return), that value is automatically pushed to XCom. Tasks can also be configured to push XComs by calling the `xcom_push()` method. Similarly, `xcom_pull()` can be used in a task to receive an XCom.

You can view your XComs in the Airflow UI by going to **Admin** > **XComs**. You should see something like this:

![XCom UI](/img/guides/xcom_ui.png)

### When to use XComs

XComs should be used to pass small amounts of data between tasks. For example, task metadata, dates, model accuracy, or single value query results are all ideal data to use with XCom.

While there is nothing stopping you from passing small data sets with XCom, be very careful when doing so. This is not what XCom was designed for, and using it to pass data like pandas dataframes can degrade the performance of your DAGs and take up storage in the metadata database.

XCom cannot be used for passing large data sets between tasks. The limit for the size of the XCom is determined by which metadata database you are using:

- Postgres: 1 Gb
- SQLite: 2 Gb
- MySQL: 64 Kb

You can see that these limits aren't very big. And even if you think your data might meet the maximum allowable limit, don't use XComs. Instead, use intermediary data storage, which is more appropriate for larger amounts of data.  

### Custom XCom backends

[Custom XCom Backends](https://airflow.apache.org/docs/apache-airflow/stable/concepts.html?highlight=xcom#custom-xcom-backend) are a new feature available in Airflow 2.0 and greater. Using an XCom backend means you can push and pull XComs to and from an external system such as S3, GCS, or HDFS rather than the default of Airflow's metadata database. You can also implement your own serialization and deserialization methods to define how XComs are handled. This is a concept in its own right and you can learn more by reading [Custom XCom Backends](custom-xcom-backends.md).

### Example DAG using XComs

In this section, you'll review a DAG that uses XCom to pass data between tasks. The DAG uses XComs to analyze cat facts that are retrieved from an API. To implement this use case, the first task makes a request to the [cat facts API](http://catfact.ninja/fact) and pulls the `fact` parameter from the results. The second task takes the results from the first task and performs an analysis. This is a valid use case for XCom, because the data being passed between the tasks is a short string.

```python
import json
from datetime import datetime, timedelta
import requests
from airflow import DAG
from airflow.operators.python import PythonOperator

def get_a_cat_fact(ti):
    """
    Gets a cat fact from the CatFacts API
    """
    url = 'http://catfact.ninja/fact'
    res = requests.get(url)
    ti.xcom_push(key='cat_fact', value=json.loads(res.text)['fact'])

def analyze_cat_facts(ti):
    """
    Prints the cat fact
    """
    cat_fact=ti.xcom_pull(key='cat_fact', task_ids='get_a_cat_fact')
    print('Cat fact for today:', cat_fact)
    #run some analysis here

with DAG('xcom_dag',
         start_date=datetime(2021, 1, 1),
         max_active_runs=2,
         schedule=timedelta(minutes=30),
         default_args={
            'retries': 1,
            'retry_delay': timedelta(minutes=5)
        },
         catchup=False
         ) as dag:

    get_a_cat_fact = PythonOperator(
        task_id = 'get_a_cat_fact',
        python_callable=get_a_cat_fact
    )

    analyze_cat_data = PythonOperator(
        task_id = 'analyze_data',
        python_callable=analyze_cat_facts
    )

    get_cat_data >> analyze_cat_data
```

In this DAG there are two `PythonOperator` tasks which share data using the `xcom_push` and `xcom_pull` functions. In the `get_a_cat_fact` function, the `xcom_push` method was used to allow the `key` name to be specified. Alternatively, the function could be configured to return the `cat_fact` value, because any value returned by an operator in Airflow is automatically pushed to XCom.

For the `xcom_pull` call in the `analyze_cat_facts` function, you specify the `key` and `task_ids` associated with the XCom you want to retrieve. This allows you to pull any XCom value (or multiple values) at any time into a task. It does not need to be from the task immediately prior as shown in this example.

If you run this DAG and then go to the XComs page in the Airflow UI, you'll see that a new row has been added for your `get_a_cat_fact` task with the key `cat_fact` and Value returned from the API.

![Example XCom](/img/guides/example_xcom.png)

In the logs for the `analyze_data` task, you can see the value from the prior task was printed, meaning the value was successfully retrieved from XCom.

![Example XCom Log](/img/guides/example_xcom_log.png)


## TaskFlow API

Another way to implement the previous DAG is to use the [TaskFlow API](https://airflow.apache.org/docs/apache-airflow/stable/tutorial_taskflow_api.html) that was released with Airflow 2.0. With the TaskFlow API, returned values are pushed to XCom as usual, but XCom values can be pulled simply by adding the key as an input to the function as shown in the following DAG:

```python
from airflow.decorators import dag, task
from datetime import datetime

import requests
import json

url = 'http://catfact.ninja/fact'

default_args = {
    'start_date': datetime(2021, 1, 1)
}

@dag('xcom_taskflow_dag', schedule='@daily', default_args=default_args, catchup=False)
def taskflow():

    @task
    def get_a_cat_fact():
        """
        Gets a cat fact from the CatFacts API
        """
        res = requests.get(url)
        return{'cat_fact': json.loads(res.text)['fact']}

    @task
    def print_the_cat_fact(cat_fact: str):
        """
        Prints the cat fact
        """
        print('Cat fact for today:', cat_fact)
        #run some further cat analysis here

    # Invoke functions to create tasks and define dependencies
    print_the_cat_fact(get_a_cat_fact())

dag = taskflow()
```

This DAG is functionally the same as the previous one, but thanks to the TaskFlow API there is less code required overall, and no additional code required for passing the data between the tasks using XCom.

## Intermediary data storage

As mentioned previously, XCom is a great option for sharing data between tasks because it doesn't rely on any tools external to Airflow itself. However, it is only designed to be used for very small amounts of data. What if the data you need to pass is a little bit larger, for example a small dataframe?

The best way to manage this use case is to use intermediary data storage. This means saving your data to some system external to Airflow at the end of one task, then reading it in from that system in the next task. This is commonly done using cloud file storage such as S3, GCS, or Azure Blob Storage, but it could also be done by loading the data in either a temporary or persistent table in a database.

While this is a great way to pass data that is too large to be managed with XCom, you should still exercise caution. Airflow is meant to be an orchestrator, not an execution framework. If your data is very large, it is probably a good idea to complete any processing using a framework like Spark or compute-optimized data warehouses like Snowflake or dbt.

### Example DAG

Building on the previous cat fact example, you are now interested in getting more cat facts and processing them. This case would not be ideal for XCom, but since the data returned is a small dataframe, it can be processed with Airflow.

```python
from datetime import datetime, timedelta
from io import StringIO

import pandas as pd
import requests
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.amazon.aws.hooks.s3 import S3Hook

s3_conn_id = 's3-conn'
bucket = 'astro-workshop-bucket'

def upload_to_s3(cat_fact_number):
    # Instantiate
    s3_hook = S3Hook(aws_conn_id=S3_CONN_ID)

    # Base URL
    url = 'http://catfact.ninja/fact'

    # Grab data
    res = requests.get(url)

    # Take string, upload to S3 using predefined method
    s3_hook.load_string(res.text, 'cat_fact_{0}.csv'.format(cat_fact_number), bucket_name=BUCKET, replace=True)

def process_data(cat_fact_number):
    '''Reads data from S3, processes, and saves to new S3 file
    '''
    # Connect to S3
    s3_hook = S3Hook(aws_conn_id=s3_conn_id)

    # Read data
    data = StringIO(s3_hook.read_key(key='cat_fact_{0}.csv'.format(cat_fact_number), bucket_name=bucket))
    df = pd.read_csv(data, sep=',')

    # Process data
    processed_data = df[['fact']]

    # Save processed data to CSV on S3
    s3_hook.load_string(processed_data.to_string(), 'cat_fact_{0}_processed.csv'.format(cat_fact_number), bucket_name=bucket, replace=True)

with DAG('intermediary_data_storage_dag',
         start_date=datetime(2021, 1, 1),
         max_active_runs=1,
         schedule='@daily',
         default_args={
            'retries': 1,
            'retry_delay': timedelta(minutes=1)
            },
         catchup=False
         ) as dag:

    generate_file = PythonOperator(
        task_id='generate_file_{0}'.format(state),
        python_callable=upload_to_s3,
        op_kwargs={'cat_fact_number': 1}
    )

    process_data = PythonOperator(
        task_id='process_data_{0}'.format(state),
        python_callable=process_data,
        op_kwargs={'cat_fact_number': 1}
    )

    generate_file >> process_data
```

In this DAG you used the [S3Hook](https://registry.astronomer.io/providers/amazon/modules/s3hook) to save data retrieved from the API to a CSV on S3 in the `generate_file` task. The `process_data` task then takes the data from S3, converts it to a dataframe for processing, and then saves the processed data back to a new CSV on S3.
