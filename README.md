# Crossplane CLI

## Installation

Get the commands on the PATH so `kubectl` can pick them up:
```
curl -o /usr/local/bin/kubectl-crossplane-stack-build https://raw.githubusercontent.com/suskin/crossplane-stack-cli/master/bin/kubectl-crossplane-stack-build -s >/dev/null
curl -o /usr/local/bin/kubectl-crossplane-stack-init https://raw.githubusercontent.com/suskin/crossplane-stack-cli/master/bin/kubectl-crossplane-stack-init -s >/dev/null
curl -o /usr/local/bin/kubectl-crossplane-stack-publish https://raw.githubusercontent.com/suskin/crossplane-stack-cli/master/bin/kubectl-crossplane-stack-publish -s >/dev/null
chmod +x /usr/local/bin/kubectl-crossplane-stack-*
```

## Usage

```
kubectl crossplane stack init 'myname/mysubname'
kubectl crossplane stack build
kubectl crossplane stack publish
```

# Quick Start: Stacks

This guide will show you the basics for using the Stacks CLI to create
and develop a stack with a basic controller that responds to a certain
type of object.

Excluding installing prerequisites, it'll take about 5 minutes!

## Install prerequisites

The workflow for working with a stack assumes that you have a few
things installed:

* [go](https://golang.org/doc/install)
* [docker](https://www.docker.com/products/docker-desktop)
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* [kubebuilder, version 2][kubebuilder quick start]
* [kustomize](https://github.com/kubernetes-sigs/kustomize/blob/master/docs/INSTALL.md)
  (this is an optional dependency of kubebuilder and is not required)
* [GNU make](https://www.gnu.org/software/make/)
* bash
* [crossplane](https://github.com/crossplaneio/crossplane),
  or your `kubectl` should be set up to talk to a crossplane
  control cluster

`make` and `bash` are probably already installed if you're using a
Unix-like environment or MacOS.

### Install Kubebuilder
Instructions to install kubebuilder come from the [kubebuilder quick
start][kubebuilder quick start], but are reproduced here for
convenience.

```
os=$(go env GOOS)
arch=$(go env GOARCH)

# download kubebuilder and extract it to tmp
curl -sL https://go.kubebuilder.io/dl/2.0.0/${os}/${arch} | tar -xz -C /tmp/

# move to a long-term location and put it on your path
# (you'll need to set the KUBEBUILDER_ASSETS env var if you put it somewhere else)
sudo mv /tmp/kubebuilder_2.0.0_${os}_${arch} /usr/local/kubebuilder
export PATH=$PATH:/usr/local/kubebuilder/bin
```

## Install CLI
Installing the `kubectl` plugins is pretty simple.

Copy the plugins to somewhere on your `PATH`. If you have
`/usr/local/bin` on your `PATH`, you can do it like this:

```
curl -o /usr/local/bin/kubectl-crossplane-stack-init https://raw.githubusercontent.com/suskin/crossplane-stack-cli/master/bin/kubectl-crossplane-stack-init -s >/dev/null
curl -o /usr/local/bin/kubectl-crossplane-stack-build https://raw.githubusercontent.com/suskin/crossplane-stack-cli/master/bin/kubectl-crossplane-stack-build -s >/dev/null
curl -o /usr/local/bin/kubectl-crossplane-stack-publish https://raw.githubusercontent.com/suskin/crossplane-stack-cli/master/bin/kubectl-crossplane-stack-publish -s >/dev/null
chmod +x /usr/local/bin/kubectl-crossplane-stack-*
```


## Init project folder
First you'll need a folder to work with. Initializing it probably looks
something like the following:

```
cd helloworld
git init
```

## Init Kubebuilder project
Once the project is initialized, we'll set it up as a kubebuilder
project.

### Create a kubebuilder v2 project
If your project is within the `GOPATH`, you can do:

```
GO111MODULE=on kubebuilder init --domain helloworld.stacks.crossplane.io
```

If your project is not within the `GOPATH`, you need to do an extra step first. See the
[kubebuilder quick start guide][kubebuilder quick start] for more details.

The instructions in this guide will use `GO111MODULE=on` in front of all
of the `kubebuilder` commands, but it could also be set once at the
beginning with `export GO111MODULE=on`, or set in other ways.

### Create an API
Next, create an API using `kubebuilder`:

```
$ GO111MODULE=on kubebuilder create api --group samples --version v1alpha1 --kind HelloWorld
> Create Resource [y/n]
$ y
> Create Controller [y/n]
$ y
```

## Initialize the stack project
Once the project is initialized as a kubebuilder project and the plugins
are installed, we can initialize the project as a stack project using
the `crossplane stack init` command.

From within the project directory:

```
kubectl crossplane stack init 'crossplane-examples/hello-world'
```

## Set up the Hello World
Now we'll add some custom functionality to our controller.

Add a "Hello World" log line to the kubebuilder controller, which should
be in the `controllers/` directory:

```
	// your logic here
  r.Log.V(0).Info("Hello World!", "instance", req.NamespacedName)
```

## Build kubebuilder stuff
First, we'll want to build the binaries and yamls in the regular way.

```
GO111MODULE=on make manager manifests
```

## Validate the stack locally

During development, we'll often be building and testing our stack
locally. The local validation section shows us how we would do that.

### Build stack and publish locally
Once the (kubebuilder) project is built in the normal way, we want to
bundle it all into a stack and publish the image locally:

```
kubectl crossplane stack build local-build
```

### Install stack locally
When the stack is built, the next step is to install it into our
Crossplane:

```
kubectl crossplane stack build stack-install
```

### Create an object for the stack to manage
Once the stack is installed into our Crossplane, we can use one of the
sample objects to create an object that the stack will respond to.

```
kubectl apply -f config/samples/*.yaml
```

### Check the stack's output
Once the object has been created, we expect the controller to do
something in response! In our case, we expect it to log a message.

We can check whether the controller is running and logging messages
using a `kubectl get` to identify the pod running the controller,
followed by a `kubectl logs` to read the logs of that pod:

```
$ kubectl get pods -A | grep hello-world | grep -v Completed
default             crossplane-examples-hello-world-65d5c59976-vzppd   1/1     Running     0          91s
$ kubectl logs crossplane-examples-hello-world-65d5c59976-vzppd | grep 'Hello World'
2019-08-28T23:37:51.795Z	INFO	controllers.HelloWorld	Hello World!	{"instance": "default/helloworld-sample"}
```

That's it! We've finished writing, building, and locally validating a
stack!

## Bonus: How to build for external publishing
After we finish developing a stack locally, we may want to publish it to
an external registry. This section shows the commands to do that.

### Build stack
To build the stack, we use the `build` subcommand once the binaries and
generated yamls have been built:

```
kubectl crossplane stack build
```

### Run publish
Once the stack is built, we can use the `publish` subcommand to publish
it to the registry:

```
kubectl crossplane stack publish
```

### Install
Installing the stack can be done with the sample which was generated for
us by the `init` that we ran earlier.

```
kubectl apply -f config/stack/samples/install.stack.yaml
```


[kubebuilder quick start]: https://book.kubebuilder.io/quick-start.html
