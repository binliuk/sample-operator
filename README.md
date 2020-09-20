# Sample-operator
Example to create a sample operator using [operator-sdk](https://github.com/operator-framework/operator-sdk)

In this example we will create a sample-operator for deploying [sample application](https://github.com/binliuk/Examples/tree/master/Sample) based on spring-boot and camel. 

## Prerequisites.

1. Build the [sample App](https://github.com/binliuk/Examples/blob/master/Sample/README.md) image
2. Install [operator-sdk](https://github.com/operator-framework/operator-sdk/blob/master/doc/user/install-operator-sdk.md)
3. Install [golang](https://golang.org/dl/)
4. Access of Openshift cluster with `cluster-admin` permissions.
5. Openshift CLI(OC)


## Steps to create the sample-operator.

1. Create sample-operator project
~~~
$ mkdir -p $GOPATH/src/github.com/binliuk/  // remane to your account.
$ cd $GOPATH/src/github.com/binliuk/
$ export GO111MODULE=on
$ operator-sdk new sample-operator
$ cd sample-operator
$ go mod tidy // Install dependencies
~~~

2. Verify the directory structure cretaed. 
~~~
├── build
│   ├── bin
│   │   ├── entrypoint
│   │   └── user_setup
│   └── Dockerfile
├── cmd
│   └── manager
│       └── main.go
├── deploy
│   ├── operator.yaml
│   ├── role_binding.yaml
│   ├── role.yaml
│   └── service_account.yaml
├── go.mod
├── go.sum
├── pkg
│   ├── apis
│   │   └── apis.go
│   └── controller
│       └── controller.go
├── tools.go
└── version
    └── version.go
~~~
More info about SDK [project layout](https://github.com/operator-framework/operator-sdk/blob/master/doc/project_layout.md) doc.

2. Create Custom Resource Definition (CRD)
~~~
$ operator-sdk add api --api-version=binliuk.com/v1alpha1 --kind=Sample
~~~

3. Update the Smaple CR Spec and Status at `pkg/apis/binliuk/v1alpha1/sample_types.go`
~~~Go
type SampleSpec struct {
  Size        int32             `json:"size"`
  BodyValue   string            `json:"bodyvalue"`
  Image        string           `json:"image"`

}

// SampleStatus defines the observed state of Sample
type SampleStatus struct {
    Nodes []string `json:"nodes"`
}

~~~

Run the below command to update the generaed code after modifying *_types.go
~~~
$ operator-sdk generate k8s

//Generate the updated CRDs
$ operator-sdk generate crds
~~~

4. Create a new Controller to watch and reconcile Sample resource
~~~
$ operator-sdk add controller --api-version=binliuk.com/v1alpha1 --kind=Sample
~~~

5. Replace the default `pkg/controller/sample/sample_controller.go` with the [sample_controller.go](https://github.com/binliuk/sample-operator/blob/master/pkg/controller/sample/sample_controller.go) 


6. Build and push the Operator image to a reqistry.
~~~
$operator-sdk build default-route-openshift-image-registry.apps.osdtest1.h7z8.p2.openshiftapps.com/test/sample-operator:v0.0.1

//Verify the operator image cretaed locally
lbin@lbin-ThinkPad-W530:~/go/src/github.com/binliuk/sample-operator$ docker images
REPOSITORY                                                                                            TAG                 IMAGE ID            CREATED             SIZE
default-route-openshift-image-registry.apps.osdtest1.h7z8.p2.openshiftapps.com/test/sample-operator   v0.0.1              d25097cb8055        About an hour ago   185MB

// Push the image
$ docker login -u `oc whoami` -p `oc whoami -t` default-route-openshift-image-registry.apps.osdtest1.h7z8.p2.openshiftapps.com
$ docker push default-route-openshift-image-registry.apps.osdtest1.h7z8.p2.openshiftapps.com/test/sample-operator
~~~

7. Update the default `deploy/operator.yaml` with the correct image deatils
~~~GO
serviceAccountName: sample-operator
containers:
  - name: sample-operator
    # Replace this with the built image name
    image: image-registry.opernshift-image-registry.svc:5000/test/sample-operator:v0.0.1 
    command:
    - sample-operator
    imagePullPolicy: Always
~~~

## Steps to deploy the sample-operator on Openshift cluster

1. Create a new project
~~~
$ oc new-project sample-operator
~~~

2. Create the sample CRD
~~~
$ oc create -f deploy/crds/binliuk.com_samples_crd.yaml
~~~

3. Deploy the Operator along with set-up the RBAC
~~~
$ oc create -f deploy/service_account.yaml
$ oc create -f deploy/role.yaml
$ oc create -f deploy/role_binding.yaml
$ oc create -f deploy/operator.yaml
~~~

4. Verify the deployment and logs
~~~
$ oc get all
NAME                                  READY   STATUS    RESTARTS   AGE
pod/sample-operator-8f4667bd-zj5d2    1/1     Running   0          47m

NAME                              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/sample-operator-metrics   ClusterIP   172.30.82.158   <none>        8383/TCP,8686/TCP   47m

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/sample-operator   1/1     1            1           49m

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/sample-operator-8f4667bd     1         1         1       47m

NAME                                             IMAGE REPOSITORY                                                                                      TAGS     UPDATED
imagestream.image.openshift.io/sample-operator   default-route-openshift-image-registry.apps.osdtest1.h7z8.p2.openshiftapps.com/test/sample-operator   v0.0.1   53 minutes ago

~~~

5. Create the Sample Custom Resource(CR)
~~~
$ cat deploy/crds/binliuk.com_v1alpha1_sample_cr.yaml 
~~~
~~~GO
apiVersion: binliuk.com/v1alpha1
kind: Sample
metadata:
  name: example-sample
spec:
  # Add fields here
  size: 2
  bodyvalue: "Response received from POD : {{env:HOSTNAME}}"
  image: "quay.io/binliuk/sample:v0.1"
~~~
~~~
$ oc create -f deploy/crds/binliuk.com_v1alpha1_sample_cr.yaml 
sample.binliuk.com/example-sample created
~~~

6. Verify the application deployemnet and POD has been created.
~~~
$ oc get deployment
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
example-sample    2/2     2            2           43m
sample-operator   1/1     1            1           51m

$ oc get pods
NAME                              READY   STATUS    RESTARTS   AGE
example-sample-56d56cdd67-bgrf9   1/1     Running   0          44m
example-sample-56d56cdd67-txlwp   1/1     Running   0          44m
nginx                             1/1     Running   12         12h
sample-operator-8f4667bd-zj5d2    1/1     Running   0          50m

$ oc get samples
NAME             AGE
example-sample   5m23s
~~~

7. Create a service and route to test the camel route application(Sample image).
~~~
$ oc create service clusterip sample --tcp=8080:8080
$ oc expose svc/sample
~~~

8. Test the Application
~~~
$ curl http://sample-test.apps.osdtest1.h7z8.p2.openshiftapps.com/test
Response received from POD : example-sample-7d856cb9fc-7pbg2

$ curl http://sample-test.apps.osdtest1.h7z8.p2.openshiftapps.com/test
Response received from POD : example-sample-7d856cb9fc-bjg47
~~~

9. To update the Application size from 2 to 1
~~~GO
apiVersion: binliuk.com/v1alpha1
kind: Sample
metadata:
  name: example-sample
spec:
  # Add fields here
  size: 1
  bodyvalue: "Response received from POD : {{env:HOSTNAME}}"
  image: "quay.io/binliuk/sample:v0.1"
~~~
~~~
$ oc apply -f deploy/crds/binliuk.com_v1alpha1_sample_cr.yaml

$ oc get deployment
NAME              READY     UP-TO-DATE   AVAILABLE   AGE
example-sample    1/1       1            1           3m1s
~~~

Finally, you can delete the namespace using the command
~~~
$ oc delete project sample-operator
~~~

