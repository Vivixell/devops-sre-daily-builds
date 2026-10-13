# kubectl Commands

> **Instructions for any assistant editing this file:** Each group below is a section. When new commands are added, append them under the matching group heading, keep the existing format (a code block with a short `#` comment per command), and replace the `_No commands yet._` placeholder when the first commands go in. Do not remove, rename, or reorder groups. Keep notes brief.

---

## Group 0: Context and Cluster Access

```bash
kubectl config current-context        # show the active context
kubectl config get-contexts           # list all contexts (* marks current)
kubectl config use-context <name>     # switch context
kubectl config view                   # show full kubeconfig (certs redacted)
kubectl config view --minify          # only the current context's config
kubectl config set-context --current --namespace=<ns>   # set default namespace
```

EKS context setup:
```bash
aws eks update-kubeconfig --name <cluster> --region <region>                  # add cluster to kubeconfig and set as current context
aws eks update-kubeconfig --name <cluster> --region <region> --alias <name>   # readable context name instead of the ARN
aws eks update-kubeconfig --name <cluster> --region <region> --profile <profile>   # use a specific AWS CLI profile
```

Note: this writes to the local kubeconfig (`~/.kube/config`). It doesn't fetch a context from the cluster. Auth uses short-lived tokens from `aws eks get-token`.

---

## Group 1: Inspecting Resources

```bash
kubectl get nodes                     # list nodes and status
kubectl get nodes -o wide             # adds internal IP, OS image, kernel, container runtime
kubectl describe node <node-name>     # capacity, conditions, taints, allocated resources
```

Note: the `CONTAINER-RUNTIME` column in `-o wide` shows containerd or CRI-O.

---

## Group 2: Creating and Applying Resources

```bash
kubectl run nginx --image=nginx                      # create a pod
kubectl run nginx --image=nginx --port=80            # declare the container port
kubectl run nginx --image=nginx --env="KEY=value"    # set an env var
kubectl run nginx --image=nginx --labels="app=web"   # add labels
kubectl run busybox --image=busybox -it --rm -- sh   # interactive temp pod, deleted on exit
kubectl run busybox --image=busybox -- sleep 3600    # override the command/args
```

Generate YAML without creating anything:
```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
```

Check and clean up:
```bash
kubectl get pods
kubectl get pods -o wide
kubectl describe pod nginx
kubectl delete pod nginx
```

---

## Group 3: Workloads (Deployments, Scaling, Rollouts, Jobs)

_No commands yet._

---

## Group 4: Debugging

_No commands yet._

---

## Group 5: Config and Secrets

_No commands yet._

---

## Group 6: Services and Networking

_No commands yet._

---

## Group 7: Namespaces, Labels and Annotations

_No commands yet._

---

## Group 8: Access and RBAC

_No commands yet._

---

## Group 9: Node Operations

_No commands yet._

---

## Group 10: Help and Discovery

```bash
kubectl --help                        # all commands
kubectl run --help                    # flags and examples for a command
kubectl explain pod                   # API docs for a resource
kubectl explain pod.spec.containers   # drill into nested fields
kubectl explain pod --recursive       # full field tree
kubectl api-resources                 # all resource types, short names, API groups
```

Note: `kubectl explain` works offline in the CKAD exam terminal.