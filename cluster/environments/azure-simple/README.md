## Getting started with azure-simple environment

The following sections provide instructions to deploy azure-simple environment in Azure.

- [Create an Azure service principal](#create-an-azure-service-principal)
- [Create Terraform configuration files](#create-terraform-configuration-files)
- [Configure Terraform to store state data in Azure](#configure-terraform-to-store-state-data-in-azure)
- [Create the AKS cluster using Terraform](#create-the-aks-cluster-using-terraform)
- [Configure `kubectl` to see your new AKS cluster](configure-kubectl-to-see-your-new-aks-cluster)
- [Verify that your AKS cluster is healthy](verify-that-your-aks-cluster-is-healthy)

Please [install required tools](/cluster/Azure/readme.md/#install-required-tools) as well as [setup GitOps repo for Flux](/cluster/Azure/readme.md/#set-up-gitops-repository-for-flux) before continuing to the next section if you have not already.

### Create an Azure Service Principal
You can generate an Azure Service Principal using the [`az ad sp create-for-rbac`](https://docs.microsoft.com/en-us/cli/azure/ad/sp?view=azure-cli-latest#az-ad-sp-create) command with `--skip-assignment` option. The `--skip-assignment` parameter limits any additional permissions from being assigned the default [`Contributor`](https://docs.microsoft.com/en-us/azure/role-based-access-control/rbac-and-directory-admin-roles#azure-rbac-roles) role in Azure subscription.

```bash
$ az ad sp create-for-rbac --subscription <id | name>
```
The output of the above commands are similar to the following example:
```bash
{
  "appId": "50d65587-abcd-4619-1234-f99fb2ac0987",
  "displayName": "azure-cli-2019-01-23-20-27-37",
  "name": "http://azure-cli-2019-01-23-20-27-37",
  "password": "3ac38e00-aaaa-bbbb-bb87-7222bc4b1f11",
  "tenant": "72f988bf-86f1-41af-91ab-2d7cd011db47"
}
```

Note: You may receive an error if you do not have sufficient permissions on your Azure subscription to create a service principal.  If this happens, contact a subscription administrator to determine whether you have contributor-level access to the subscription.

### Create Terraform Configuration Files

This is a two step process:

1. Create a new cluster configuration by copying an existing Terraform template
2. Customize your cluster by entering configuration values into '*.tfvars' files 

### Clone Terraform Template

The typical way to create a new environment is to start from an existing template. To create a cluster environment based on the `azure-simple` template, copy it to a new subdirectory with the name of the cluster you want to create:

```bash
$ cp -r cluster/environments/azure-simple cluster/environments/<your new cluster name>
```

### Edit Configuration Values

With this new environment created, edit `environments/azure/<your new cluster name>/terraform.tfvars` and update the following variables:

- `resource_group_name`: Name of the resource group where the cluster will be located.
- `resource_group_location`: Azure region the resource group should be created in.
- `cluster_name`: Name of the Kubernetes cluster you want to create.
- `agent_vm_count`: The number of agents VMs in the the node pool.
- `dns_prefix`: DNS name for accessing the cluster from the internet (up to 64 characters in length, alphanumeric characters and hyphen '-' allowed, and must start with a letter).
- `service_principal_id`: The id of the service principal used by the AKS cluster.  This is generated using the Azure CLI (see [Create an Azure service principal](#create-an-azure-service-principal) for details).
- `service_principal_secret`: The secret of the service principal used by the AKS cluster.  This is generated using the Azure CLI (see [Create an Azure service principal](#create-an-azure-service-principal) for details).
- `ssh_public_key`: Contents of a public key authorized to access the virtual machines within the cluster.  Copy the entire string contents of the gitops_repo_key.pub file that was generated in the [Set up GitOps repository for Flux](#set-up-gitops-repository-for-flux) step.
- `gitops_ssh_url`: The git repo that contains the resource manifests that should be deployed in the cluster in ssh format (eg. `git@github.com:timfpark/fabrikate-cloud-native-manifests.git`). This repo must have a deployment key configured to accept changes from `GitOps_ssh_key` (see [Set up GitOps repository for Flux](#set-up-gitops-repository-for-flux) for more details).
- `gitops_ssh_key`: Absolute path to the *private key file* (i.e. gitops_repo_key) that was generated in the [Set up GitOps repository for Flux](#set-up-gitops-repository-for-flux) step and configured to work with the GitOps repository.
-`gitops_path`: Path to a subdirectory, or folder in a git repo

(For a full list of customizable variables, see [variables.tf](../azure/aks/variables.tf).

Finally, if you are deploying a production cluster, you will want to [configure storage](#storing-terraform-state) of Terraform state.

### Configure Terraform to Store State Data in Azure

Terraform records the information about what is created in a [Terraform state file](https://www.terraform.io/docs/state/) after it finishes applying.  By default, Terraform will create a file named `terraform.tfstate` in the directory where Terraform is applied.  Terraform needs this information so that it can be loaded when we need to know the state of the cluster for future modifications.

In production scenarios, storing the state file on a local file system is not desired because typically you want to share the state between operators of the system.  Instead, we configure Terraform to store state remotely, and in Bedrock we use Azure Blob Store for this storage.  This is defined using a `backend` block.  The basic block looks like:

```bash
terraform {
   backend “azure” {
   }
}
```

In order to setup an Azure backend, navigate to the [backend state](/Azure/backend-state) directory and issue the following command:

```bash
> terraform apply -var 'name=<storage account name>' -var 'location=<storage account location>' -var 'resource_group_name=<storage account resource group>'
```

where `storage account name` is the name of the storage account to store the Terraform state, `storage account location` is the Azure region the storage account should be created in, and `storage account resource group` is the name of the resource group to create the storage account in.  

Once the storage account is created, we need to fetch storage account key so we can configure Terraform with it:

```bash
>  az storage account keys list --account-name <storage account name>
```

With this, update `backend.tfvars` file in your cluster environment directory with these values and use `terraform init -backend-config=./backend.tfvars` to setup usage of the Azure backend.

### Create the AKS Cluster using Terraform

Bedrock requires a bash shell for the executing the automation. Currently MacOSX, Ubuntu, and the Windows Subsystem for Linux (WSL) are supported.

From the directory of the cluster you defined above (eg. `environments/azure/<your new cluster name>`), run:

```
$ terraform init
```

This will download all of the modules needed for the deployment.  You can then deploy the cluster with:

```
$ terraform apply
```

This will display the plan for what infrastructure Terraform plans to deploy into your subscription and ask for your confirmation.

Once you have confirmed the plan, Terraform will deploy the cluster, install [Flux](https://github.com/weaveworks/flux)
in the cluster to start a [GitOps](https://www.weave.works/blog/GitOps-operations-by-pull-request) operator in the cluster, and deploy any resource manifests in the `gitops_ssh_url`.

If errors occur during deployment, follow-on actions will depend on the nature of the error and at what stage it occurred.  If the error cannot be resolved in a way that enables the remaining resources to be deployed/installed, it is possible to re-attempt the entire cluster deployment.  First, from within the `environments/azure/<your new cluster name>` directory, run `terraform destroy`, then fix the error if applicable (necessary tool not installed, for example), and finally re-run `terraform apply`.

### Configure `kubectl` to see your new AKS cluster

Once your cluster has been created the credentials for the cluster will be placed in the specified `output_directory` which defaults to `./output`. 

With the default kube config file name, you can copy this to your `~/.kube/config` by executing:

```bash
$ KUBECONFIG=./output/bedrock_kube_config:~/.kube/config kubectl config view --flatten > merged-config && mv merged-config ~/.kube/config
```

or directly use the kube_config file with default output file name:

```
$ KUBECONFIG=./output/bedrock_kube_config kubectl get po --namespace=flux` 
```

### Verify that your AKS cluster is healthy

Enter the following command to view the pods running in your cluster:

```bash
kubectl get pods --n flux
```

You should see two Flux pods running (flux & flux-memcached). If your cluster is healthy and Flux is able to successfully connect to your GitOps repo, you will see something like this:

```bash
NAME                              READY   STATUS    RESTARTS   AGE
flux-568b7ccbbc-qbnmv             1/1     Running   0          8m07s
flux-memcached-59947476d9-d6kqw   1/1     Running   0          8m07s
```

If the Flux pod shows a status other than 'Running' (e.g. 'Restarting...'), it likely indicates that it is unable to connect to your
GitOps repo. In this case, verify that you have assigned the correct public key to the GitOps repo (with write privileges) and that 
you have specified the matching private key in your Terraform configuration.
