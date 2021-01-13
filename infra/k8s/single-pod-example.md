# Writing YAML for a single pod is not standard practice so we are going to write a deployment instead
apiVersion: v1
kind: Pod
metadata:
  name: posts
spec:
  containers:
    - name: posts
      image: nickyiliwang/posts:0.0.1