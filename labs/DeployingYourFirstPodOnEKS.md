# 🧪 Understanding `kubectl run` and Pod Manifest Creation

There are several things that happen when you execute the `kubectl run` command, but before we review them, let’s look at the manifest being created.

---

## 🔍 Dry Run to Inspect the Manifest

```bash
kubectl run busybox --image busybox --restart Never --dry-run=client -o yaml
