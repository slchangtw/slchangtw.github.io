---
title: "Testing Functions with Localstack for Cloud Service Interactions"
date: 2023-10-08T15:31:49+08:00
tags: ["Docker", "Docker-compose", "Localstack"]
---

In software development, it's common to write functions that interact with cloud services. Instead of testing these functions against the actual cloud services, we can utilize Localstack to create local mock versions of these services. This approach enables us to test our functions without network dependencies and without altering the state of the real cloud services. In this post, we will explore how to use Localstack to test functions that interact with AWS DynamoDB.
<!--more-->

The code for this post can be found [here](https://github.com/slchangtw/blog_examples/tree/main/ci_tests).

# Task Description

AWS DynamoDB is a key-value database, and it proves especially valuable when a function requires significant time to compute a result. Additionally, DynamoDB allows the use of sort keys, permitting a single primary key to have multiple results (it can happen when we modify part of the function's logic, but we prefer not to recreate the entire table.) that can be sorted. In such scenarios, DynamoDB can serve as a caching table. If the function is invoked with the same input, we can directly retrieve the result from the DynamoDB table."

To simplify the task, we create a class called `TextManipulator` capable of reversing input text. Additionally, it will query a DynamoDB table to check if the input text already exists. If it's found, the class will return the reversed text from the table. Otherwise, it will calculate the reversed text and store it in the table.

To proceed with this task, you should have `docker-compose` installed on your machine.

# Constructing the Class and Tests

The class is defined as follows:

```python
import os
from datetime import datetime
from typing import Optional

import boto3
from boto3.dynamodb.conditions import Key

DYNAMODB_TABLE = "reversed_texts"


class TextManipulator:
    def __init__(self):
        self._dynamodb_table = self._get_dynamodb_table()

    def _get_dynamodb_table(self) -> boto3.resource:
        region = os.environ.get("AWS_REGION")
        end_point_url = os.environ.get("AWS_ENDPOINT_URL")

        dynamodb = boto3.resource(
            "dynamodb", region_name=region, endpoint_url=end_point_url
        )
        return dynamodb.Table(DYNAMODB_TABLE)

    def _reverse_text(self, text: str) -> str:
        return text[::-1]

    def query_dynamodb(self, text: str) -> Optional[str]:
        response = self._dynamodb_table.query(
            KeyConditionExpression=Key("text").eq(text),
            ScanIndexForward=False,
            Limit=1,
        )

        if response["Count"] != 0:
            return response["Items"][0]["reversed_text"]

        return None

    def write_dynamodb(self, text: int, reversed_text: str) -> None:
        self._dynamodb_table.put_item(
            Item={
                "text": text,
                "updated_at": datetime.now().isoformat(),
                "reversed_text": reversed_text,
            }
        )

    def reverse(self, text: str) -> str:
        if reversed_text := self.query_dynamodb(text):
            return reversed_text

        reversed_text = self._reverse_text(text)
        self.write_dynamodb(text, reversed_text)

        return reversed_text
```

Let's examine the noteworthy methods.

`_get_dynamodb_table`: This method generates a DynamoDB table object. It relies on the environment variables `AWS_REGION` and `AWS_ENDPOINT_URL` to determine the region and endpoint URL of the table, enabling us to distinguish between the production and Localstack instances of the table.

`write_dynamodb`: "This method stores the input text and its corresponding reversed text in the DynamoDB table. It utilizes the current timestamp as the sort key, indicating when the entry was last updated.

`query_dynamodb`: This method queries the DynamoDB table for the input text. It will return the latest reversed text if it exists, or `None` otherwise.

`reverse`: The main method of the class. It first queries the DynamoDB table for the input text. If the text is found, it will return the reversed text. Otherwise, it will calculate and return the reversed text while also storing it in the table.

The testing file is defined as follows. It includes two auxiliary functions for creating and deleting the DynamoDB table. In the test,
1. Create the table.
2. Test with the text 'grandpa,' which is not in the table.
3. Verify the reverse method for correct reversed text and storage in the table.
4. Add a new result (assuming we have updated the function's logic) to the table and verify that the `query_dynamodb` method returns the latest updated result.
5. Delete the table.


```python
import os

import boto3

from src.text_manipulator import TextManipulator


def create_table() -> None:
    dynamodb = boto3.resource(
        "dynamodb",
        region_name=os.environ.get("AWS_REGION"),
        endpoint_url=os.environ.get("AWS_ENDPOINT_URL"),
    )
    dynamodb.create_table(
        TableName="reversed_texts",
        KeySchema=[
            {"AttributeName": "text", "KeyType": "HASH"},  # Partition_key
            {"AttributeName": "updated_at", "KeyType": "RANGE"},  # Sort_key
        ],
        AttributeDefinitions=[
            {"AttributeName": "text", "AttributeType": "S"},
            {"AttributeName": "updated_at", "AttributeType": "S"},
        ],
        ProvisionedThroughput={"ReadCapacityUnits": 10, "WriteCapacityUnits": 1},
    )


def delete_table() -> None:
    dynamodb = boto3.resource(
        "dynamodb",
        region_name=os.environ.get("AWS_REGION"),
        endpoint_url=os.environ.get("AWS_ENDPOINT_URL"),
    )
    dynamodb.Table("reversed_texts").delete()


def test_text_manipulator():
    try:
        create_table()

        text_manipulator = TextManipulator()

        assert text_manipulator.query_dynamodb("grandpa") is None

        assert text_manipulator.reverse("grandpa") == "apdnarg"
        assert text_manipulator.query_dynamodb("grandpa") == "apdnarg"

        text_manipulator.write_dynamodb("grandpa", "grandson")
        assert text_manipulator.query_dynamodb("grandpa") == "grandson"
    finally:
        delete_table()
```

To complete this task successfully, you'll also require the Dockerfile and pyproject.toml files, which can be located in the aforementioned repository. If you are unfamiliar with these files, you can refer to this [post]({{< ref "20231005_build_docker_image.md" >}}) for more detailed information.


# Using docker-compose to Create Localstack Instance

To enable our tests to utilize Localstack, we'll employ docker-compose to create a Localstack instance. The `docker-compose.yml` file is defined as follows. It includes two services, namely `tests` and `localstack`. The tests service is constructed using the `Dockerfile` in the current directory, with environment variables set to ensure the `TextManipulator` class interacts with the local instance correctly. The `localstack` service is built from the Localstack image, exposing port 4566 (the default port for DynamoDB) and configuring necessary environment variables such as `SERVICES` and `DEBUG` as required by Localstack


```yaml
services:
  tests:
    container_name: tests
    build:
      context: .
      dockerfile: ./Dockerfile
    environment:
      - AWS_ACCESS_KEY_ID=test
      - AWS_SECRET_ACCESS_KEY=test
      - AWS_REGION=us-east-1
      - AWS_ENDPOINT_URL=http://localstack:4566
    depends_on:
      - localstack
  localstack:
    image: localstack/localstack
    container_name: localstack
    restart: always
    ports:
      - "4566:4566"
    environment:
      - SERVICES=dynamodb
      - DEBUG=1
```

# Running the Tests

Initially, we need to build the Docker image. Following that, we can start the `tests` service defined in the `docker-compose.yml` file and run the tests. Note that the initial image download may take some time. Eventually, you will receive the test results

Although we've specified the `--rm` flag in the `docker-compose` run command, please note that the `localstack` service won't be removed by this flag. It will only be removed when you execute `docker-compose down`.

```bash
$ docker-compose -f ./docker-compose.yml build
$ docker-compose -f ./docker-compose.yml run --rm tests

$ docker-compose down # To stop the localstack service
```

# Learn More

- [Try Docker compose](https://docs.docker.com/compose/gettingstarted/).
- [Integration tests (using Localstack)](https://docs.localstack.cloud/contributing/integration-tests/). The provided principles are beneficial for developing integration tests with Localstack.
