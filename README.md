# Install Kubernetes on Power

There are two methods to install Kubernetes on PowerVS:

## 1. Using Ansible Playbook

### Steps:

1. **Manually create instances for master and worker nodes on PowerVS or PowerVC:**
   - Provision the virtual machines (VMs) with the required specifications (CPU, memory, disk space, etc.) based on your requirements.
2. **Clone the repository:**
   ```bash
   git clone https://github.com/kubernetes-sigs/provider-ibmcloud-test-infra.git
   cd provider-ibmcloud-test-infra/kubetest2-tf/data/k8s-ansible
   ls -la
   ```

3. Get the latest Kubernetes build:
   - Retrieve the latest K8s build version from [dl.k8s.io](https://dl.k8s.io/ci/latest.txt).
   - Update the `common.auto.tfvars.json` file with the latest build version.
    ```bash
    cat <<EOF > common.auto.tfvars.json
    {
      "release_marker": "v1.35.0-alpha.0.820+0b6dba8eb028bb",
      "build_version": "v1.35.0-alpha.0.820+0b6dba8eb028bb",
      "runtime": "containerd",
      "cluster_name": "k8s-poorna", # Update with your desired cluster name
      "apiserver_port": 992,
      "workers_count": 1, # Update with the number of workers you want
      "bootstrap_token": "3fb0vq.pf75bfrldw49egvw",
      "kubeconfig_path": "/workspace/kubeconfig", # Update path if necessary 
      "ssh_private_key": "/workspace/id_rsa" # Update path to your private SSH key
    }
    EOF
    ```

4. Get the IP addresses for the master and worker nodes.
    - Retrieve the IP addresses for **masters** and **workers** from your PowerVS or PowerVC environment. These will be needed to update the `hosts` inventory file for Ansible.

    ```
    cat <<EOF > hosts
    [masters]
    135.90.79.100
    
    [workers]
    135.90.79.99
    135.90.79.101
    EOF
    ```
5. Run the Ansible playbook to install Kubernetes:
    - Once the IPs are added to the `hosts` file and the configuration in `common.auto.tfvars.json` is updated, run the following command to trigger the k8s installation:
    ```bash
    ansible-playbook -vv -i hosts --extra-vars @common.auto.tfvars.json install-k8s.yml
    ```



## 2. Using the kubetest-tf Utility

This method will create the virtual machines (VMs) and install Kubernetes in one go. You will need to provide the required details. Additionally, it will run E2E tests if you specify the necessary flags.

### Steps:

1. **Create a Docker container using the image used by the Prow CI job and exec into it:**

    ```bash
    docker run --entrypoint /bin/bash -d -it us-central1-docker.pkg.dev/k8s-staging-test-infra/images/kubekins-e2e:v20250905-c89b045f57-master
    docker exec -it b97066d90dc2 bash
    ```
2. **Clone the repository and install necessary tools (terraform, ansible, kubetest2-tf):**

    ```bash
    git clone https://github.com/kubernetes-sigs/provider-ibmcloud-test-infra.git
    cd provider-ibmcloud-test-infra
    make install-deployer-tf
    ```

3. **Install Kubernetes on PowerVS with the following command:**

    ```bash
    export TF_VAR_powervs_api_key=******

    kubetest2 tf --powervs-image-name CentOS-Stream-10 \
                    --powervs-region syd --powervs-zone syd05 \
                    --powervs-service-id 8880u0-j99i00i0-64b92a \
                    --powervs-ssh-key poorna-ssh-key \
                    --ssh-private-key /workspace/id_rsa \
                    --build-version $(curl https://storage.googleapis.com/k8s-release-dev/ci/latest.txt) \
                    --release-marker $(curl https://storage.googleapis.com/k8s-release-dev/ci/latest.txt) \
                    --cluster-name test-poorna-k8s \
                    --workers-count 1 \
                    --up \
                    --auto-approve --retry-on-tf-failure 3 \
                    --break-kubetest-on-upfail true \
                    --powervs-memory 32 
    ```

This will automatically create the necessary instances, install k8s, and trigger E2E tests. Make sure to replace the placeholders with actual values where necessary (e.g., `TF_VAR_powervs_api_key`, SSH keys).

