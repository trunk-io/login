# trunk-io/login

Log a GitHub Actions job in to [Trunk](https://trunk.io) with the run's own GitHub credential.
There is no Trunk secret to create, store or rotate, and it works on pull requests from forks,
where repository secrets are not available.

The action installs the `trunk` CLI, puts it on `PATH`, and runs `trunk auth login --github-actions`.
Later steps in the same job run `trunk` commands as that login.

## Usage

```yaml
on:
  pull_request:
  merge_group:

permissions:
  contents: read
  id-token: write # the run's OIDC token, for pull requests from this repository

jobs:
  impacted-targets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # ... compute the impacted targets into targets.txt, one per line ...

      - uses: trunk-io/login@v1

      - run: trunk mergequeue upload-impacted-targets --targets-file targets.txt
        env:
          GITHUB_TOKEN: ${{ github.token }} # lets a long fork-PR job renew its login
```

The login is good only for uploading impacted targets for this repository. Every other Trunk API
refuses it.

## How it authenticates

| Run                                                            | Credential                                    | Needs                                                                                   |
| -------------------------------------------------------------- | --------------------------------------------- | --------------------------------------------------------------------------------------- |
| `pull_request` from this repository, `merge_group`, `push`, `workflow_dispatch` | The run's GitHub OIDC token                   | `permissions: id-token: write`                                                           |
| `pull_request` from a fork (and `pull_request_target`)          | The job's `GITHUB_TOKEN`, bound to that PR and its head commit | An organization admin turns on **Fork PR CI access** for the repository in Trunk (Settings → Repositories) |

- The CLI picks the credential. It sends the `GITHUB_TOKEN` only on a pull request from a fork.
- A fork PR's login is bound to that pull request and its head commit, so it can upload only for
  them.
- Other events, such as `issue_comment`, `workflow_run` and `schedule`, are refused: some of them
  let someone outside the repository run code with the repository's OIDC token.
- GitHub may hold a first-time contributor's run until a maintainer approves it.

## Inputs

| Input          | Default                   | Description                                                                                             |
| -------------- | ------------------------- | ------------------------------------------------------------------------------------------------------- |
| `api-url`      | `https://api.trunk.io/v2` | The Trunk API root.                                                                                     |
| `cli-version`  | pinned per release        | The `trunk` CLI version to install.                                                                     |
| `cli-channel`  | `prod`                    | The release channel `cli-version` is published on.                                                      |
| `github-token` | `${{ github.token }}`     | The job's GitHub token. Sent to Trunk only on a pull request from a fork.                               |

## Security

- Nothing is exported to the job's environment or to later steps' inputs. The login is saved to the
  CLI's credential file (mode `0600`) and used only by the upload.
- The login expires after at most 15 minutes. The CLI renews it when it is close to expiring or
  refused. That needs the run's credential in the step doing the renewing: the OIDC token is
  available to every step with `id-token: write`, but a fork PR's step needs `GITHUB_TOKEN` in its
  `env`, as in the example above.
- Each login is tied to the run that made it. A later job on a reused self-hosted runner does not
  pick it up.

## Requirements

- Linux (x64 or arm64) or macOS (arm64) runners. Windows is not supported.
- The repository is connected to Trunk: the Trunk GitHub App is installed on it.

## Releasing (maintainers)

Only repository admins can create, move or delete `v*` tags: whoever can move `v1` runs code in
every workflow that uses it. To release:

1. Bump the `cli-version` default in `action.yaml` if a newer `trunk` CLI should ship, and merge.
2. Tag the merged commit with an exact version, then move the major tag to it:

   ```sh
   git tag v1.0.1 <commit> && git push origin v1.0.1
   git tag -f v1 v1.0.1 && git push --force origin v1
   ```

## License

[MIT](LICENSE)
