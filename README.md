# konflux-opentelemetry

This repository contains Konflux configuration to build Red Hat build of OpenTelemetry.

## Build locally

```bash
docker login brew.registry.redhat.io -u
docker login registry.redhat.io -u

podman build -t docker.io/user/otel-operator:$(date +%s) -f Dockerfile.operator 
```

## Release

Open PR `Release - update bundle version` and update [patch_csv.yaml](./bundle-patch/patch_csv.yaml) by submitting a PR with follow-up changes:
1. `spec.version` with the current version e.g. `opentelemetry-operator.v0.108.0`
1. `spec.name` with the current version e.g. `opentelemetry-operator.v0.108.0`
1. `spec.replaces` with [the previous shipped version](https://catalog.redhat.com/software/containers/rhosdt/opentelemetry-operator-bundle/615618406feffc5384e84400) of CSV e.g. `opentelemetry-operator.v0.107.0-4`
1. `metadata.extra_annotations.olm.skipRange` with the version being productized e.g. `'>=0.33.0 <0.108.0'`

Once the PR is merged and bundle is built. Open another PR `Release - update catalog` with:
 * Updated [catalog template](./catalog/catalog-template.json) with the new bundle (get the bundle pullspec from [Konflux](https://console.redhat.com/application-pipeline/workspaces/rhosdt/applications/otel/components/otel-bundle)):
    ```bash
    opm alpha render-template basic catalog/catalog-template.json > catalog/opentelemetry-product/catalog.json && \
    opm validate catalog/opentelemetry-product/ 

    # This does not generate valid catalog, e.g. it is smaller and missing relatedImages
    docker run --rm -it -v $(pwd)/catalog:/tmp:Z  --entrypoint /bin/bash registry.redhat.io/openshift4/ose-operator-registry-rhel9:v4.16
    opm alpha render-template basic /tmp/catalog-template.json > /tmp/opentelemetry-product/catalog-ose-operator.json
    ```

(TODO verify) After konflux builds the bundle create one more PR to change the registry to `registry.redhat.io` see https://konflux-ci.dev/docs/advanced-how-tos/releasing/maintaining-references-before-release/

## Test locally

Images can be found at https://quay.io/organization/redhat-user-workloads

### Deploy bundle

```bash
operator-sdk olm install 
operator-sdk run bundle quay.io/redhat-user-workloads/rhosdt-tenant/otel/otel-bundle@sha256:a09e1fa7c42b3f89b8a74e83d9d8c5b501ef9cd356612d6e146646df1f3d5800
operator-sdk cleanup opentelemetry-product
```

### Deploy catalog

Get catalog for specific version from [Konflux](https://console.redhat.com/application-pipeline/workspaces/rhosdt/applications/otel-fbc-v4-15/components/otel-fbc-v4-15)

```yaml
kubectl apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
   name: konflux-catalog-otel
   namespace: openshift-marketplace
spec:
   sourceType: grpc
   image: quay.io/redhat-user-workloads/rhosdt-tenant/otel/otel-fbc-v4-15@sha256:337009c69204eed22bd90acf5af45f3db678bd65531c8847c59e9532f8427d29
   displayName: Konflux Catalog OTEL
   publisher: grpc
EOF

kubectl get pods -w -n openshift-marketplace
kubectl delete CatalogSource konflux-catalog-otel -n openshift-marketplace
```

`Konflux catalog OTEL` menu should appear in the OCP console under Operators->OperatorHub.

### Inspect bundle image

```bash
mkdir /tmp/bundle
docker image save -o /tmp/bundle/image.tar quay.io/redhat-user-workloads/rhosdt-tenant/otel/otel-bundle@sha256:193358e912cd6a1d06eacf27363d85f2082c21596084110f026f43682ca3cecf
tar xvf /tmp/bundle/image.tar -C /tmp/bundle
tar xvf /tmp/bundle/c6f6e1b5441a6acfc03bb40f4b2d47b98dcfca1761e77e47fba004653eb596d7/layer.tar -C /tmp/bundle/c6f6e1b5441a6acfc03bb40f4b2d47b98dcfca1761e77e47fba004653eb596d7
```

### Inspect multi-arch image

The pinned image pullspec in [update-bundle.sh](bundle-patch/update_bundle.sh) should be image index digest.
The `skopeo` should return a list of manifests. 

```bash
skopeo inspect --raw docker://quay.io/redhat-user-workloads/rhosdt-tenant/otel/operator@sha256:2a8b137c4b9774405a84c4719da6162a56cb97761dce68e59a0d2ed974fae1f0  | jq                                                                                                                                                                                                  ploffay@fedora
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.index.v1+json",
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:5c7b1445c7d1f170bdcdcb814c7015a898d861807992fe61f8c36b8fe7ebfb3f",
      "size": 947,
      "platform": {
        "architecture": "amd64",
        "os": "linux"
      }
    },
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:b69e51d647805347e8e55b16cb2114a8bfc240ac4f91db71e21b78af012f2817",
      "size": 947,
      "platform": {
        "architecture": "arm64",
        "os": "linux"
      }
    },
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:761683f288d76d24d75699da56981d3baa9d24883b3bc19f67821d8f6d766321",
      "size": 947,
      "platform": {
        "architecture": "ppc64le",
        "os": "linux"
      }
    },
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:c1f79211f283a60ca066a4333cc904aeeca35d3aaf351217631d6a21aa58ce18",
      "size": 947,
      "platform": {
        "architecture": "s390x",
        "os": "linux"
      }
    }
  ]
}
```

The `sosign` returns list of platforms even for no image index digest:

```bash
cosign download attestation quay.io/redhat-user-workloads/rhosdt-tenant/otel/operator@sha256:5c7b1445c7d1f170bdcdcb814c7015a898d861807992fe61f8c36b8fe7ebfb3f | jq -r '.payload | @base64d | fromjson | .predicate.invocation.parameters'                                                                                                                        127 ↵ ploffay@fedora
{
  "build-args": [],
  "build-args-file": "",
  "build-image-index": "true",
  "build-platforms": [
    "linux/x86_64",
    "linux/arm64",
    "linux/ppc64le",
    "linux/s390x"
  ],
  "build-source-image": "false",
  "dockerfile": "Dockerfile.operator",
  "git-url": "https://github.com/pavolloffay/konflux-opentelemetry",
  "hermetic": "false",
  "image-expires-after": "",
  "output-image": "quay.io/redhat-user-workloads/rhosdt-tenant/otel/operator:00ebe9a6475bda2c3f7be7278841de0d7d81feab",
  "path-context": ".",
  "prefetch-input": "",
  "rebuild": "false",
  "revision": "00ebe9a6475bda2c3f7be7278841de0d7d81feab",
  "skip-checks": "false"
}

skopeo inspect --raw docker://quay.io/redhat-user-workloads/rhosdt-tenant/otel/operator@sha256:5c7b1445c7d1f170bdcdcb814c7015a898d861807992fe61f8c36b8fe7ebfb3f | jq                                                                                                                                                                                             127 ↵ ploffay@fedora
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "digest": "sha256:5bc060cce164bec15ed725164a272c7aa670a211dadab60d4b4ce1a63cb6ba9e",
    "size": 8317
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:2384c7c17092245bda9218fee9b2ae475ee8a53cd8a66e63c1d5f37433276ff0",
      "size": 39365328
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:978e7c294b1c89f7c5f330764d97a80007288964bfd640388024afcd0387dc91",
      "size": 45065115
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:326159d96fc4212c9c599da6c577e490fb5c87c0f6d51ac3203e69f57c60ccbb",
      "size": 99814
    }
  ],
  "annotations": {
    "org.opencontainers.image.base.digest": "sha256:11bb492c19d974e6f67be661e76691e977184e98aff1cfad365363ae9055cff0",
    "org.opencontainers.image.base.name": "registry.redhat.io/ubi8/ubi-minimal:8.10-1052.1724178568"
  }
}
```