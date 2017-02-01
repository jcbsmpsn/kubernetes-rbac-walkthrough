# Provisioning Google Cloud Platform Resources

You will end up with the GCP resources.

```sh
gcloud compute instances list
```

```sh
NAME         ZONE           MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP      STATUS
controller0  us-central1-b  n1-standard-1               10.128.0.2   104.154.198.183  RUNNING
worker0      us-central1-b  n1-standard-1               10.128.0.3   104.154.203.53   RUNNING
```

## Creating Instances

```sh
gcloud compute instances create controller0 \
 --boot-disk-size 200GB \
 --can-ip-forward \
 --image ubuntu-1604-xenial-v20160921 \
 --image-project ubuntu-os-cloud \
 --machine-type n1-standard-1
```

```sh
gcloud compute instances create worker0 \
 --boot-disk-size 200GB \
 --can-ip-forward \
 --image ubuntu-1604-xenial-v20160921 \
 --image-project ubuntu-os-cloud \
 --machine-type n1-standard-1
```
