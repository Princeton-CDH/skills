# Princeton CDH RSE team Skills

This repository provides agentic skills related to Princeton CDH RSE team workflows, based on the [agentskills.io](http://agentskills.io) standard.

## Skills:

- [release-prep](docs/release-prep.md): prepare a software package for a new release, following the first portion of our team software release checklist

## To use:

The skills can be downloaded and installed manually, but we recommend installing and upgrading with [GitHub CLI](https://cli.github.com/):

```console
gh skill install princeton-cdh/skills
```

## Contributor Setup:

[Install uv](https://docs.astral.sh/uv/getting-started/installation) if you don't already have it.

Install `pre-commit` with `uv`, then install configured pre-commit hooks:

```sh
uv tool install pre-commit --with pre-commit-uv
pre-commit install
```

## License

Content is licensed under the [Apache 2.0 License](LICENSE).

(c)2026 Trustees of Princeton University. Permission granted for non-commercial
distribution online under a standard Open Source license.
