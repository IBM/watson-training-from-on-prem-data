# Train a cloud-based machine learning model from on-premise Db2 data

[![Build Status](https://api.travis-ci.org/IBM/watson-training-from-on-prem-data.svg?branch=master)](https://travis-ci.org/IBM/watson-training-from-on-prem-data)

Many companies and individuals struggle to use their on-premises data &mdash;
the kind of data that lives on a local machine, within a company data center, behind
a firewall &mdash; for machine learning in the cloud. It can be challenging
to find a quick, easy, and secure solution for connecting resources in a
protected environment to resources in the cloud.

With Db2, Watson Studio & Machine Learning, and Secure Gateway, it is possible
to establish a secure, persistent connection between your on-premises data and
the cloud to train machine learning models leveraging cloud computing resources
like Spark, elastic environments, and GPUs.

In this guide, we will create an on-premises Db2 database on our local
computer, populate it, and then connect it to Watson Studio via Secure Gateway.
Then, we'll read Buildings Violations data from this database and build a model
to predict the likelihood that a particular building will FAIL an inspection
based on historical data from the City of Chicago. After we build the model, we
will deploy it as an API endpoint with Watson Machine Learning that only
authorized users can access.

![Architecture](doc/source/images/architecture.png)

## Flow

1. Source data is retained in on-premises Db2 database.
1. Data is made accessible to Watson Studio via a Secure Gateway.
1. Data is utilized to train a cloud-based machine learning model.

## Included components

* [Watson Studio](https://www.ibm.com/cloud/watson-studio): Build and train
  AI models, and prepare and analyze data, in a single, integrated environment.
* [Secure Gateway](https://www.ibm.com/cloud/secure-gateway): Create a secure,
  persistent connection between your protected environment and the cloud.
* [IBM Db2](https://www.ibm.com/analytics/us/en/db2/): Relational database management system optimized
  to deliver industry-leading performance while lowering costs.

## Featured technologies

* [Docker](https://www.docker.com/): Docker containers wrap up software and its
  dependencies into a standardized unit for software development that includes
  everything it needs to run: code, runtime, system tools and libraries.

## Steps

1. [Create and populate an on-premises Db2 database](#create-and-populate-an-on-premises-db2-database)
1. [Create IBM Cloud service dependencies](#create-ibm-cloud-service-dependencies)
1. [Configure a secure gateway to IBM Cloud](#configure-a-secure-gateway-to-ibm-cloud)
1. [Create a Watson Studio project](#create-a-watson-studio-project)
1. [Create a machine learning model](#create-a-machine-learning-model)

### Create and populate an on-premises Db2 database

The fastest way to get started with Db2 on-premises is to use the no-charge
Db2 Developer Community Edition, which is Db2 running in a Docker container. However, if you already have
an on-premises Db2 database, feel free to substitute that instead.

We'll be populating the database with a sample dataset of building code
violations, provided by the city of Chicago.

If you don't already have Docker installed, follow the official [installation
guide](https://docs.docker.com/installation/).

Start by using [`docker run`](https://docs.docker.com/engine/reference/run/) to
launch Db2 community edition in a container running in the background.

> Including the `--env LICENSE="accept"` argument indicates your acceptance of
> the [license
> agreement](https://www-03.ibm.com/software/sla/sladb.nsf/displaylis/5DF1EE126832D3F185257DAB0064BEFA?OpenDocument)
> to use the software contained in the Docker image.

This command sets a password for the default instance user (`db2inst1`) to
`db2inst1-pwd` and binds Db2 to listen on port `50000` of your workstation:

```bash
docker run \
  --name db2 \
  --env LICENSE="accept" \
  --env DB2INST1_PASSWORD="db2inst1-pwd" \
  --publish 50000:50000 \
  --detach \
  ibmcom/db2express-c \
  db2start
```

Next, run a `db2` command inside the running container using [`docker
exec`](https://docs.docker.com/engine/reference/commandline/exec/) as the
`db2inst1` user to [create a
database](https://www.ibm.com/support/knowledgecenter/en/SSEPGG_10.5.0/com.ibm.db2.luw.admin.dbobj.doc/doc/t0004916.html)
named `onprem`:

```bash
docker exec db2 su - db2inst1 -c "db2 CREATE DATABASE onprem"
```

Next, we need to create a regular user for Watson Studio to connect as. Users
in Db2 are managed externally from Db2 itself, so let's create a system user
named `watson`, assign them a password (`secrete`), and grant them
database-level permissions.

```bash
docker exec db2 useradd -g db2iadm1 watson
```

```bash
docker exec db2 bash -c "echo secrete | passwd --stdin watson"
```

```bash
docker exec db2 su - db2inst1 -c "db2 CONNECT TO onprem; db2 'GRANT DBADM, CREATETAB, BINDADD, CONNECT, CREATE_NOT_FENCED, IMPLICIT_SCHEMA, LOAD ON DATABASE TO watson'"
```

Using the new `watson` user account, create a database table in the `watson`
database for building code violations named `violations`:

```bash
docker exec db2 su - db2inst1 -c "db2 CONNECT TO onprem USER watson USING secrete; db2 'CREATE TABLE violations(ID INTEGER, VIOLATION_CODE VARCHAR(20), INSPECTOR_ID VARCHAR(15), INSPECTION_STATUS VARCHAR(10), INSPECTION_CATEGORY VARCHAR(12), DEPARTMENT_BUREAU VARCHAR(30), ADDRESS VARCHAR(250), LATITUDE DOUBLE, LONGITUDE DOUBLE)'"
```

Then, use [`docker
cp`](https://docs.docker.com/engine/reference/commandline/cp/) to push the
[`violations.csv`](violations.csv) file (from this repository) into the `db2`
container.

```bash
docker cp violations.csv db2:/tmp/
```

Load the sample data into the `onprem` Db2 database:

```bash
docker exec db2 su - db2inst1 -c "db2 CONNECT TO onprem USER watson USING secrete; db2 'IMPORT FROM /tmp/violations.csv OF DEL SKIPCOUNT 1 INSERT INTO violations'"
```

At this point, you have an on-premises Db2 database loaded with sample data.

Before you proceed further, you also need to take note of your workstation's
LAN IP. You can find that address using either `hostname -I` or `ifconfig`,
depending on what platform you're on.

```bash
$ hostname -I
192.168.1.100

$ ifconfig
wlp4s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.100  netmask 255.255.255.0  broadcast 192.168.1.255
        ether 00:19:c3:14:ca:fb  txqueuelen 1000  (Ethernet)
        RX packets 38087684  bytes 48297697882 (48.2 GB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 4759813  bytes 781912123 (781.9 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

> Your workstation may have more than one IP, but any of them will likely work
> for the purposes of this code pattern, because Docker will bind to all
> network interfaces.

> It's worth mentioning that `loopback` addresses (e.g. `127.0.0.1`) will not
> work here, because IBM Cloud services consuming the Secure Gateway will not
> be able to distinguish between their own `loopback` interfaces and a
> `loopback` interface on your end of the Secure Gateway.

For the purposes of this code pattern, we'll assume that your LAN IP is
`192.168.1.100`.

### Create IBM Cloud service dependencies

In order to build and train your machine learning model, you'll first need to
[sign up for Watson Studio](https://www.ibm.com/cloud/watson-studio), which you
can do for free. In our use case, Watson Studio will rely on the Object Storage
service to store it's data, the Apache Spark service for data processing, and
the Machine Learning service for building machine learning models.

From the [IBM Cloud Catalog](https://cloud.ibm.com/catalog/), select the
[**Storage**](https://cloud.ibm.com/catalog?category=storage) category,
and then the [**Object
Storage**](https://cloud.ibm.com/catalog/services/cloud-object-storage)
service. Then, click **Create**.

![Create IBM Cloud Object Storage service](doc/source/images/watson-training-from-on-prem-data-01.png)

From the [IBM Cloud Catalog](https://cloud.ibm.com/catalog/), select the
[**Web and
Application**](https://cloud.ibm.com/catalog?category=app_services)
category, and then the [**Apache
Spark**](https://cloud.ibm.com/catalog/services/apache-spark) service.
Then, click **Create**.

![Create IBM Cloud Apache Spark instance](doc/source/images/watson-training-from-on-prem-data-02.png)

### Configure a secure gateway to IBM Cloud

The secure gateway allows limited network ingress to your on-premise network as
governed by an access control list (ACL). For our use case, we will allow
Watson Studio to securely communicate with your on-premise Db2 instance.

From the [IBM Cloud Catalog](https://cloud.ibm.com/catalog/), select the
[**Integration**](https://cloud.ibm.com/catalog?category=app_services)
category, and then the [**Secure
Gateway**](https://cloud.ibm.com/catalog/services/apache-spark) service.
Then, click **Create**.

![New secure gateway](doc/source/images/watson-training-from-on-prem-data-03.png)

From the **Secure Gateway** creation screen, select the **Essentials** plan and
click **Create**.

![New secure gateway](doc/source/images/watson-training-from-on-prem-data-04.png)

Once the gateway is created, click **Add Gateway** to add a gateway named _Db2_.

![New Secure Gateway](doc/source/images/watson-training-from-on-prem-data-05.png)

Select **Connect Client**, and take note of your **Gateway ID** and **Security
Token**. Choose **Docker** as the connection method.

![Connect Secure Gateway client](doc/source/images/watson-training-from-on-prem-data-06.png)

The console will provide you with a complete Docker command to download and run
the secure gateway client, which looks something like `docker run -it
ibmcom/secure-gateway-client $GATEWAY_ID -t $SECURITY_TOKEN` (with a real
gateway ID and security token unique to your Secure Gateway). Use the copy icon
to copy the command, and run it locally.

By default, the gateway starts with an access control list in a default-deny
state, meaning that all incoming connections through the secure gateway will be
blocked from accessing any network resources. To allow Watson Studio to access
your Db2 instance, we need to specifically allow access to the port published
by Docker on your workstation's LAN IP address (e.g. `192.168.1.100`):

    acl allow 192.168.1.100:50000

Incoming connections from Watson Studio which pass through the secure gateway
will now be able to access Db2.

> When you're finished with this code pattern, you can close the secure gateway
> by typing `quit` at the secure gateway prompt and pressing <kbd>Enter</kbd>.

### Create a Watson Studio project

From the [IBM Cloud Catalog](https://cloud.ibm.com/catalog/), select the
[**AI**](https://cloud.ibm.com/catalog?category=ai) category, and then
the [**Watson
Studio**](https://cloud.ibm.com/catalog/services/watson-studio) service.
Then, click **Create**.

![Create IBM Cloud Watson Studio service](doc/source/images/watson-training-from-on-prem-data-07.png)

Click the **Get Started** button to navigate to Watson Studio.

From Watson Studio, click the **New Project** button, and select **Complete**, when prompted, to enable all features in Watson Studio.

![Watson Studio](doc/source/images/watson-training-from-on-prem-data-08.png)

![Watson Studio](doc/source/images/watson-training-from-on-prem-data-09.png)

Enter `Violations` as the project name, and click **Create**.

![Watson Studio](doc/source/images/watson-training-from-on-prem-data-10.png)

Near the top right of the screen, select the **Add to project** dropdown and choose
**Connection**.

![New Watson Studio connection](doc/source/images/watson-training-from-on-prem-data-11.png)

Select **Db2** from the available options to connect to your on-premise Db2 server.

![New Watson Studio connection](doc/source/images/watson-training-from-on-prem-data-12.png)

Configure the connection as follows:

* **Name**: `On-Premise`
* **Database**: `onprem`
* **Hostname or IP Address**: your workstation's LAN IP (e.g. `192.168.1.100`)
* **Port**: `50000`
* **Secure Gateway**: &#9745; Use a secure gateway, and select _Db2_
* **Username**: `watson`
* **Password**: `secrete`

Click **Create** when you are ready.

![New connection configuration](doc/source/images/watson-training-from-on-prem-data-13.png)

In the secure gateway terminal, you may see a log message indicating that a
connection was successfully established from Watson Studio:

    [2018-08-31 10:38:38.708] [INFO] (Client ID K75lSQ0Oppd_d87) Connection #1 is being established to 192.168.1.100:50000

#### Refine the asset

Watson Machine Learning models do not directly support data assets from
on-premise Db2 instances, so we have to setup a conversion process to "refine"
the data asset into a `CSV` file in object storage.

Click the **Add to Project** dropdown again, and choose **Connected assets**.

![Add connected assets](doc/source/images/watson-training-from-on-prem-data-14.png)

Click **Select source**, then choose the **On-Premise** connection, then the
**WATSON** database, then the **VIOLATIONS** table, and finally, click the
**Select** button at the bottom of the screen.

![Select source](doc/source/images/watson-training-from-on-prem-data-15.png)

Name the data asset `Violations On-Premise` and click **Create**.

![Name the data asset](doc/source/images/watson-training-from-on-prem-data-16.png)

We now need to associate our Apache Spark service and Watson Machine Learning services
to Watson Studio so that Watson Studio can leverage them.

Navigate to the **Settings** tab of your project, and click **Add service**.
Choose **Spark** from the dropdown, switch to the **Existing** tab, select your Apache Spark instance
from the list, and click **Select**.

![Associate Apache Spark](doc/source/images/watson-training-from-on-prem-data-17.png)

Once more, click **Add service**. Choose **Watson** from the dropdown,
select your Watson Machine Learning instance from the catalog, click **Create**, and then **Confirm**.

![Associate Watson](doc/source/images/watson-training-from-on-prem-data-18.png)

From your project overview, click the **Assets** tab, and click on
**Violations On-Premise** (with **Data Asset** in the **Type** column).

![Data assets](doc/source/images/watson-training-from-on-prem-data-19.png)

At the top right, click **Refine**. We don't need to
manipulate the data, so simply click the "run" button labeled with a
**&#9654;** icon at the top right. The data flow output will show that you're
creating a `CSV` file, which will be saved into your object storage bucket.
Click **Save and Run**.

![Refine data asset](doc/source/images/watson-training-from-on-prem-data-20.png)

You can then opt to view the data flow's progress by
clicking **View Flow**.

From your project **Assets** screen, you should now see a new **Data asset**
named `Violations On-Premise_shaped.csv`.

![Shaped data](doc/source/images/watson-training-from-on-prem-data-21.png)

### Create a machine learning model

From your project's **Assets** screen, click **New Watson Machine Learning
model**.

![New machine learning model](doc/source/images/watson-training-from-on-prem-data-22.png)

Name your model **Violations Predictor**, select your Apache Spark service
instance from the **Spark Service or Environment** dropdown, and click
**Create**.

![Violations predictor](doc/source/images/watson-training-from-on-prem-data-23.png)

You'll then be asked to choose your data asset. Use the radio button to select
the CSV file you just created, named `Violations On-Premise_shaped.csv`, and click
**Next**.

![Shaped data](doc/source/images/watson-training-from-on-prem-data-24.png)

When the data is finished loading, you'll be asked to **Select a technique**.
Choose `INSPECTION_STATUS (String)` as your **Column value to predict (Label
Col)**, and choose **Multiclass Classification** as your technique.

Next, we need to manually select which classifier we'd like to use. Use the
**Add Estimators** button to add a **Decision Tree Classifier**, and then
repeat to add a **Random Forest Classifier**.

Finally, click **Next**.

![Configure model](doc/source/images/watson-training-from-on-prem-data-25.png)

When training for each model is complete (_Trained &amp; Evaluated_), review
each of their performance metrics, select the one that best suits your use case
(we'll use the _DecisionTreeClassifier_ here). Click **Save** to store your
model.

![Model training](doc/source/images/watson-training-from-on-prem-data-26.png)

You can now deploy the model to use in your own application. Click **Add Deployment**.

![Model trained](doc/source/images/watson-training-from-on-prem-data-27.png)

Name the model **Inspection Status**, and click **Save**.

![Model deployment](doc/source/images/watson-training-from-on-prem-data-28.png)

Wait for your model to be deployed (_DEPLOY_SUCCESS_), and then click on it.

![Model deployed](doc/source/images/watson-training-from-on-prem-data-29.png)

The deployed model will be provided with documentation to consume the model in several different programming languages.

![Ready to consume cloud-based model](doc/source/images/watson-training-from-on-prem-data-30.png)

> **[Was this pattern helpful to you?](https://goo.gl/forms/mgFPdKgiblVlKTsq1)** Answer three short questions to let us know.

## Troubleshooting

* `No secure gateways found`: Your secure gateway client (in Docker) is not
  connected. Verify that the Docker container is running and that the secure
  gateway ID, token, and IP address allowed by the ACL are all configured
  correctly.
* `Retrieve instance summary error: An error occurred when getting instance
  summary data.`: This is a transient error that may occur while creating a
  secure gateway service. It can be avoided by dismissing the error dialog and
  refreshing the page.

## Links

* [IBM Db2 Developer Community Edition](https://www.ibm.com/us-en/marketplace/ibm-db2-direct-and-developer-editions) software
* [IBM Watson Studio](https://dataplatform.cloud.ibm.com/docs/content/getting-started/welcome-main.html) documentation
* [IBM Secure Gateway](https://cloud.ibm.com/docs/services/SecureGateway/index.html) documentation
* [Docker](https://docs.docker.com/) documentation
* [IBM Db2](https://www.ibm.com/support/knowledgecenter/SSEPGG_10.5.0/com.ibm.db2.luw.kc.doc/welcome.html) documentation
* Related code pattern: [Continuous learning on Db2](https://github.com/IBM/watson-continuous-learning-on-db2)
* Related video: [Continuous Learning on Watson Data Platform](https://www.youtube.com/watch?v=HCVxGMd1RiQ)

## Learn more

* **Artificial Intelligence Code Patterns**: Enjoyed this Code Pattern? Check out our other [AI Code Patterns](https://developer.ibm.com/technologies/artificial-intelligence/).
* **AI and Data Code Pattern Playlist**: Bookmark our [playlist](https://www.youtube.com/playlist?list=PLzUbsvIyrNfknNewObx5N7uGZ5FKH0Fde) with all of our Code Pattern videos
* **With Watson**: Want to take your Watson app to the next level? Looking to utilize Watson Brand assets? [Join the With Watson program](https://www.ibm.com/watson/with-watson/) to leverage exclusive brand, marketing, and tech resources to amplify and accelerate your Watson embedded commercial solution.

## License

This code pattern is licensed under the Apache Software License, Version 2.  Separate third party code objects invoked within this code pattern are licensed by their respective providers pursuant to their own separate licenses. Contributions are subject to the [Developer Certificate of Origin, Version 1.1 (DCO)](https://developercertificate.org/) and the [Apache Software License, Version 2](https://www.apache.org/licenses/LICENSE-2.0.txt).

[Apache Software License (ASL) FAQ](https://www.apache.org/foundation/license-faq.html#WhatDoesItMEAN)
