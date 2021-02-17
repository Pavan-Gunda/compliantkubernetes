# Compliant Kubernetes Deployment on GCP

This document contains instructions on how to set up a service cluster and a workload cluster on GCP. The document is split into two parts:

- Cluster setup (Setting up infrastructure and the Kubernetes clusters)
- Apps setup

Before starting, make sure you have [all necessary tools](https://github.com/elastisys/compliantkubernetes/blob/main/docs/operator-manual/getting-started.md). In addition to these general tools, you will also need:

- A GCP project
- JSON keyfile for your project
- SSH key that you will use to access GCP, which you have [added to the metadata in your GCP project](https://cloud.google.com/compute/docs/instances/adding-removing-ssh-keys).

## Cluster setup

1. Clone the [compliantkubernetes-kubespray](https://github.com/elastisys/compliantkubernetes-kubespray) repository. For all commands in this guide, your working directory is assumed to be the root directory of this repository.
2. Init and update the [kubespray](https://github.com/kubernetes-sigs/kubespray) gitsubmodule:
   ```
   git submodule init
   git submodule update
   ```
3. In `config/gcp/group_vars/all/ck8s-gcp.yml`, set the value of `gcp_pd_csi_sa_cred_file` to the path of your JSON keyfile.
4. Modify `kubespray/contrib/terraform/gcp/tfvars.json` in the following way:
    - Set `gcp_project_id` to the ID of your GCP project.
    - Set `keyfile_location` to the location of your JSON keyfile.
    - Set `ssh_pub_key` to the path of your public ssh key.
    - In `ssh_whitelist`, `api_server_whitelist` and `nodeport_whitelist`, add IP address(es) that you want to be able to access the cluster.
5. Set up the nodes by performing the following steps, replacing `<prefix>` with `sc`/`wc`:
    1. Set up the nodes with terraform. If desired, first modify `"machines"` in `kubespray/contrib/terraform/gcp/tfvars.json` to add/remove nodes, change node sizes, etc. (For setting up compliantkubernetes-apps in the service cluster, one `n1-standard-8` worker and one `n1-standard-4` worker is enough.)
    ```bash
    cd kubespray/contrib/terraform/gcp
    export CLUSTER=<prefix>
    terraform init
    terraform apply -var-file tfvars.json -state $CLUSTER.tfstate -var prefix=$CLUSTER
    ```
    Save the outputs from `apply`. (Alternatively, get them later by running `terraform output -state $CLUSTER.tfstate` in the folder with the state file)

    2. Generate inventory file:
    ```bash
    cd kubespray/contrib/terraform/gcp
    ./generate-inventory.sh $CLUSTER.tfstate > $CLUSTER-inventory.ini
    ```
    3. Initialize the config:
    ```bash
    export CK8S_CONFIG_PATH=~/.ck8s/<environment-name>
    ./bin/ck8s-kubespray init $CLUSTER gcp <path to ssh key> [<SOPS fingerprint>]
    ```  
    * `path to ssh key` should point to your private ssh key. It will be copied into your config path and encrypted with SOPS, the original file left as it were.
    * `SOPS fingerprint` is the gpg fingerprint that will be used for SOPS encryption. You need to set this or the environment variable `CK8S_PGP_FP` the first time SOPS is used in your specified config path.
    4. Edit the IP addresses and nodes in your `inventory.ini` (found in your config path) to match the VMs that should be part of the cluster. The contents of the `$CLUSTER-inventory.ini` file that you generated in the previous section can be copy-pasted into the appropriate `inventory.ini` file.
    5. Run the following command to fix some formatting in your config:
        ```bash
        sed -e 's@^---$@@' -i $CK8S_CONFIG_PATH/$CLUSTER-config/group_vars/all/all.yml
        sed -e 's@^---$@@' -i $CK8S_CONFIG_PATH/$CLUSTER-config/group_vars/k8s-cluster/k8s-cluster.yml
        ```
    6. Run kubespray to set up the kubernetes cluster:
        ```bash
        ./bin/ck8s-kubespray apply $CLUSTER
        ```
    7. In your config path, open `.state/kube_config_<prefix>.yaml` with SOPS and change `clusters.cluster.server` to the `control_plane_lb_ip_address` you got from `terraform apply`.

## Apps setup

The following instructions were made for release v0.9.0 of compliantkubernetes-apps. There may be discrepancies with newer versions.

Note that in release v0.9.0 of compliantkubernetes-apps, fluentd will not work in the service cluster.

1. Set up [ck8s-dns](https://github.com/elastisys/ck8s-dns) on a provider of your choice, using the `ingress_controller_lb_ip_address` from `terraform apply` as your loadbalancer IPs.
2. In `compliantkubernetes-apps`, run:
    ```bash
    export CK8S_ENVIRONMENT_NAME=<environment-name>
    export CK8S_CLOUD_PROVIDER=baremetal
    ./bin/ck8s init
    ```
3. You will need to modify `secrets.yaml`, `sc-config.yaml` and `wc-config.yaml` in your config path.
    - `secrets.yaml`
        - Uncomment `objectStorage.gcs.keyfileData` and paste the contents of your JSON keyfile as the value.
    - `sc-config.yaml` AND `wc-config.yaml`
        - Set `global.baseDomain` to `<environment-name>.<dns-domain>` and `global.opsDomain` to `ops.<environment-name>.<dns-domain>` where `<dns-domain>` is the value of `dnsDomain` found in `dns-tfvars.json` in your config path.
        - Set `storageClasses.default` to `csi-gce-pd`. Also set any `storageClasses.*.enabled` that are set to `null` to `false`.
        - Set `objectStorage.type` to `"gcs"`.
        - Uncomment `objectStorage.gcs.project` and set it to the name of your GCP project.
        - Set `issuers.letsencrypt.prod.email` and `issuers.letsencrypt.staging.email` to email addresses of choice.
    - `sc-config.yaml`
        - Set `influxDB.backupRetention.enabled` to `false`.
        - Set `elasticsearch.dataNode.storageClass` to `csi-gce-pd`.
4. Create buckets for storage on GCP (found under "Storage"). The names must match the bucket names found in your `sc-config.yaml` and `wc-config.yaml` in the config path.
5. Set the default storageclass by running the following command:
    ```
    bin/ck8s ops kubectl sc "patch storageclass csi-gce-pd -p '{\"metadata\": {\"annotations\":{\"storageclass.kubernetes.io/is-default-class\":\"true\"}}}'"
    bin/ck8s ops kubectl wc "patch storageclass csi-gce-pd -p '{\"metadata\": {\"annotations\":{\"storageclass.kubernetes.io/is-default-class\":\"true\"}}}'"
    ```
6. Apply the apps:
    ```
    bin/ck8s apply sc
    bin/ck8s apply wc
    ```

Done. You should now have a functioning Compliant Kubernetes environment.
