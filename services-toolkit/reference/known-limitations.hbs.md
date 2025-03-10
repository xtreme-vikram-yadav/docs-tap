# Known issues and limitations for services on Tanzu Application Platform

This topic describes known issues, limitations, and their workarounds related to working with services on
Tanzu Application Platform (commonly known as TAP).
This includes information related to Services Toolkit, Bitnami Services, and Crossplane.

For further troubleshooting guidance, see
[Troubleshoot Services Toolkit](../how-to-guides/troubleshooting.hbs.md).

## <a id="limitations"></a> Limitations

This section details the limitations when using services on Tanzu Application Platform.

### <a id="multi-workloads"></a> Cannot claim and bind to the same service instance from across multiple namespaces

**Description:**

Two or more workloads located in two or more distinct namespaces cannot claim and bind to the same
service instance.
This is due to the mutually exclusive nature of claims. After a claim has claimed a service instance,
no other claim can then claim that same service instance.

This limitation does not exist for two or more workloads located in the same namespace.
In this case, the workloads can all still all bind to one claim.
This is not possible across the namespace divide.

### <a id="too-many-crds"></a> Cluster performance degradation due to large number of CRDs

**Description:**

While not strictly a limitation with Services Toolkit, take care before choosing to
install additional Crossplane `Providers` into Tanzu Application Platform.
Some of these `Providers` install hundreds of additional CRDs into the cluster.

This is particularly true of the `Providers` for AWS, Azure, and GCP.
For the number of CRDs installed with these `Providers`, see:

- [provider-aws CRDs](https://marketplace.upbound.io/providers/upbound/provider-aws/latest/managed-resources)
- [provider-azure CRDs](https://marketplace.upbound.io/providers/upbound/provider-azure/latest/managed-resources)
- [provider-gcp CRDs](https://marketplace.upbound.io/providers/upbound/provider-gcp/latest/managed-resources)

You must ensure that your cluster has sufficient resource to support this number of additional CRDs
if you choose to install them.

**Workaround:**

To address this issue, Upbound have released a new feature named Provider Families.
For more information, see the [Crossplane blog](https://blog.crossplane.io/crd-scaling-provider-families/).

## <a id="known-issues"></a> Known issues

This section details the known issues related to using services on Tanzu Application Platform.

### <a id="compositeresourcedef"></a> Unexpected error if `additionalProperties` is `true` in a CompositeResourceDefinition

**Description:**

When creating a `CompositeResourceDefinition`, if you set `additionalProperties: true` in the
`openAPIV3Schema` section, an error occurs during the validation step of the creation of any
`ClassClaim` that refers to a class that refers to the `CompositeResourceDefinitions`.

**Error:**

```console
json: cannot unmarshal bool into Go struct field JSONSchemaProps.AdditionalProperties of type apiextensions.JSONSchemaPropsOrBool
```

**Workaround:**

Rather than setting `additionalProperties: true`, you can set `additionalProperties: {}` instead.
This has the same effect but does not cause unexpected errors.

### <a id="private-reg"></a> Private registry or VMware Application Catalog configuration does not take effect

**Description:**

As of Tanzu Application Platform v1.5.0, there is a known issue that occurs if you try to
[configure private registry integration for the Bitnami services](../../bitnami-services/how-to-guides/configure-private-reg-integration.hbs.md)
after having already created a claim for one or more of the Bitnami services using the default configuration.

The issue is that the updated private registry configuration does not appear to take effect.
This is due to caching behavior in the system which is not currently accounted for during configuration
updates.

**Workaround:**

Delete the `provider-helm-*` pods in the `crossplane-system` namespace and wait for new pods to come
back online after having applied the updated registry configuration.

### <a id="cp-custom-cert-inject"></a>Crossplane Providers cannot communicate with systems using a custom CA

**Description:**

A known issue occurs when a Crossplane Provider needs to communicate with systems that are set up
with a custom CA.
For example, when the Crossplane Helm Provider uses `releases.helm.crossplane.io` to try to pull a
Helm chart from a registry that uses a custom CA, you see that:

- The `releases.helm.crossplane.io` never reports the status condition `Ready=True`.
- The `releases.helm.crossplane.io` shows a certificate verification error for either:
  - The status condition of type `SYNCED`.
  - The event on the `release.helm.crossplane.io`.

To confirm whether you are affected by this issue:

1. Verify the status by running:

    ```console
    kubectl get releases.helm.crossplane.io
    ```

    Example output if you are affected by the issue:

    ```console
    NAME                CHART   VERSION   SYNCED   READY   STATE   REVISION   DESCRIPTION   AGE
    wordpress-example           15.2.5    False    False                                    7m37s
    ```

1. Find out more about the reason by running the following command, or similar:

    ```console
    kubectl get event --namespace default --field-selector involvedObject.name=wordpress-example,involvedObject.kind=Release,type!=Normal | grep -e 'LAST SEEN' -e 'failed to login'
    ```

    Example output if you are affected by the issue:

    ```console
    LAST SEEN   TYPE      REASON                            OBJECT                      MESSAGE
    3m41s       Warning   CannotCreateExternalResource      release/wordpress-example   failed to install release: failed to login to registry: Get "https://insecure-registry:443/v2/": tls: failed to verify certificate: x509: certificate signed by unknown authority
    ```

This issue occurs because the Providers are installed by Crossplane itself rather than directly by the
Tanzu Application Platform [Crossplane Package](../../crossplane/about.hbs.md).
CA certificate data configured through the `tap-values.yaml` file is not passed down to Crossplane
Providers. Therefore, the Providers are unable to connect to the systems they need to communicate with,
such as, the Helm Provider connecting to a registry hosting the Helm charts or the Kubernetes Provider
connecting to a Kubernetes APIServer. This issue might be resolved in a later release.

**Workaround:**

Use admission control that allows you to mutate objects, in this case pods, and injects the appropriate
CA certificates.
You can use any system that can mutate objects at admission to mutate the Crossplane Provider pods.
For example, to inject CA certificates you can use this sample in the [Kyverno documentation](https://kyverno.io/policies/other/add-certificates-volume/add-certificates-volume/).

> **Note** From Tanzu Application Platform v1.7, the Crossplane Package will inherit this data through
> `shared.ca_cert_data` of `tap-values.yaml` and configure the Crossplane Providers accordingly.
> This workaround will no longer be needed.
