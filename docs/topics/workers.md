# Adding custom libraries to a Brigade worker

A worker is the component in Brigade that executes a `brigade.js`. Brigade ships
with a general purpose worker focused on running jobs. This generic worker
exposes a host of Node.js libraries to `brigade.js` files, as well as the
`brigadier` Brigade library.

Sometimes, though, it is desirable to include additional libraries -- perhaps even
custom libraries -- to your workers. 
There are two ways of adding custom dependencies to a Brigade worker:

- by adding a `brigade.json` file in your repository (similar to a `package.json` file) 
that contains the dependencies and that are specific on every Brigade project

- by creating a custom Docker image for the worker that already contains the dependencies 
and which will be used for all Brigade projects

This guide explains both processes and describes the cases where each method is most suitable.

**Note:** This area of Brigade is still evolving. If you have ideas for improving
it, feel free to [file an issue](https://github.com/Azure/brigade/issues) explaining
your idea.

## Add custom dependencies using a `brigade.json` file

If you need different dependencies for every Brigade project, this can be easily achieved 
using a `brigade.json` file placed side-by-side the `brigade.js` file. This file contains 
the dependency name and version, and has the following structure:

```
{
    "dependencies": {
        "is-thirteen": "2.0.0"
    }
}
```
Before starting to execute the `brigade.js` script, the worker will install the  
dependencies using `yarn`, adding them to the `node_modules` folder.

Then, in the `brigade.js` file, the new dependency can be used just like any 
other NodeJS dependency:

```
const { events } = require("brigadier")
const is = require("is-thirteen");

events.on("exec", function (e, p) {
    console.log("is 13 thirteen? " + is(13).thirteen());
})
```

Now if we run a build for this project, we see the `is-thirteen` dependency added, 
as well as the console log resulted from using it:

```
$ brig run brigade-86959b08a89af5b6b83f0ace6d9030f1fdca7ed8ea0a296e27d72e
Event created. Waiting for worker pod named "brigade-worker-01cexrvrs08shcev26961cwd6n".
Started build 01cexrvrs08shcev26961cwd6n as "brigade-worker-01cexrvrs08shcev26961cwd6n"
installing is-thirteen@2.0.0
prestart: src/brigade.js written
[brigade] brigade-worker version: 0.14.0
[brigade:k8s] Creating PVC named brigade-worker-01cexrvrs08shcev26961cwd6n
is 13 thirteen? true
[brigade:app] after: default event handler fired
[brigade:app] beforeExit(2): destroying storage
[brigade:k8s] Destroying PVC named brigade-worker-01cexrvrs08shcev26961cwd6n
```

Notes:

- the dependencies should point the exact version (and not use the tilde ~ and caret ^ to indicate semver compatible versions)

- when adding a custom dependency using `brigade.json`, `yarn` will add it side-by-side with [the worker's 
dependencies](../../brigade-worker/package.json) - this means that the process will fail if a dependency that conflicts with one of the 
worker's dependency is added. However, already existing dependencies (such as `@kubernetes/client-node`, `ulid` or `chai`) 
can be used from `brigade.js` without adding them to `brigade.json`. 

However, the only issue is trying to add a different version of an already existing dependency of the worker.

- as the Brigade worker is capable of modifying the state of the cluster, be mindful 
of any external dependencies added through `brigade.json`

- for now, the only part of `brigade.json` used is the `dependencies` object - it means the rest of the file 
can be used to pass additional information to `brigade.js`, such as environment data - however, as the `brigade.json` 
file is passed in the source control repository, information passed there will be available to anyone with access to the repository.

- all dependencies added in `brigade.json` are dynamically installed on every Brigade build - this means if the dependencies added 
are large, and the build frequency is high for a particular project, it might make sense to make a pre-built Docker image that 
already contains the dependencies, as described in the sections below.

## Workers and Docker Images

The Brigade worker (`brigade-worker`) is captured in a Docker image, and that
image is then executed in a container on your cluster. Augmenting the worker,
then, is done by creating a custom Docker image and then configuring Brigade
to use that image instead of the default `brigade-worker` image.

Next we will see how to quickly create a custom worker by creating a new
Docker image based on the base image.

## Creating a Custom Worker

As we saw above, workers are Docker images. And the default worker is a Docker
image that contains the Brigade worker runtime, which can read and execute
`brigade.js` files.

At its core, the Brigade worker is just a Node.js application. That means that
we can use the usual array of Node.js tools and libraries. Here, we'll load an
extra library from NPM so that it is available inside of our `brigade.js`
scripts.

Since the main worker is already tooled to do the main processing, the easiest
way to add your own libraries is to start with the existing worker and add to
it. Docker makes this convenient.

Say we want to provide an XML parser to our Brigade scripts. We can do that
using a `Dockerfile`:

```Dockerfile
FROM deis/brigade-worker:latest

RUN yarn add xml-simple
```

The above will begin with the default Brigade worker and simply add the `xml-simple`
library. We can build this into an image and then push it to our Docker registry
like this:

```console
$ docker build -t myregistry/myworker:latest .
$ docker push myregistry/myworker:latest
```

> IMPORTANT: Make sure you replace `myregistry` and `myworker` with your own
> account and image names.

**Tip:** If you are running a local Kubernetes cluster with Docker or Minikube,
you do not need to push the image. Just configure your Docker client
to point to the same Docker daemon that your Kubernetes cluster is using. (With
Minikube, you do this by running `eval $(minikube docker-env)`.)

Now that we have our image pushed to a usable location, we can configure Brigade
to use this new image.

## Configuring Brigade to Use Your New Worker

As of Brigade v0.10.0, worker images can be configured _globally_. Individual
projects can choose to override the global setting.

To set the version globally, you should override the following values in your
`brigade/brigade` chart:

```yaml
# worker is the JavaScript worker. These are created on demand by the controller.
worker:
  registry: myregistry
  name: myworker
  tag: latest
  #pullPolicy: IfNotPresent # Set this to Always if you are testing and using
  #                           an upstream registry like Dockerhub or ACR
```

You can then use `helm upgrade` to load those new values to Brigade.

### Project Overrides

To configure the worker image per-project, you can set up a custom `worker` section
in your `brigade/brigade-project` values file:

myvalues.yaml
```yaml
# OPTIONAL: Project-specific worker settings which takes precedence over brigade-wide settings
# Useful when you want a specific brigade-worker image for running brigade.js of this project.
worker:
  registry: myregistry
  name: myworker
  tag: latest
  pullPolicy: IfNotPresent
```

The above will set the worker image to `myregistry/myworker:latest`. Now you
can create a project that uses that worker by running `helm install -f myvalues.yaml brigade/brigade-project`

## Using Your New Image

Once you have set the Docker image (above), your new Brigade workers will
automatically switch to using this new image.

Assuming you have configured your project (as explained above) and
your Kubernetes cluster can see the Docker registry that you pushed your image to,
you can now simply assume that you are using your new custom image. So now
we can import our new `simple-xml` library:

[brigade.js](examples/workers/brigade.js)
```javascript
const { events } = require("brigadier");
const XML = require("simple-xml");

events.on("exec", () => {
  var o = XML.parse("<say><to>world</to></say>")
  console.log(`Saying hello to ${o.say.to}`);
});

```

Running the above with `brig run -f brigade.js my/project` (where `my/project`
is some project you have already created) should result in a successful run.

Here is an example:
```console
$ brig run -f brigade.js deis/empty-testbed
Started build 01c7kmserwyc5y05rrhpvnp5m0 as "brigade-worker-01c7kmserwyc5y05rrhpvnp5m0-master"
prestart: src/brigade.js written
[brigade] brigade-worker version: 0.10.0
[brigade:k8s] Creating PVC named brigade-worker-01c7kmserwyc5y05rrhpvnp5m0-master
Saying hello to world
[brigade:app] after: default event fired
[brigade:app] beforeExit(2): destroying storage
[brigade:k8s] Destroying PVC named brigade-worker-01c7kmserwyc5y05rrhpvnp5m0-master
```

You can see that `Saying hello to ${o.say.to}` rendered correctly
as `Saying hello to world`.

## Adding a Custom (Non-NPM) Library

Sometimes it is useful to encapsulate commonly used Brigade code into a library
that can be shared between projects internally. While the NPM model above is
easier to manage over the longer term, there is a simple method for loading
custom code into an image. This section illustrates that method.

Here is a small library that adds an `alpineJob()` helper function:

[mylib.js](examples/workers/mylib.js)
```javascript
const {Job} = require("./brigadier");

exports.alpineJob = function(name) {
  j = new Job(name, "alpine:3.7", ["echo hello"])
  return j
}
```

**Note:** Because we are loading our code directly into Brigade, we import
`./brigadier`, not `brigadier`. This may change in the future.

We can build this file into our Dockerfile by copying it into the image:

```
FROM deis/brigade-worker:latest

RUN yarn add xml-simple
COPY mylib.js /home/src/dist
```

And now we can build the above:

```console
$ docker build -t myregistry/myworker:latest .
$ docker push myregistry/myworker:latest
```

Assuming you have configured Brigade to use your `myworker` image (explained
above in "Configuring Brigade to Use Your New Worker"), you can begin using the
library:

```javascript
const { events } = require("brigadier");
const XML = require("xml-simple");
const { alpineJob } = require("./mylib");

events.on("exec", () => {
  XML.parse("<say><to>world</to></say>", (e, say) => {
    console.log(`Saying hello to ${say.to}`);
  })

  const alpine = alpineJob("myjob");
  alpine.run();
});

```

Now we've added a few new lines to the script. We import `alpineJob` from our
`./mylib` module at the top. Then, late in the script, we call `alpineJob` to
create a new job for us.

## Best Practices

As we have seen, it is easy to add new functionality to the Brigade worker. But
it is important to keep in mind that the Worker is intended to do one thing:
execute Brigade chains.

To that end, it is best to fight the temptation to put too much logic into the
Brigade worker. Where possible, use Jobs to perform specific tasks within their
own containers, and use workers to control the execution of a series of Jobs.

We strongly discourage attempting to turn a worker into a long-running server.
This violates the design assumptions of Brigade, and can result in unintended
side effects.
