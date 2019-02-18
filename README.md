[![Build Status](https://travis-ci.org/ing-bank/airlock.svg?branch=master)](https://travis-ci.org/ing-bank/airlock)
[![codecov.io](http://codecov.io/github/ing-bank/airlock/coverage.svg?branch=master)](https://codecov.io/gh/ing-bank/airlock?branch=master)
[![](https://images.microbadger.com/badges/image/wbaa/airlock:latest.svg)](https://hub.docker.com/r/wbaa/airlock/tags/)

![](./docs/img/airlock.jpg)

*A security layer between s3 user (eg. application using aws sdk) and s3 backend (eg. ceph RadosGW).*

# What do you need

To get started with Airlock you only need a few applications set up:

- Docker
- AWS CLI
- [Optional for coding] SBT

We've added a small description on how to setup the AWS CLI [here](#setting-up-aws-cli).

# How to run
1. To test the proxy (both live and integration tests), we need all dependencies to be running. For this we use a `docker-compose.yml` which defines all dependencies, run it using:

        docker-compose up

2. Before we can run our proxy, we have to specify the configuration for Apache Ranger. Ranger can be configured by creating a file called `ranger-s3-security.xml` on the classpath.
    There are 2 places you can put it:
    
    1. `REPO_ROOTDIR/src/main/resources/ranger-s3-security.xml`
    2. `/etc/airlock/ranger-s3-security.xml`
    
    An example of this file can be found [here](./src/it/resources/ranger-s3-security.xml). 
    No modification to this is needed if you run this project with the accompanying docker containers.

3. When all is running, we can start the proxy:

        sbt run

> for windows docker runs on different it so you need to:
> set environmental variables:
> * AIRLOCK_STS_HOST
> * AIRLOCK_STORAGE_S3_HOST
> * AIRLOCK_KEYCLOAK_TOKEN_URL
> * change AIRLOCK_KEYCLOAK_URL in the docker-compose.yml
> * change the ranger.plugin.s3.policy.rest.url in ranger-s3-security.xml


## Proxy as docker image

When proxy is started as docker image, the ranger-s3-security.xml file can be added in the following way:

        docker run -d -v /host/dir/with/xmls:/etc/airlock -p 8010:8010 airlock

# Getting Started

> This guide assumes you're using the default docker containers provided, see: [How to run](#how-to-run)

Now you've got everything running, you may wonder: what now? This section we'll describe a basic flow on how to use 
Airlock to perform operations in S3. You may refer to the [What is Airlock?](./docs/What_is_airlock.md) document
before diving in here. That will introduce you to the various components used.

1. Authorise with keycloak to request a `keycloak token`:

        curl -s \
             -d 'client_id=sts-airlock' \
             -d 'username=testuser' \
             -d 'password=password' \
             -d 'grant_type=password' \
             'http://localhost:8080/auth/realms/auth-airlock/protocol/openid-connect/token'

    Search for the field `access_token` which contains your token.
    
2. Retrieve your short term `session credentials` from the STS service:

        aws sts get-session-token --endpoint-url http://localhost:12345 --token-code YOUR_KEYCLOAK_TOKEN_HERE
   
   This should give you an `accessKeyId`, `secretAccessKey`, `sessionToken` and `expirationDate`.
   
3. Setup your local environment to use the credentials received from STS. You can do this in 2 ways.

    1. Set them in your environment variables:
        
            export AWS_ACCESS_KEY_ID=YOUR_ACCESSKEY
            export AWS_SECRET_ACCESS_KEY=YOUR_SECRETKEY
            export AWS_SESSION_TOKEN=YOUR_SESSIONTOKEN
            
    2. Set them in your `~/.aws/config` file. See the [Setting up AWS CLI](#setting-up-aws-cli) guide on how to do this.
   
    > NOTE: This session expires at the expiration date specified by the STS service. You'll need to repeat these steps
    > everytime your session expires.
 
4. Technically you're now able to use the aws cli to perform any commands through Airlock to S3. Airlock automatically
   creates the user on Ceph for you. Since the authorisation is completely handled by ranger, authorization in Ceph
   should be removed to avoid conflicts. For this reason, ceph users need to have admin rights, granting them full
   control over all buckets. Airlock creates new users with `system` roles automatically, but the Airlock NPA must be
   manually configured as `system` user to be able to grant admin rights to other users:
   
            docker-compose exec ceph radosgw-admin user modify --uid ceph-admin --system
            
5. Go nuts with the default bucket called `demobucket` that exists on Ceph already:

        aws s3api list-objects --bucket demobucket
        aws s3api put-object --bucket demobucket --key SOME_FILE
        
   **!BOOM!** What happened?!
   
   Well, your policy in Ranger only allows you to read objects from the `demobucket`. So we'll need to allow a write as
   well. 
   
   1. Go to Ranger on [http://localhost:6080](http://localhost:6080) and login with `admin:admin`. 
   2. Go to the `testservice` under the S3 header.
   3. Edit the one existing policy. You'll have to allow the `testuser` to write, but also don't forget to remove the 
   deny condition!
   4. Save the policy at the bottom of the page.
   
   Now it'll take maximum 30 seconds for this policy to sync to the Proxy. Then you should be able to retry:
   
        aws s3api put-object --bucket demobucket --key SOME_FILE
        aws s3api list-objects --bucket demobucket
        aws s3api get-object --bucket demobucket --key SOME_FILE SOME_TARGET_FILE

# Difference between the proxy and Ceph

1. Ceph allows only list all your own buckets. We need to see all buckets by all users so the functionality is modified.

But the functionality is separated in the class 
[ProxyServiceWithListAllBuckets](https://github.com/ing-bank/airlock/blob/master/src/main/scala/com/ing/wbaa/airlock/proxy/api/ProxyServiceWithListAllBuckets.scala) 
so if you want to have standard behaviour use 
the [ProxyService](https://github.com/ing-bank/airlock/blob/master/src/main/scala/com/ing/wbaa/airlock/proxy/api/ProxyService.scala) 
in [AirlockS3Proxy](https://github.com/ing-bank/airlock/blob/master/src/main/scala/com/ing/wbaa/airlock/proxy/AirlockS3Proxy.scala)

# Verified AWS clients

We've currently verified that the following set of AWS clients work with Airlock:

- CLI (`s3` and `s3api`) using region `us-east-1`
- Java SDK (using signerType: `S3SignerType`)

Other options may work but haven't been checked yet by us. There are known limitiations for other signer types within the Java SDK.

# Architecture
![alt text](./docs/img/architecture.png)

Dependencies:
* [Keycloak](https://www.keycloak.org/) for MFA authentication of users.
* [STS Service](https://github.com/ing-bank/airlock-sts) to provide authentication and short term access to resources on S3.
* STS persistence storage to maintain the user and session tokens issued. Current implementation uses [MariaDB](https://mariadb.org).
Information regarding the tables database and the tables to be created in MariaDB can be found [here](https://github.com/ing-bank/airlock-dev-mariadb/blob/master/database/airlockdb.sql). 
* [Ranger](https://ranger.apache.org/) to manage authorisation to resources on S3.
The Apache Ranger docker images are created from this repo: https://github.com/ing-bank/airlock-dev-apache-ranger.git
* S3 Backend (Current setup contains Ceph image with RadosGW).

A more in-depth discussion of the architecture and interaction of various components can be found here: [What is Airlock?](docs/What_is_airlock.md)


# Docker Ceph settings

In order to enable debug logging on Ceph RadosGW:

1. Edit  /etc/ceph/ceph.conf and add following lines, under [global] section
```
debug rgw = 20
debug civetweb = 20
```

2. Restart rgw process (either docker stop <ceph/daemon rgw> or whole ceph/demo)

# Lineage to Atlas

Currently it is possible to create lineage based on incoming request to proxy server. It is however disabled by
default (preview feature). To enable lineage shipment to Atlas, following setting has to be added to application.conf:

```
airlock {
     atlas {
        enabled = true
     }
}
``` 

As alternative environment value `AIRLOCK_ATLAS_ENABLED` should be set to true. 

Lineage is done according following model
 
![alt text](./docs/img/atlas_model.jpg)

To check lineage that has been created, login to Atlas web UI console, [default url](http://localhost:21000) with
admin user and password 

# Events Notification

Airlock can send event notification to message queue based on user requests, in [AWS format](https://docs.aws.amazon.com/AmazonS3/latest/dev/NotificationHowTo.html).

Currently, two types are emitted:

- s3:ObjectCreated:*
- s3:ObjectRemoved:*

In order to enable update application.conf, mainly:

```
        enabled = ${?AIRLOCK_KAFKA_ENABLED}
        bootstrapServers = ${?AIRLOCK_KAFKA_BOOTSTRAP_SERVERS}
        createTopic = ${?AIRLOCK_KAFKA_CREATE_TOPIC}
        deleteTopic = ${?AIRLOCK_KAFKA_DELETE_TOPIC}
```

# Setting Up AWS CLI

It is possible to set up the AWS command-line tools for working with Ceph RadosGW and Airlock. Following are instructions
to set this up using `virtualenv_wrapper` or [Anaconda](https://www.anaconda.com/).

1. Create an environment for this work:

    a. **virtualenv_wrapper**

       % mkvirtualenv -p python3 airlock
       
    b. **Anaconda**

       % conda create -n airlock python=3
       % conda activate airlock

2. Install the AWS command-line tools and the endpoint plugin:

       % pip install awscli awscli-plugin-endpoint

3. Configure profiles and credentials for working with Airlock or the RadosGW directly (more info can be found in the
[aws documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-config-files.html)):

       % mkdir -p ~/.aws
       
       % cat >> ~/.aws/credentials << EOF
       [radosgw]
       aws_access_key_id = accesskey
       aws_secret_access_key = secretkey

       [airlock]
       aws_access_key_id = YOUR_ACCESSKEY
       aws_secret_access_key = YOUR_SECRETKEY
       aws_session_token = YOUR_SESSIONTOKEN
       EOF
       
       % cat >> ~/.aws/config << EOF
       [plugins]
       endpoint = awscli_plugin_endpoint

       [profile airlock]
       output = json
       region = us-east-1
       s3 =
           endpoint_url = http://localhost:8987/
       s3api =
           endpoint_url = http://localhost:8987/
       sts =
           endpoint_url = http://localhost:12345/

       [profile radosgw]
       output = json
       region = us-east-1
       s3 =
           endpoint_url = http://localhost:8010/
       s3api =
           endpoint_url = http://localhost:8010/
       EOF

> Note: for now, we default to us-east-1 AWS region for signature verification

4. Configure the default profile and reactivate the virtual environment:

    a. **virtualenv_wrapper**
    
       % cat >> ${WORKON_HOME:-$HOME/.virtualenvs}/airlock/bin/postactivate << EOF
       AWS_DEFAULT_PROFILE=airlock
       export AWS_DEFAULT_PROFILE
       EOF
       
       % cat >> ${WORKON_HOME:-$HOME/.virtualenvs}/airlock/bin/predeactivate << EOF
       unset AWS_DEFAULT_PROFILE
       EOF
       
       % deactivate
       
       % workon airlock

    b. **Anaconda**
    
       % cat >> /YOUR_CONDA_HOME/envs/airlock/etc/conda/deactivate.d/aws.sh << EOF
       unset AWS_DEFAULT_PROFILE
       EOF
       
       % cat >> /YOUR_CONDA_HOME/envs/airlock/etc/conda/activate.d/aws.sh << EOF
       AWS_DEFAULT_PROFILE=airlock
       export AWS_DEFAULT_PROFILE
       EOF
       
       % source deactivate
       
       % source activate airlock

By default S3 and STS commands will now be issued against the proxy service. For example:

    % aws s3 ls

Commands can also be issued against the underlying RadosGW service:

    % aws --profile radosgw s3 ls

The default profile can also be switched by modifying the `AWS_DEFAULT_PROFILE` environment variable.

# Security environment

For kerberos environment (e.g connecting to ranger) you need to provide keytab file and principle name.

```bash
AIRLOCK_KERBEROS_KEYTAB: "keytab_full_path"
AIRLOCK_KERBEROS_PRINCIPAL: "user"
```

 
