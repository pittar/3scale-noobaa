# 3Scale with Noobaa for S3 Object Storage

## Preface

These instructions are for my own exploration and shouldn't be taken as a supported solution.  *Use at your own risk!*

This has been tested on:
* OpenShift Container Platform 4.3.5 (RHPDS)
* 3Scale 2.8 deployed with the official Operator (Red Hat Integration - 3Scale, not the "Community" one)

### Install Noobaa Operator

The easiest way to install the Noobaa operator and create a Noobaa instance is with the Noobaa cli tool.  You can install it on MacOS and Linux by [following the Noobaa Operator Github instructions](https://github.com/noobaa/noobaa-operator). 

Of course, you can also use the Noobaa operator from [OperatorHub](https://operatorhub.io/operator/noobaa-operator).

If you have installed the Noobaa cli:
1. Make sure you are logged into your cluster with the `oc` cli tool as a user with `cluster-admin`
2. Make sure you are in the project where you want Noobaa to be installd. For example: `oc new-project noobaa`
3. `$ noobaa install`

This should install the operator and instantiate Noobaa in your cluster.  You need to setup a region for your buckets.

1. Login to the Noobaa management console (noobaa core route, login with your OCP admin credentials).
2. From the left-nav, click on **Resources**, then click on `noobaa-default-backing-store`.
3. On the right side of the screen, click the **Assign Region** button.
4. Enter `noobaa` and enter.  You will now the list in "Cloud Resource Properties" lists **noobaa** as the region.

Of course, you can set the region to whatever you want, but you will need to adjust the instructions below accordingly.

### Creating your ObjectBucketClain

After Noobaa is installed and running, you will have a new `StorageClass` available in your cluster.

From the Admin console: `Admin -> Storage -> Object Bucket Claim -> New Bucket Claim`

Give your bucket a name and choose the Noobaa storage class and bucket class.  These should be the only options.

Click "Create".

In your project this will create:
* a new `ObjectBucketClaim` in your project.
* a new `Secret` with the same name as your bucket.
    * This also contains your `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY_ID`.  If you "Reveal Values" here you will see the actual values you will need for your 3Scale secret.
* a new `ConfigMap` with the same name as your bucket.  It has info on the bucket host and name.  We won't need any of it!

**Warning:** If you "Reveal Values" in the Object Bucket Claim, the "Access Key" and "Secret Key" are still base64 encoded.  If you use these values in your 3Scale secret, you'll need to decode them first!

### Creating Your S3 Config Secret for 3Scale

To create your configuration secret you will need the following info:
1: The public route URL for the `noobaa-endpoint` deployment.
2: The `AWS_ACCESS_KEY_ID` from the new bucket `Secret`
3: The `AWS_SECRET_ACCESS_KEY` from the new bucket `Secret`
4: The bucket name from the ConfigMap.  It starts with the name you chose, but it's got a long identifier attached.

With this info in mind, create a secret like this:
```
apiVersion: v1
kind: Secret
metadata:
  name: s3-auth
stringData:
  AWS_ACCESS_KEY_ID: <number 2 from list above>
  AWS_SECRET_ACCESS_KEY: <number 3 from list above>
  AWS_BUCKET: <number 4 from list above>
  AWS_REGION: noobaa <can be anything... just not empty>
  AWS_HOSTNAME: <number 1 from list above>
  AWS_PROTOCOL: HTTPS
  AWS_PATH_STYLE: 'true'
type: Opaque
```

An example:
```
apiVersion: v1
kind: Secret
metadata:
  name: s3-auth
stringData:
  AWS_ACCESS_KEY_ID: UzmNcyUuHtsBZeYKaCvD
  AWS_SECRET_ACCESS_KEY: 8FrIYgJqjY1sygBSl9umg8h1rcR4xWPZ+ePiTlxH
  AWS_BUCKET: 3scale-s3-2212ab03-6561-4588-ad07-73eb9ebd1754
  AWS_REGION: noobaa
  AWS_HOSTNAME: s3-3scale.apps.cluster-pitt-fde8.pitt-fde8.example.opentlc.com
  AWS_PROTOCOL: HTTPS
  AWS_PATH_STYLE: 'true'
type: Opaque
```

Next, create this secret in your 3Scale project:
```
oc apply -f s3-auth.yaml -n 3scale
```

### Create your APIManager Instance

First, make sure you have installed the `2.8` version of the official 3Scale operator in your project.

Next, create an `APIManager` Custom Resource like the one below.  Make sure you change the name if you like, and the wildcard URL to match your cluster.

```
apiVersion: apps.3scale.net/v1alpha1
kind: APIManager
metadata:
  name: my-apimanager
  namespace: 3scale
spec:
  wildcardDomain: apps.cluster-pitt-fde8.pitt-fde8.example.opentlc.com
  resourceRequirementsEnabled: false
  system:
    fileStorage:
      simpleStorageService:
        configurationSecretRef:
          name: s3-auth
```

Then apply it to your cluster in your 3Scale project:
```
oc apply -f apimanager.yaml -n 3scale
```

This should spin up a new 3Scale instance and use your Noobaa S3 bucket.  In the Noobaa interface, you should see a number of files in your bucket after `system-sidekiq` has fully initialized (this can take a few min even after it is reporting as ready).