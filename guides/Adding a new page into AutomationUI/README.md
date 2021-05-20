# Installing a new page into an existing IAF instance
This guide will show you how to install a new page into an IAF instance running on Openshift and add a hamburger menu item in the left hand sidebar to access it.

## Prerequisites
You will need IBM Automation Foundation (IAF) installed, running in an Openshift cluster on a Cloud Provider such as IBM Cloud. Follow this [guide](https://www.ibm.com/docs/en/automationfoundation/1.0_ent?topic=installing-using-ui) to get to this point if you have not already done so.

Make sure you have this repo cloned locally, as we will need to build an image using the files and have a copy of the YAML for our deployment available to the command line.

> If you are using a custom storage configuration, you will need to create a `AutomationBase` operand with details of your configuration to tell `AutomationUI` what it needs to use before moving on. Failure to do so may cause issues with the installation of the `Cartridge` operand, specifically bringing up the `AutomationUI` pods.

## 1. Install `Cartridge` operand
We must first create a `Cartridge` operand. This is responsible for creating the AutomationUI (Zen) and AutomationUI pods and allowing us to route into our cluster and access the UI.

Unless you have already made one as described above, this step will also create a default `AutomationBase` operand with some pre-filled values, as it's a requirement before `Cartridge` can instantiate.

### Cartridge operand install video

https://user-images.githubusercontent.com/24469095/112845372-b3c0ff80-909c-11eb-852d-0d03bbbbba48.mov

The operand is installed when all the status' are marked as True, as seen here:
<img width="1278" alt="Screenshot 2021-03-29 at 2 37 35 PM" src="https://user-images.githubusercontent.com/24469095/112845493-d2bf9180-909c-11eb-9a42-0a1ca779d8bf.png">


## 2. Access the `AutomationUI` in a browser.
Now we have installed `AutomationUI` and its requirements, we can navigate to the UI in our browser and login. To do so, we need to know the route of our UI instance. To get this route:

A) Login to your cluster from the command line using the credentials found in the `Copy login command` link. This is located at the top right corner of the Openshift console under the logged in users name.

B) Ensure your in the correct project. Enter `oc project` to see the name of the project you're currently working in. You will need to be in the same project you installed IAF into. To change the current active project, just enter `oc project <IAF_PROJECT_NAME>` substituting your project name between the angled brackets.

C) Show the route using `oc get route cpd`. Copy the URL under the HOST/PORT column from the terminal and paste it into a web browser.

D) There are two ways to login by default, either with admin credentials stored in a secret in the `ibm-common-services` namespace or using Openshift's `kubeadmin` credentials.

To get the secret's credentials and login as admin, you can run the following command and paste the value into the web browsers password field (username will be admin).

```
oc get secrets -n ibm-common-services platform-auth-idp-credentials -ojsonpath='{.data.admin_password}' | base64 --decode && echo ""
```

> This command gets the `platform-auth-idp-credentials` secret, inside the `ibm-common-services` namespace and decodes this data. It then prints it to the console.

## 3. Build and push the Docker image

We now need to build and push the Docker image that will become the basis of our `Deployment`.

You will need to decide on somewhere to push the image too. Openshift clusters can be configured to act as their own image registries, but setting that up is outside the scope of this walkthrough. See [Internal Registry Overview](https://docs.openshift.com/container-platform/3.11/install_config/registry/index.html) for more information.

### 3.1 Build the Docker image

From the root of the project, and with Docker running locally, run the following with your route instead of `ROUTE_TO_REGISTRY`:

```
docker build -t ROUTE_TO_REGISTRY/newcontent:v1 .
```

### 3.2 Push to a Docker registry

Simply run the following, again with your route instead of ``IMAGE_URL_REGISTRY`` :

```
docker push IMAGE_URL_REGISTRY/newcontent:v1
```

The docker image has now been pushed up to your registry. The tag you gave the image needs to be added to the `combined_deployment.yaml`, so remember where you pushed the image for the next step.

## 4. Create the new objects on your cluster

You'll need to create a new series of objects in your cluster:

* A `Deployment`, which contains the code for the new page, runs it in a container and exposes a port.
* A `Service` that exposes the new deplooyment and helps route traffc to it.
* Proxy `ConfigMap` to help proxy the traffic from the AutomationUI dashboard to your new service.
* Navigation `ConfigMap` to adds a menu item to access the new page.

We also need to update the image source URL so Openshift knows where to pull your image from. Change <strong>line 16</strong> to use the full URL (tag) of the image you created and pushed in [Step 3](#Step3). The line you must change looks like this:

```
image: IMAGE_URL_REGISTRY/newcontent:v1
```


We also need to make a small change on <strong>line 49</strong>, to point to your clusters IAF project. Edit this line and replace `PROJECT_NAMESPACE` with the project you chose to install IAF too (same one we used earlier).

```
proxy_pass http://newcontent-service.PROJECT_NAMESPACE.svc:80/;
```

After you have updated the URL for your image, we can apply the YAML to your cluster with the following command:

```
oc apply -f combined_deployment.yaml
```

You'll see the following lines printed to show the objects were created on your cluster:

```
deployment.apps/newcontent-deployment created
service/newcontent-service created
configmap/newcontent-proxy created
configmap/newcontent-nav-ext created
```

## 5. Load into your new page

Just refresh the dashboard and you should now see a new item in the lefthand sidebar (to open use the hamburger menu at the top right). Click this and it should open a page displaying "Hello world!". You'll see the black Header along the top of the page with your logo, along with access to the hamburger menu.

That's it! Congratulations on creating a new page and adding it to IAF, and making it accessible from within the AutomationUI environment.

## How AutomationUI is hooking into our new microservice's UI

The docker image we use as the base for our new page is a simple Express server that serves an `index.html` file, shown below:

```
<!doctype html>
<html style="margin-top:3rem;">
  <head>
    <link rel="stylesheet" href="/zen/v2/nav/css/nav-carbon10.min.css"/>
    <script src="/zen/v2/nav/js/nav.min.js"></script>
    <title>Hello World page</title>
  </head>
  <body>
    <p>Hello world!</p>
  </body>
</html>
```

This file is very simple, and the AutomationUI hooks found in the `<link>` and `<script>` tags are how we get the two UI's showing together. They bring the Header and Navigation menu into our miroservice, in this case, the `index.html` page.

> You may note a small amount of styling for the HTML. This is to stop our content and the Header provided by AutomationUI overlapping when the page is rendered. A `margin-top: 3rem` is recommended.

## Troubleshooting

* If you applied the YAML and find no changes to the nav menu, or get a 400 error code, the cache and cookies created by your browser may be the cause of the problem. Try hard refreshing, clearing the cache and cookies or just restarting your browser.

* Helpful pods for assessing issues with routing and new content are the two `ibm-nginx` pods and the `zen-watcher` pod. The logs from them can provide useful details on issues, and entering the `ibm-nginx` pod and checking for NGINX configurations can help when looking into proxy issues.

