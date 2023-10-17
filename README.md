### Kubernetes Trusted Platform Module (TPM) using Device Plugin and Gatekeeper

Exposing the `Trusted Platform Module (TPM)` to a non-privleged kubernetes pod.

This allows an pod to interact directly with the TPM device `/dev/tpm0` while the pod runs without `privleged: true` setting (which is dangerous).

Basically, we will use [Kubernetes Generic Device Plugin](https://github.com/squat/generic-device-plugin) to mount the device and also setup [apply custom Pod-level security policies using Gatekeeper](https://cloud.google.com/kubernetes-engine/docs/how-to/pod-security-policies-with-gatekeeper) to limit privleged mode.

Gatekeeper will also only allow mounting the TPM event log from the host to each pod.

Essentially

* Device plugin runs as privleged DaemonSet in `namespace: kube-system`
* Device Plugin surfaces TPM access to each pod in `namespace: ns1`
* Gatekeeper prevents privleged access to `namespace: ns1` (see [Gatekeeper Privleged Containers](https://open-policy-agent.github.io/gatekeeper-library/website/validation/privileged-containers))
* Gatekeeper allows `hostPath` mount to TPM Eventlog to `namespace: ns1` (see [GateKeeper Host Filesystem](https://open-policy-agent.github.io/gatekeeper-library/website/validation/host-filesystem/))


---

### References

- [Kubernetes Generic Device Plugin](https://github.com/squat/generic-device-plugin)
- [Apply custom Pod-level security policies using Gatekeeper](https://cloud.google.com/kubernetes-engine/docs/how-to/pod-security-policies-with-gatekeeper) 
- [Gatekeeper Pod Security Policy](https://github.com/open-policy-agent/gatekeeper-library/tree/master/library/pod-security-policy#pod-security-policies)
- [Pod Security Policies Deprecation](https://kubernetes.io/docs/concepts/security/pod-security-policy/)
- [Kubernetes Trusted Platform Module (TPM) DaemonSet](https://github.com/salrashid123/tpm_daemonset)
- [Trusted Platform Module (TPM) recipes with tpm2_tools and go-tpm](https://github.com/salrashid123/tpm2)
- [golang-jwt for Trusted Platform Module (TPM)](https://github.com/salrashid123/golang-jwt-tpm)
- [TPM Credential Source for Google Cloud SDK](https://github.com/salrashid123/gcp-adc-tpm)
- [TPM based TLS using Attested Keys](https://github.com/salrashid123/tls_ak)

---


### Setup


```bash
gcloud container clusters create cluster-1  \
     --region=us-central1 --machine-type=n2d-standard-2 \
     --enable-confidential-nodes    --enable-shielded-nodes \
     --shielded-secure-boot --shielded-integrity-monitoring \
     --num-nodes=1 --enable-network-policy 

# init gatekeeper
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml

# init the device plugin and gatekeer configs
kubectl apply -f tpm-generic-device-plugin.yaml 

# create the deployment
## note, the pod you are deploying here is basically just tpm2_tools:
###  https://hub.docker.com/r/salrashid123/tpm2_tools

kubectl apply -f pod_manifest.yaml

## access the tpm
kubectl get po -n ns1

POD=$(kubectl get pod -n ns1 -l "app.kubernetes.io/name=tpm-client" -o jsonpath="{.items[0].metadata.name}")
echo $POD

kubectl exec  -it $POD --namespace ns1  -- tpm2_pcrread
  sha1:
    0 : 0x2AAB58E23EA5120D70A3EBCE56BD0E6D5E3035B7
    1 : 0xE3E9E1D9DEACD95B289BBBD3A1717A57AF7D211B
    2 : 0xB2A83B0EBF2F8374299A5B2BDFC31EA955AD7236
    3 : 0xB2A83B0EBF2F8374299A5B2BDFC31EA955AD7236

## read eventlog mounted at /root/binary_bios_measurements
kubectl exec  --namespace ns1 -it $POD   -- tpm2_eventlog /root/binary_bios_measurements
```

---
