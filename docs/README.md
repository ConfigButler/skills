This folder is empty if you did a 'normal' clone. **Fine if you are just using the skills.**

If you want to improve these skills then I suggest you to run `git submodule update --init --recursive` -> it will check out the relevant Kubernetes repos in /docs folder so that you (and your AI agent) have immediate access to relevant parts of the upstream Kubernetes documentation.

Highlights:

* [api-conventions](kubernetes/community/contributors/devel/sig-architecture/api-conventions.md) ([official GitHub source](https://github.com/kubernetes/community/blob/main/contributors/devel/sig-architecture/api-conventions.md#spec-and-status))
* [api-changes](kubernetes/community/contributors/devel/sig-architecture/api_changes.md)
* [custom-resource-definitions](kubernetes/website/content/en/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions.md)
* [custom-resource-definition-versioning](kubernetes/website/content/en/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning.md)
* [kstatus](kubernetes-sig/cli-utils/pkg/kstatus/README.md)
