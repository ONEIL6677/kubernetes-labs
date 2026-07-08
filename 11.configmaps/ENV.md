# Using Environment Variables with a Kubernetes ConfigMap in an Nginx Pod

## Objective

In this practical, we will learn how to:

- Create a configuration file containing course information
- Create a Kubernetes ConfigMap
- Load ConfigMap values as environment variables
- Pass the variables into an `nginx:alpine` container
- Verify the environment variables inside the container

---

# 1. Requirements

Make sure you have:

- Kubernetes installed and running
- `kubectl` installed
- Access to a Kubernetes cluster

Check your Kubernetes connection:

```bash
kubectl get nodes
```

Example output:

```
NAME       STATUS   ROLES
minikube   Ready    control-plane
```

---

# 2. Create a Configuration File

Create a file called:

```
courses.env
```

Command:

```bash
nano courses.env
```

Add the following content:

```text
COURSE1=java
COURSE2=python
COURSE3=html
COURSE4=devops
```

Save the file.

Check the content:

```bash
cat courses.env
```

Output:

```
COURSE1=java
COURSE2=python
COURSE3=html
COURSE4=devops
```

---

# 3. Create a ConfigMap from the Environment File

A ConfigMap stores configuration data that can be used by containers.

Create a ConfigMap called:

```
course-config
```

Command:

```bash
kubectl create configmap course-config --from-env-file=courses.env
```

Explanation:

- `create configmap` → creates a Kubernetes ConfigMap
- `course-config` → name of the ConfigMap
- `--from-env-file` → reads values from an environment file

---

# 4. Verify the ConfigMap

Check available ConfigMaps:

```bash
kubectl get configmaps
```

Output:

```
NAME             DATA
course-config    4
```

View the stored values:

```bash
kubectl describe configmap course-config
```

Expected output:

```
Data
====
COURSE1:
java

COURSE2:
python

COURSE3:
html

COURSE4:
devops
```

---

# 5. Create the Nginx Pod Configuration

Create a file:

```
nginx-env-pod.yaml
```

Command:

```bash
nano nginx-env-pod.yaml
```

Add:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: nginx-env-pod


spec:

  containers:

  - name: nginx-container
    image: nginx:alpine

    envFrom:

    - configMapRef:
        name: course-config
```

---

# 6. Understanding the YAML File

## Container Image

```yaml
image: nginx:alpine
```

The Pod uses the lightweight Nginx Alpine image.

---

## Environment Variables

```yaml
envFrom:

- configMapRef:
    name: course-config
```

This tells Kubernetes:

"Take all values from the ConfigMap called `course-config` and create environment variables inside the container."

The container will automatically receive:

```
COURSE1=java
COURSE2=python
COURSE3=html
COURSE4=devops
```

---

# 7. Create the Pod

Deploy the Pod:

```bash
kubectl apply -f nginx-env-pod.yaml
```

Expected output:

```
pod/nginx-env-pod created
```

---

# 8. Check Pod Status

Run:

```bash
kubectl get pods
```

Example:

```
NAME             STATUS
nginx-env-pod    Running
```

---

# 9. Check Environment Variables Inside the Container

Access the container:

```bash
kubectl exec -it nginx-env-pod -- sh
```

You are now inside the nginx container.

Display all environment variables:

```bash
env
```

You should find:

```
COURSE1=java
COURSE2=python
COURSE3=html
COURSE4=devops
```

---

# 10. Display Individual Variables

Check each course:

```bash
echo $COURSE1
```

Output:

```
java
```

---

```bash
echo $COURSE2
```

Output:

```
python
```

---

```bash
echo $COURSE3
```

Output:

```
html
```

---

```bash
echo $COURSE4
```

Output:

```
devops
```

---

# 11. Check Environment Variables Without Entering the Container

You can also run:

```bash
kubectl exec nginx-env-pod -- env | grep COURSE
```

Output:

```
COURSE1=java
COURSE2=python
COURSE3=html
COURSE4=devops
```

---

# 12. Cleanup

Delete the Pod:

```bash
kubectl delete pod nginx-env-pod
```

Delete the ConfigMap:

```bash
kubectl delete configmap course-config
```

---

# Final Architecture

```
courses.env file

COURSE1=java
COURSE2=python
COURSE3=html
COURSE4=devops

          |
          |
          v

Kubernetes ConfigMap

course-config

          |
          |
          v

Environment Variables

COURSE1
COURSE2
COURSE3
COURSE4

          |
          |
          v

nginx:alpine Container

          |
          |
          v

Application can read the variables
```

---

# Difference Between Volume Mount and Environment Variables

| ConfigMap Method | How Data Appears |
|---|---|
| Volume Mount | Configuration appears as files inside the container |
| Environment Variables | Configuration appears as variables available to the application |

Both methods allow Kubernetes to keep configuration separate from the container image.