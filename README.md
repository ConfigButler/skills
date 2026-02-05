A small collection of reusable “skills” for AI agents for desiging CRDs.

## Install with `npx skills`

Install everything from this repo:

```bash
npx skills add ConfigButler/skills
```

Install only one skill:

```bash
# install only k8s-crd-design-review (e.g. if you don't use kubebuilder)
npx skills add ConfigButler/skills --only k8s-crd-design-review
```

## Usage

Use the npx skills and check in your agent if the skill shows up in the list of skills. (e.g. kilocode has documented that [here](https://kilo.ai/docs/customize/skills), your own agent should have a similar description).

## Contributing

Bug reports or PRs are welcome, but do make sure to checkout with submodules. For normal usage not needed: but very usefull to check if your addition is in line with the offical Kubernetes docs (see [docs/README.md](docs/README.md).
