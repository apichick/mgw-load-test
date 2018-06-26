# Overview

The objective of this project is to automatically set up an environment to make an Edge Microgateway load Test.

We have created all the jinja templates to automatically set all the VMs that are required. See below the list of VMs:

* Monitor

In this VM we have influxDB and Grafana. As we run the load test all the test metrics are sent to influxDB. Grafana has the influxDB datasources configured and a few dashboards that consume data from those datasources, so you can automatically see the load test results without having to set up anything.

* Microgateway

We will run Edge Microgateway in this VM. Microgateway was the oauth plugin enabled. On top of setting up and running Microgateway, in the startup script we are also importing and deploying the API proxy that we will be testing against, and creating a developer, an API product and a developer app. 

In this VM we have collectd installed and submitting metrics (CPU, memory, network, etc) to InfluxDB.

* Load Generator

We will use this VM to run the load test script using [k6](https://github.com/loadimpact/k6). The load test script will be automatically created for you in the following location: /tmp/script.js

The command to run the load test is the following one:

    $ k6 run --out influxdb=http://MONITOR_VM_IP:8086/k6 script.js

By default the test makes requests against http://MICROGATEWAY_VM_IP:8000/test/v1 sending the consumer key of the developer app that was created as a header named X-API-KEY. It ramps up from 0 to 1000 VUs in 30 minutes. You can edit it an change the test set up.

* Target

This VM will host the target that our Edge Microgateway proxy will be hitting. It runs a "Hello World" Node.js server app.

# Set up

1. Create a new GCP project

2. Set that project as your working project

        $ gcloud config set project PROJECT_ID

3. Create a keyring called apigee

        $ gcloud kms keyrings create apigee --location global

4. Create a new key for encryption called in the apigee keyring

        $ gcloud kms keys create apigee --purpose=encryption --keyring=apigee

5. Create a new file called apigee-credentials.txt as follows:

        $ echo "APIGEE-USER:APIGEE-PASSWORD" > apigee-credentials.txt

6. Create a file storing your Apigee credentials as follows:

        $ gcloud kms encrypt \
            --location=global  \
            --keyring=apigee \
            --key=apigee \
            --version=1 \
            --plaintext-file=apigee-credentials.txt \
            --ciphertext-file=apigee-credentials.txt.enc

6. Create a bucket and upload there the encrypted file apigee-credentials.txt.enc

        $ gsutil mb -c regional -l REGION gs://BUCKET-NAME

7. Edit the microgateway-load-test.yaml and set the following properties for the microgateway resource:

        credentialsFile: gs://BUCKET-NAME/apigee-credentials.txt.enc
        organization: APIGEE-ORGANIZATION
        environment: APIGEE-ENVIRONMENT

8. Create a new deployment with deployment manager

        $ gcloud deployment-manager deployments create microgateway-load-test \
        --config microgateway-load-test.yaml




