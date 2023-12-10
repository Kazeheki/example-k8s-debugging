## Tools you need

- [docker (engine)](https://docs.docker.com/engine/install/)
- [k3d](https://k3d.io/#releases)
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)

## Commands to run

### Preparation

Start the cluster, build and push images, apply resources.
```bash
./scripts/start
```

Get the name of the PODs.
```bash
POD_DISTROLESS=$(kubectl get pod -o jsonpath='{.items[?(@.metadata.labels.app=="distroless")].metadata.name}')
POD_JVM=$(kubectl get pod -o jsonpath='{.items[?(@.metadata.labels.app=="jvm")].metadata.name}')
```

### The interesting part

Let's say you want to check whether there is a shell like 'sh' so that we can
maybe use it to find some file.
```bash
kubectl exec $POD_DISTROLESS -it -- sh
```

Nope, so start the debugging-pod.
```bash
kubectl debug $POD_DISTROLESS \
    --attach=false \
    --image=k3d-app-registry:15000/debugger \
    --container=my-debug-container \
    --share-processes \
    --copy-to=debug-pod
```

Use the shell of your 'debugger' image.
With the `share-processes` you can access the file system and processes of
the POD you'd like to debug (we copied it though, so not on the real POD).
```bash
kubectl exec -it pod/debug-pod -c my-debug-container -- sh
```

To access the file system,
you need to have the same user as the one that is running on the POD.
For that you can run on the now connected container:
```bash
# find the user and group you need for the process you want to debug (the java one).
ps ax -o 'pid,user,group,comm'
# for the distroless pod we have 65532:65532.

# add group with whatever name.
# addgroup --gid $GROUPID $NAME
addgroup --gid 65532 dbg

# add user with that group and whatever name.
# adduser -u $USERID --ingroup $NAME --disabled-password --no-create-home $NAME
adduser -u 65532 --ingroup dbg --disabled-password --no-create-home dbg

# switch to user.
# su $NAME
su dbg
```

Now you can access the file access of the java-process (nr. 7):
```bash
# e.g. the files we copied for running the quarkus app.
ls /proc/7/root/deployments
```

Once done, delete the debug-pod.
```bash
kubectl delete pod/debug-pod
```

If you're using the Dockerfiles provided by Quarkus,
then you can 'exploit' the way the image is built and the (current) way debug is working.
```bash
kubectl debug $POD_JVM \
    --attach=false \
    --image=k3d-app-registry:15000/example-jvm \
    --container=app \
    --copy-to=debug-pod \
    --env="[JAVA_DEBUG=true,JAVA_DEBUG_PORT=5005]"
```
Notice that in the debug-pod there is now still only one container.

Now with a little port-forwarding, we can make the debugging possible.
```bash
kubectl port-forward pod/debug-pod 5005:5005 8080:8080
```
You can now attach your debugger on port `5005`.

### Clean-up

Tear down cluster and remove the docker images we created.
```bash
./scripts/clean_up
```
