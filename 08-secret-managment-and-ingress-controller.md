# Configure AKS Ingress Controller with Azure Key Vault integration

Previously you have configured [workload prerequisites](./08-workload-prerequisites.md). These steps configure Traefik, the AKS ingress solution used in this reference implementation, so that it can securely expose the web app to your Application Gateway.

## Steps

1. Get AKS `kubectl` credentials.

   > In the [Azure Active Directory Integration](03-aad.md) step, we placed our cluster under AAD group-backed RBAC. This is the first time we are seeing this used. `az aks get-credentials` allows you to use `kubectl` commands against your cluster. Without the AAD integration, you'd have to use `--admin` here, which isn't what we want to happen. In a following step, you'll log in with a user that has been added to the Azure AD security group used to back the Kubernetes RBAC admin role. Executing the first `kubectl` command below will invoke the AAD login process to auth the _user of your choice_, which will then be checked against Kubernetes RBAC to perform the action. The user you choose to log in with _must be a member of the AAD group bound_ to the `cluster-admin` ClusterRole. For simplicity you could either use the "break-glass" admin user created in [Azure Active Directory Integration](03-aad.md) (`bu0001a0042-admin`) or any user you assigned to the `cluster-admin` group assignment in your [`cluster-rbac.yaml`](cluster-manifests/cluster-rbac.yaml) file. If you skipped those steps you can use `--admin` to proceed, but proper AAD group-based RBAC access is a critical security function that you should invest time in setting up.

   ```bash
   az aks get-credentials -g rg-bu0001a0042-03 -n $AKS_CLUSTER_NAME_BU0001A0042_03
   az aks get-credentials -g rg-bu0001a0042-04 -n $AKS_CLUSTER_NAME_BU0001A0042_04
   ```

   :warning: At this point two important steps are happening:

   - The `az aks get-credentials` command will be fetch a `kubeconfig` containing references to the AKS cluster you have created earlier.
   - To _actually_ use the cluster you will need to authenticate. For that, run any `kubectl` commands which at this stage will prompt you to authenticate against Azure Active Directory. For example, run the following command:

   ```bash
   kubectl get nodes --context $AKS_CLUSTER_NAME_BU0001A0042_03
   ```

   Once the authentication happens successfully, some new items will be added to your `kubeconfig` file such as an `access-token` with an expiration period. For more information on how this process works in Kubernetes please refer to [the related documentation](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens).

1. Get the AKS Ingress Controller Managed Identity details.

   ```bash
   TRAEFIK_USER_ASSIGNED_IDENTITY_RESOURCE_ID_BU0001A0042_03=$(az deployment group show --resource-group rg-bu0001a0042-03 -n cluster-stamp --query properties.outputs.aksIngressControllerPodManagedIdentityResourceId.value -o tsv)
   TRAEFIK_USER_ASSIGNED_IDENTITY_CLIENT_ID_BU0001A0042_03=$(az deployment group show --resource-group rg-bu0001a0042-03 -n cluster-stamp --query properties.outputs.aksIngressControllerPodManagedIdentityClientId.value -o tsv)
   ```

1. Ensure Flux has created the following namespace.

   ```bash
   # press Ctrl-C once you receive a successful response
   kubectl get ns a0042 -w --context $AKS_CLUSTER_NAME_BU0001A0042_03
   ```

1. Create Traefik's Azure Managed Identity binding.

   > Create the Traefik Azure Identity and the Azure Identity Binding to let Azure Active Directory Pod Identity to get tokens on behalf of the Traefik's User Assigned Identity and later on assign them to the Traefik's pod.

   ```bash
   cat <<EOF | kubectl create --context $AKS_CLUSTER_NAME_BU0001A0042_03 -f -
   apiVersion: "aadpodidentity.k8s.io/v1"
   kind: AzureIdentity
   metadata:
     name: podmi-ingress-controller-identity
     namespace: a0042
   spec:
     type: 0
     resourceID: $TRAEFIK_USER_ASSIGNED_IDENTITY_RESOURCE_ID_BU0001A0042_03
     clientID: $TRAEFIK_USER_ASSIGNED_IDENTITY_CLIENT_ID_BU0001A0042_03
   ---
   apiVersion: aadpodidentity.k8s.io/v1
   kind: AzureIdentityBinding
   metadata:
     name: podmi-ingress-controller-binding
     namespace: a0042
   spec:
     azureIdentity: podmi-ingress-controller-identity
     selector: podmi-ingress-controller
   EOF
   ```

1. Create the Traefik's Secret Provider Class resource.

   > The Ingress Controller will be exposing the wildcard TLS certificate you created in a prior step. It uses the Azure Key Vault CSI Provider to mount the certificate which is managed and stored in Azure Key Vault. Once mounted, Traefik can use it.
   >
   > Create a `SecretProviderClass` resource with with your Azure Key Vault parameters for the [Azure Key Vault Provider for Secrets Store CSI driver](https://github.com/Azure/secrets-store-csi-driver-provider-azure).

   ```bash
   KEYVAULT_NAME_BU0001A0042_03=$(az deployment group show -g rg-bu0001a0042-03 -n cluster-stamp  --query properties.outputs.keyVaultName.value -o tsv)
   cat <<EOF | kubectl create --context $AKS_CLUSTER_NAME_BU0001A0042_03 -f -
   apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
   kind: SecretProviderClass
   metadata:
     name: aks-ingress-contoso-com-tls-secret-csi-akv
     namespace: a0042
   spec:
     provider: azure
     parameters:
       usePodIdentity: "true"
       keyvaultName: $KEYVAULT_NAME_BU0001A0042_03
       objects:  |
         array:
           - |
             objectName: traefik-ingress-internal-aks-ingress-contoso-com-tls
             objectAlias: tls.crt
             objectType: cert
           - |
             objectName: traefik-ingress-internal-aks-ingress-contoso-com-tls
             objectAlias: tls.key
             objectType: secret
       tenantId: $TENANTID_AZURERBAC
   EOF
   ```

1. Import the Traefik container image to your container registry.

   > Public container registries are subject to faults such as outages (no SLA) or request throttling. Interruptions like these can be crippling for an application that needs to pull an image _right now_. To minimize the risks of using public registries, store all applicable container images in a registry that you control, such as the SLA-backed Azure Container Registry.

   ```bash
   # Get your ACR cluster name
   ACR_NAME_BU0001A0042=$(az deployment group show -g rg-bu0001a0042-shared -n shared-svcs-stamp --query properties.outputs.containerRegistryName.value -o tsv)

   # Import ingress controller image hosted in public container registries
   az acr import --source docker.io/library/traefik:2.2.1 -n $ACR_NAME_BU0001A0042
   ```

1. Install the Traefik Ingress Controller.

   > Install the Traefik Ingress Controller; it will use the mounted TLS certificate provided by the CSI driver, which is the in-cluster secret management solution.

   > If you used your own fork of this GitHub repo, update the one `image:` value in [`traefik.yaml`](./workload/traefik.yaml) to reference your container registry instead of the default public container registry and change the URL below to point to yours as well.

   :warning: Deploying the traefik `traefik.yaml` file unmodified from this repo will be deploying your workload to take dependencies on a public container registry. This is generally okay for learning/testing, but not suitable for production. Before going to production, ensure _all_ image references are from _your_ container registry or another that you feel confident relying on.

   ```bash
   kubectl create -f https://raw.githubusercontent.com/mspnp/aks-secure-baseline/main/workload/traefik-03.yaml --context $AKS_CLUSTER_NAME_BU0001A0042_03
   ```

1. Wait for Traefik to be ready.

   > During Traefik's pod creation process, AAD Pod Identity will need to retrieve token for Azure Key Vault. This process can take time to complete and it's possible for the pod volume mount to fail during this time but the volume mount will eventually succeed. For more information, please refer to the [Pod Identity documentation](https://github.com/Azure/secrets-store-csi-driver-provider-azure/blob/master/docs/pod-identity-mode.md).

   ```bash
   kubectl wait -n a0042 --for=condition=ready pod --selector=app.kubernetes.io/name=traefik-ingress-ilb --timeout=90s --context $AKS_CLUSTER_NAME_BU0001A0042_03
   ```

1. Install the Traefik Ingress Controller in the second AKS Cluster

   ```bash
   TRAEFIK_USER_ASSIGNED_IDENTITY_RESOURCE_ID_BU0001A0042_04=$(az deployment group show --resource-group rg-bu0001a0042-04 -n cluster-stamp --query properties.outputs.aksIngressControllerPodManagedIdentityResourceId.value -o tsv)
   TRAEFIK_USER_ASSIGNED_IDENTITY_CLIENT_ID_BU0001A0042_04=$(az deployment group show --resource-group rg-bu0001a0042-04 -n cluster-stamp --query properties.outputs.aksIngressControllerPodManagedIdentityClientId.value -o tsv)
   kubectl get ns a0042 -w --context $AKS_CLUSTER_NAME_BU0001A0042_04
   cat <<EOF | kubectl create --context $AKS_CLUSTER_NAME_BU0001A0042_04 -f -
   apiVersion: "aadpodidentity.k8s.io/v1"
   kind: AzureIdentity
   metadata:
     name: podmi-ingress-controller-identity
     namespace: a0042
   spec:
     type: 0
     resourceID: $TRAEFIK_USER_ASSIGNED_IDENTITY_RESOURCE_ID_BU0001A0042_04
     clientID: $TRAEFIK_USER_ASSIGNED_IDENTITY_CLIENT_ID_BU0001A0042_04
   ---
   apiVersion: "aadpodidentity.k8s.io/v1"
   kind: AzureIdentityBinding
   metadata:
     name: podmi-ingress-controller-binding
     namespace: a0042
   spec:
     azureIdentity: podmi-ingress-controller-identity
     selector: podmi-ingress-controller
   EOF

   KEYVAULT_NAME_BU0001A0042_04=$(az deployment group show -g rg-bu0001a0042-04 -n cluster-stamp  --query properties.outputs.keyVaultName.value -o tsv)

   cat <<EOF | kubectl create --context $AKS_CLUSTER_NAME_BU0001A0042_04 -f -
   apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
   kind: SecretProviderClass
   metadata:
     name: aks-ingress-contoso-com-tls-secret-csi-akv
     namespace: a0042
   spec:
     provider: azure
     parameters:
       usePodIdentity: "true"
       keyvaultName: $KEYVAULT_NAME_BU0001A0042_04
       objects:  |
         array:
           - |
             objectName: traefik-ingress-internal-aks-ingress-contoso-com-tls
             objectAlias: tls.crt
             objectType: cert
           - |
             objectName: traefik-ingress-internal-aks-ingress-contoso-com-tls
             objectAlias: tls.key
             objectType: secret
       tenantId: $TENANTID_AZURERBAC
   EOF

   kubectl create -f https://raw.githubusercontent.com/mspnp/aks-secure-baseline/main/workload/traefik-04.yaml --context $AKS_CLUSTER_NAME_BU0001A0042_04
   kubectl wait --namespace a0042 --for=condition=ready pod --selector=app.kubernetes.io/name=traefik-ingress-ilb --timeout=90s --context $AKS_CLUSTER_NAME_BU0001A0042_04
   ```

### Next step

:arrow_forward: [Deploy the Workload](./09-workload.md)
