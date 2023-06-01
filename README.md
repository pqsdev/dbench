
# dbench
[![Docker Repository on Quay](https://quay.io/repository/pqsdev/dbench/status "Docker Repository on Quay")](https://quay.io/repository/pqsdev/dbench)

Benchmark Kubernetes persistent disk volumes with `fio`: Read/write IOPS, bandwidth MB/s and latency. Based on https://github.com/leeliu/dbench

# Usage

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: dbench
spec:
  storageClassName: ...
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: batch/v1
kind: Job
metadata:
  name: dbench
spec:
  template:
    spec:
      containers:
      - name: dbench
        image: sotoaster/dbench:latest
        imagePullPolicy: IfNotPresent
        env:
          - name: DBENCH_MOUNTPOINT
            value: /data
          - name: FIO_SIZE
            value: 1G
        volumeMounts:
        - name: dbench-pv
          mountPath: /data
      restartPolicy: Never
      volumes:
      - name: dbench-pv
        persistentVolumeClaim:
          claimName: dbench
  backoffLimit: 4
```

1. Edit the `storageClassName` to match your Kubernetes provider's Storage Class `kubectl get storageclasses`
1. Deploy Dbench using: `kubectl apply -f dbench.yaml`
1. Once deployed, the Dbench Job will:
    * provision a Persistent Volume of `1000Gi` (default) using `storageClassName:` (default)
    * run a series of `fio` tests on the newly provisioned disk
    * currently there are 9 tests, 15s per test - total runtime is ~2.5 minutes
1. Follow benchmarking progress using: `kubectl logs -f job/dbench` (empty output means the Job not yet created, or `storageClassName` is invalid, see Troubleshooting below)
1. At the end of all tests, you'll see a summary that looks similar to this:
```
==================
= Dbench Summary =
==================
Random Read/Write IOPS: 75.7k/59.7k. BW: 523MiB/s / 500MiB/s
Average Latency (usec) Read/Write: 183.07/76.91
Sequential Read/Write: 536MiB/s / 512MiB/s
Mixed Random Read/Write IOPS: 43.1k/14.4k
```
6. Once the tests are finished, clean up using: `kubectl delete -f dbench.yaml` and that should deprovision the persistent disk and delete it to minimize storage billing.

## Notes / Troubleshooting

* If the Persistent Volume Claim is stuck on Pending, it's likely you didn't specify a valid Storage Class. Double check using `kubectl get storageclasses`. Also check that the volume size of `1000Gi` (default) is available for provisioning.
* It can take some time for a Persistent Volume to be Bound and the Kubernetes Dashboard UI will show the Dbench Job as red until the volume is finished provisioning.
* It's useful to test multiple disk sizes as most cloud providers price IOPS per GB provisioned. So a `4000Gi` volume will perform better than a `1000Gi` volume. Just edit the yaml, `kubectl delete -f dbench.yaml` and run `kubectl apply -f dbench.yaml` again after deprovision/delete completes.
* A list of all `fio` tests are in [docker-entrypoint.sh](https://github.com/logdna/dbench/blob/master/docker-entrypoint.sh).

## Contributors

* Lee Liu (LogDNA)
* [Alexis Turpin](https://github.com/alexis-turpin)
* [YUUDJ](https://github.com/yuudj)

## License

* MIT

