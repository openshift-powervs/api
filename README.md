# api
The canonical location of the OpenShift API definition.  This repo holds the API type definitions and serialization code used by [openshift/client-go](https://github.com/openshift/client-go)

## RETIRED
This fork is no longer used, as the API changes for Power VS have been merged upstream.

## pull request process

Pull requests that change API types in this repo that have corresponding "internal" API objects in the 
[openshift/origin](https://github.com/openshift/origin) repo must be paired with a pull request to
[openshift/origin](https://github.com/openshift/origin).

To ensure the corresponding origin pull request is ready to merge as soon as the pull request to this repo is merged:
1. Base your pull request to this repo on latest [openshift/api#master](https://github.com/openshift/api/commits/master) and ensure CI is green
2. Base your pull request to openshift/origin on latest [openshift/origin#master](https://github.com/openshift/origin/commits/master)
3. In your openshift/origin pull request:
   1. Add a TMP commit that points [glide.yaml](https://github.com/openshift/origin/blob/master/glide.yaml#L39-L41) at your fork of openshift/api, and the branch of your pull request:

      ```
      - package: github.com/openshift/api
        repo:    https://github.com/<your-username>/api.git
        version: "<your-openshift-api-branch>"
      ```

    2. Update your `bump(*)` commit to include the result of running `hack/update-deps.sh`, which will pull in the changes from your openshift/api pull request
    3. Make sure CI is green on your openshift/origin pull request 
    4. Get LGTM on your openshift/api pull request (for API changes) and your openshift/origin pull request (for code changes)

Once both pull requests are ready, the openshift/api pull request can be merged.

Then do the following with your openshift/origin pull request:
1. Drop the TMP commit (pointing glide back at openshift/api#master)
2. Rerun `hack/update-deps.sh` and update your `bump(*)` commit
3. It can then be tagged and merged by CI

## generating CRD schemas

Since Kubernetes 1.16, every CRD created in `apiextensions.k8s.io/v1` is required to have a [structural OpenAPIV3 schema](https://kubernetes.io/blog/2019/06/20/crd-structural-schema/). The schemas provide server-side validation for fields, as well as providing the descriptions for `oc explain`. Moreover, schemas ensure structural consistency of data in etcd. Without it anything can be stored in a resource which can have security implications. As we host many of our CRDs in this repo along with their corresponding Go types we also require them to have schemas. However, the following instructions apply for CRDs that are not hosted here as well.

These schemas are often very long and complex, and should not be written by hand. For OpenShift, we provide Makefile targets in [build-machinery-go](https://github.com/openshift/build-machinery-go/) which generate the schema, built on upstream's [controller-gen](https://github.com/kubernetes-sigs/controller-tools) tool.

If you make a change to a CRD type in this repo, simply calling `make update-codegen-crds` should regenerate all CRDs and update the manifests. If yours is not updated, ensure that the path to its API is included in our [calls to the Makefile targets](https://github.com/openshift/api/blob/release-4.5/Makefile#L17-L29).

To add this generator to another repo:
1. Vendor `github.com/openshift/build-machinery-go`

2. Update your `Makefile` to include the following:
```
include $(addprefix ./vendor/github.com/openshift/build-machinery-go/make/, \
  targets/openshift/crd-schema-gen.mk \
)

$(call add-crd-gen,<TARGET_NAME>,<API_DIRECTORY>,<CRD_MANIFESTS>,<MANIFEST_OUTPUT>)
```
The parameters for the call are:

1. `TARGET_NAME`: The name of your generated Make target. This can be anything, as long as it does not conflict with another make target. Recommended to be your api name.
2. `API_DIRECTORY`: The location of your API. For example if your Go types are located under `pkg/apis/myoperator/v1/types.go`, this should be `./pkg/apis/myoperator/v1`.
3. `CRD_MANIFESTS`: The directory your CRDs are located in. For example, if that is `manifests/my_operator.crd.yaml` then it should be `./manifests`
4. `MANIFEST_OUTPUT`: This should most likely be the same as `CRD_MANIFESTS`, and is only provided for flexibility to output generated code to a different directory.

You can include as many calls to different APIs as necessary, or if you have multiple APIs under the same directory (eg, `v1` and `v2beta1`) you can use 1 call to the parent directory pointing to your API.

After this, calling `make update-codegen-crds` should generate a new structural OpenAPIV3 schema for your CRDs.

**Notes** 
- This will not generate entire CRDs, only their OpenAPIV3 schemas. If you do not already have a CRD, you will get no output from the generator.
- Ensure that your API is correctly declared for the generator to pick it up. That means, in your `doc.go`, include the following:
  1. `// +groupName=<API_GROUP_NAME>`, this should match the `group` in your CRD `spec`
  2. `// +kubebuilder:validation:Optional`, this tells the operator that fields should be optional unless explicitly marked with `// +kubebuilder:validation:Required`
  
For more information on the API markers to add to your Go types, see the [Kubebuilder book](https://book.kubebuilder.io/reference/markers.html)

### Post-schema-generation Patches

Schema generation features might be limited or fall behind what CRD schemas supports in the latest Kubernetes version.
To work around this, there are two patch mechanisms implemented by the `add-crd-gen` target. Basic idea is that you 
place a patch file next to the CRD yaml manifest with either `yaml-merge-patch` or `yaml-patch` as extension, 
but with the same base name. The `update-codegen-crds` Makefile target will apply these **after** calling 
kubebuilder's controller-gen:

- `yaml-merge-patch`: these are applied via `yq m -x <yaml-file> <patch-file>` compare https://mikefarah.gitbook.io/yq/commands/merge#overwrite-values.
- `yaml-patch`: these are applied via `yaml-patch -o <patch-file> < <yaml-file>` using https://github.com/krishicks/yaml-patch.
