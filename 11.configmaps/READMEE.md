# Mounting a ConfigMap Created from a Text File into an Nginx Pod

## Objective

In this practical, we will learn how to:

- Create a text file called `courses`
- Store course names inside the file
- Create a Kubernetes ConfigMap from the file
- Mount the ConfigMap as a file inside a Pod
- Use the `nginx:alpine` container to access the mounted file

---

# 1. Requirements

Before starting, make sure you have:

- Kubernetes cluster running
- `kubectl` installed
- A working Kubernetes context

Check Kubernetes connection:

```bash
kubectl get nodes
```

Example output:

```
NAME        STATUS   ROLES
minikube    Ready    control-plane
```

---

# 2. Create the Courses Text File

First create a file named:

```
courses
```

Command:

```bash
nano courses
```

Add the following content:

```
java
python
html
devops
```

Save the file.

Verify the content:

```bash
cat courses
```

Output:

```
java
python
html
devops
```

---

# 3. Create a Kubernetes ConfigMap from the Text File

A ConfigMap stores configuration data separately from the application.

We will create a ConfigMap called:

```
course-config
```

Command:

```bash
kubectl create configmap course-config --from-file=courses
```

Explanation:

- `configmap` → creates a Kubernetes ConfigMap
- `course-config` → name of the ConfigMap
- `--from-file=courses` → takes data from the courses text file


Check that the ConfigMap was created:

```bash
kubectl get configmaps
```

Expected output:

```
NAME             DATA
course-config    1
```

---

# 4. View ConfigMap Content

To see what is stored inside:

```bash
kubectl describe configmap course-config
```

You should see:

```
Data
====
courses:
----
java
python
html
devops
```

---

# 5. Create a Pod YAML File

Create a file:

```
nginx-pod.yaml
```

Command:

```bash
nano nginx-pod.yaml
```

Add the following configuration:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-course-pod

spec:

  containers:

  - name: nginx-container
    image: nginx:alpine

    volumeMounts:
    - name: course-volume
      mountPath: /usr/share/nginx/html/courses


  volumes:

  - name: course-volume
    configMap:
      name: course-config
```

---

# 6. Understanding the YAML File

## Pod Information

```yaml
metadata:
  name: nginx-course-pod
```

Creates a Pod called:

```
nginx-course-pod
```

---

## Container Section

```yaml
containers:

- name: nginx-container
  image: nginx:alpine
```

Creates a container using:

```
nginx:alpine
```

Nginx is a lightweight web server image.

---

## Volume Mount

```yaml
volumeMounts:

- name: course-volume
  mountPath: /usr/share/nginx/html/courses
```

This means:

The ConfigMap content will appear inside the container at:

```
/usr/share/nginx/html/courses
```

---

## Volume Definition

```yaml
volumes:

- name: course-volume

  configMap:
    name: course-config
```

This connects the Pod to the ConfigMap.

The Pod receives the data stored inside:

```
course-config
```

---

# 7. Create the Pod

Deploy the Pod:

```bash
kubectl apply -f nginx-pod.yaml
```

Expected output:

```
pod/nginx-course-pod created
```

---

# 8. Check Pod Status

Run:
```bash
kubectl get pods
```

Example:

```
NAME                 STATUS
nginx-course-pod     Running
```

---

# 9. Verify the Mounted File Inside the Container

Enter the container:
```bash
kubectl exec -it nginx-course-pod -- sh
```

You are now inside the nginx container.

Navigate to the mounted location:

```bash
cd /usr/share/nginx/html/courses
```

List files:
```bash
ls
```
>Output:
```
courses
```

Display the file:

```bash
cat courses
```

Output:

```
java
python
html
devops
```

---

# 10. Access the File Through Nginx

Because the file is inside the nginx web directory:

```
/usr/share/nginx/html
```

we can access it through the nginx server.

Find the Pod IP:

```bash
kubectl get pod nginx-course-pod -o wide
```

Example:

```
IP:
10.244.0.5
```

Test using:

```bash
curl http://10.244.0.5/courses/courses
```

Output:

```
java
python
html
devops
```

---

# 11. Cleanup

Delete the Pod:

```bash
kubectl delete pod nginx-course-pod
```

Delete the ConfigMap:

```bash
kubectl delete configmap course-config
```

---

# Summary

In this exercise we learned:

| Step | Action |
|---|---|
| 1 | Created a text file called `courses` |
| 2 | Added course names |
| 3 | Created a ConfigMap from the file |
| 4 | Mounted the ConfigMap into a Pod |
| 5 | Used nginx:alpine as the container image |
| 6 | Accessed the mounted file inside the container |

The final architecture is:

```
courses file
     |
     |
     v
ConfigMap
(course-config)
     |
     |
     v
Kubernetes Pod
     |
     |
     v
nginx:alpine container
     |
     |
     v
/usr/share/nginx/html/courses/courses
```

The application container does not store the configuration directly. Kubernetes injects the configuration using a ConfigMap.