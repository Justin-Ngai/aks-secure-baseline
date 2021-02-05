# Prepare to bootstrap the cluster

Now that [the AKS cluster](./09-aks-cluster.md) has been deployed, the next step to talk a bit about container image security, starting with the images used to bootstrap this cluster.

## Expected results

Your cluster is about to be bootstrapped with some base operating container images. These components will place your cluster under GitOps control and will be foundational security elements and any other cluster-wide resources you want deployed before workloads start landing on the cluster. This means this is the first time we'll be bringing images directly into this cluster.

You'll end up with the following images imported into your ACR instance, after having passed through a (partly) simulated quarantine process.

* Flux (kustomize-controller, source-controller)
* Flaco
* Busybox
* Kured
* Envoy

## Quarantine pattern

Quarantining first and third party images is a recommended security practice. This allows you to get your images onto a dedicated container registry and subject them to any sort of security/compliance scrutiny you wish to apply. Once validated, they can then be promoted to being available to your cluster. There are many variations on this pattern, with different tradeoffs for each. For simplicity in this walkthrough we are simply going to import our images to repository names that starts with `quarantine/`. We'll then show you Azure Security Center's scan of those images, and then you'll import those same images directly from `quarantine/` to `live/` repositories. We've restricted our cluster to only allow pulling from `live/` repositories and we've built an alert if an image was imported to `live/` from a source other than `quarantine/`. To be clear, this isn't a preventative security control; _this won't block a direct import_ request or _validate_ that the image actually passed quarantine checks. As mentioned, there are other solutions you can use for this pattern that are more exhaustive. [Aquasec](https://go.microsoft.com/fwlink/?linkid=2002601&clcid=0x409) and [Twistlock](https://go.microsoft.com/fwlink/?linkid=2002600&clcid=0x409) both offer integrated solutions specifically for Azure Container Registry scanning and compliance management. Azure Container Registry has an [integrated quarantine feature](https://docs.microsoft.com/azure/container-registry/container-registry-faq#how-do-i-enable-automatic-image-quarantine-for-a-registry) as well that could be considered, however it is in preview at this time.

## Deployment pipelines

Your deployment pipelines are one of the first lines of defense in container image security. Shifting left by introducing build steps like [GitHub Image Scanning](https://github.com/Azure/container-scan) (which leverages common tools like [dockle](https://github.com/goodwithtech/dockle) and [Aquasec trivy](https://github.com/aquasecurity/trivy)) will help ensure that, at build time, your images are linted, CIS benchmarked, and free from known vulnerabilities. You can use any tooling at this step that you trust, including paid, ISV solutions that help provide your desired level of confidence and compliance.

Once your images are built (or identified from a public container registry such as Docker Hub or GitHub Container Registry), the next step is pushing/importing those images to your own container registry. This is the next place a security audit should take place and is in fact the quarantine process identified above. Your newly pushed images undergo any scanning desired. Your pipeline should be gated on the outcome of that scan. If the scan is complete and returned healthy results, then pipeline should move the image into your final container registry repository. If the scan does not complete or is not found healthy, you stop that deployment immediately.

## Continuous scanning

The quarantine pattern is ideal for detecting issues with newly pushed images, but _continuous scanning_ is also desirable as CVEs can be found at any time for your images that are in use. Azure Defender for container registries will perform daily scanning of active images (based on images that have been recently pulled). Third party ISV solutions can perform similar tasks. It is recommended that you implement continuous scanning at the registry level. Azure Defender for container registries currently has limitations with private Azure Container Registry instances (such as yours, exposed exclusively via Private Link). Ensure your continuous scan solution can work within your network restrictions. You may need to bring a third-party ISV solution into network adjacency to your container registry to be able to perform your desired scanning. How you react to a CVE (or other issue) detected in your image, will be based on your policies. For example, you could remove the image from being available to pull, but that could cause a downstream outage while you're remediating. Document a strategy on how you plan to deal with situations like this.

## In-cluster scanning

Using a security agent that is container-aware and can operate from within the cluster is another layer of defense, and is the closest to the actual runtime. This should be used to detect threats that were not detectable in an earlier stage. Having an in-cluster runtime security agent be your _only_ issue detection point is _too late_ in the process. These agents come at a cost (compute power, operational complexity, their own security posture), but are often times still considered a valuable addition to your overall defense in depth position. This topic is touched on a bit more on the next page.

**Static analysis, registry scanning, and continuous scanning should be the workflow for all of your images; both your own first party and any third party images you use.**

## Steps

1. Quarantine Flux and other public baseline security/utility images.

   ```bash
   # Get your Quarantine Azure Container Registry service name
   # You only deploy one ACR instance, but this could be a separate, dedicated quarantine instance if desired.
   ACR_NAME_QUARANTINE=$(az deployment group show -g rg-bu0001a0005 -n cluster-stamp --query properties.outputs.containerRegistryName.value -o tsv)
   
   # [Combined this takes about two minutes.]
   az acr import --source ghcr.io/fluxcd/kustomize-controller:v0.6.3 -t quarantine/fluxcd/kustomize-controller:v0.6.3 -n $ACR_NAME_QUARANTINE
   az acr import --source ghcr.io/fluxcd/source-controller:v0.6.3 -t quarantine/fluxcd/source-controller:v0.6.3 -n $ACR_NAME_QUARANTINE
   az acr import --source docker.io/falcosecurity/falco:0.26.2 -t quarantine/falcosecurity/falco:0.26.2 -n $ACR_NAME_QUARANTINE
   az acr import --source docker.io/library/busybox:1.33.0 -t quarantine/library/busybox:1.33.0 -n $ACR_NAME_QUARANTINE
   az acr import --source docker.io/weaveworks/kured:1.6.1 -t quarantine/weaveworks/kured:1.6.1 -n $ACR_NAME_QUARANTINE
   az acr import --source docker.io/envoyproxy/envoy-alpine:v1.15.0 -t quarantine/envoyproxy/envoy-alpine:v1.15.0 -n $ACR_NAME_QUARANTINE
   ```

   > For simplicity we are NOT importing images that are coming from Microsoft Container Registry (MCR). This is not an endorsement of the suitability of those images to be pulled without going through quarantine or depending public container registries for production runtime needs. All container images that you bring to the cluster should pass through this quarantine step. For transparency, images that we skipped importing are for [Open Service Mesh](./cluster-manifests/cluster-baseline-settings/osm/) and [CSI Secret Store](./cluster-manifests/cluster-baseline-settings/secrets-store-csi/). Both of these are [progressing to eventually be AKS add-ons in the cluster](https://aka.ms/aks/roadmap), and as such would have been pre-deployed to your cluster like other add-ons (E.g. Azure Policy and Azure Monitor) so you wouldn't need to bootstrap the cluster with them yourself. We recommend you do bring these into this import process, and once you've done that you can update the Azure Policy `allowedContainerImagesRegex` to remove `mcr.microsoft.com/.+` as a valid source of images, leaving just `<your acr instance>/live/.+` as the only valid source.

1. Run security audits on images.

   If you had sufficient permissions when we did [subscription configuration](./04-subscription.md), Azure Defender for container registries is enabled on your subscription. Azure Defender for container registries will begin [scanning all newly imported images](https://docs.microsoft.com/azure/security-center/defender-for-container-registries-introduction#when-are-images-scanned) in your Azure Container Registry using a Microsoft hosted version of Qualys. The results of those scans will be available in Azure Security Center within 15 minutes. _If Azure Defender for container registries is not enabled on your subscription, you can skip this step._

   To see the scan results in Azure Security Center, perform the following actions:

   1. Open the [Azure Security Center's **Recommendations** page](https://portal.azure.com/#blade/Microsoft_Azure_Security/SecurityMenuBlade/5).
   1. Under **Controls** expand **Remediate vulnerabilities**.
   1. Click on **Vulnerabilities in Azure Container Registry images should be remediated (powered by Qualys)**.
   1. Expand **Affected resources**.
   1. Click on your ACR instance name.

   In here, you can see which container images are **Unhealthy** (had a scan detection), **Healthy** (was scanned, but didn't result in any alerts), and **Unverified** (was unable to be scanned). Unfortunately, Azure Defender for container registries is [unable to scan all container types](https://docs.microsoft.com/azure/security-center/defender-for-container-registries-introduction#availability). Also, because your container registry is exposed exclusively through Private Link, you won't get a list of those Unverified images listed here. Azure Defender for container registries is only full-featured with non-network restricted container registries.

   As with any Azure Security Center product, you can set up alerts or via your connected SIEM to be identified when an issue is detected. Periodically checking and discovering security alerts via the Azure Portal is not the expected method to consume these security status notifications.

   **There is no action for you to take, in this step.** This was just a demonstration of Azure Security Center's scanning features. Ultimately, you'll want to build a quarantine pipeline that solves for your needs and aligns with your image deployment strategy.

1. Release images from quarantine.

   ```bash
   # Get your Azure Container Registry service name
   ACR_NAME=$(az deployment group show -g rg-bu0001a0005 -n cluster-stamp --query properties.outputs.containerRegistryName.value -o tsv)
   
   # [Combined this takes about two minutes.]
   az acr import --source quarantine/fluxcd/kustomize-controller:v0.6.3 -r $ACR_NAME_QUARANTINE -t live/fluxcd/kustomize-controller:v0.6.3 -n $ACR_NAME
   az acr import --source quarantine/fluxcd/source-controller:v0.6.3 -r $ACR_NAME_QUARANTINE -t live/fluxcd/source-controller:v0.6.3 -n $ACR_NAME
   az acr import --source quarantine/falcosecurity/falco:0.26.2 -r $ACR_NAME_QUARANTINE -t live/falcosecurity/falco:0.26.2 -n $ACR_NAME
   az acr import --source quarantine/library/busybox:1.33.0 -r $ACR_NAME_QUARANTINE -t live/library/busybox:1.33.0 -n $ACR_NAME
   az acr import --source quarantine/weaveworks/kured:1.6.1 -r $ACR_NAME_QUARANTINE -t live/weaveworks/kured:1.6.1 -n $ACR_NAME
   az acr import --source quarantine/envoyproxy/envoy-alpine:v1.15.0 -r $ACR_NAME_QUARANTINE -t live/envoyproxy/envoy-alpine:v1.15.0 -n $ACR_NAME
   ```

   > You've deployed an alert that will fire if you import an image directly to `live/` without coming from `quarantine/`. If you'd like to see that trigger, go ahead and import some other image directly to `live/`. Within ten minutes, you should see [a related alert in the Azure Portal](https://portal.azure.com/#blade/Microsoft_Azure_Monitoring/AlertsManagementSummaryBlade).

### Next step

:arrow_forward: [Place the cluster under GitOps management](./11-gitops.md)
