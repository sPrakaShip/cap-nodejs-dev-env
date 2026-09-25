# CAP Node.js dev container

This folder defines a **development container**: a Docker image with every
tool needed to build SAP CAP (Cloud Application Programming Model) apps on
Node.js. VS Code runs inside that container, so you don't have to install
Node.js, the CDS tools or the Cloud Foundry CLI on your own machine.

## What this setup gives you

- **Nothing to install on your machine** except VS Code, the Dev Containers
  extension and Docker Desktop (or Podman).
- **The same setup for everyone.** Each person, and each GitHub Codespace,
  gets the same Node version, the same tools and the same VS Code extensions.
- **Separation from your machine.** Projects don't clash over Node versions or
  global npm packages. If the container breaks, rebuild it.
- **Everything needed for CAP**, from `cds init` to `cf push`.

## How it works

```text
devcontainer.json  ──►  builds the image from Dockerfile
                   ──►  starts a container from that image
                   ──►  mounts your project folder at /workspaces/<folder-name>
                   ──►  installs VS Code Server and the listed extensions in the container
                   ──►  forwards ports so http://localhost:4004 works on your machine
```

Your source files stay on your machine; the container reads and writes them
through the mount. Anything installed inside the container only (for example
a global npm package installed from the terminal) is lost on rebuild unless
you add it to the Dockerfile.

## The files

### `Dockerfile`: what goes into the image

| Section | What it does | Why |
|---|---|---|
| `# syntax=docker/dockerfile:1` | Uses the current Dockerfile syntax | Makes BuildKit features such as `TARGETARCH` available |
| `ARG VARIANT="24"` + `FROM mcr.microsoft.com/devcontainers/javascript-node:${VARIANT}` | Starts from Microsoft's Node.js dev container image, Node 24 by default | This base image already has Node, npm, git, zsh, and a non-root user `node` with `sudo` rights. Change `VARIANT` to change the Node version |
| `apt-get install ...` | Installs general command-line tools (see below) | Handy when working on CAP |
| `rm -rf /var/lib/apt/lists/*` | Deletes the apt package index after installing | Makes the image smaller |
| `ARG TARGETARCH` + `curl ... \| tar ...` | Downloads the cf CLI v8 and puts `cf8` and `cf` in `/usr/local/bin` | Needed to deploy to SAP BTP Cloud Foundry. Uses the release tarball because the apt repo's signing key is broken (see [Troubleshooting](#container-build-fails-at-apt-get-update-cloud-foundry-repo-not-signed)). `TARGETARCH` picks the amd64 or arm64 build |
| `&& cf version` | Checks that the cf CLI runs | The build fails right away if the download breaks, instead of failing later |
| `USER node` | Runs the remaining steps as the non-root user | Global npm packages then belong to `node`, the user you work as |
| `npm install -g @sap/cds-dk@latest` | Installs the CAP development kit globally | Provides the `cds` command: `cds init`, `cds watch`, `cds add`, `cds deploy` and so on |
| `WORKDIR /home/node` | Sets the default working directory | Somewhere sensible to start from |

The command-line tools installed with apt:

| Tool | What it's for in CAP work |
|---|---|
| `curl` | Calling your OData services from the terminal; downloading files |
| `entr` | Re-running a command whenever files change, e.g. `ls *.cds \| entr cds compile srv` |
| `git` | Version control |
| `sqlite3` | Looking inside the SQLite database CAP uses in development (`sqlite3 db.sqlite`) |
| `watch` | Re-running a command on a timer, e.g. `watch -n 2 cf apps` |
| `yq` | Reading and editing YAML such as `mta.yaml` from the command line |

### `devcontainer.json`: how VS Code uses the image

| Setting | Value | Meaning |
|---|---|---|
| `name` | `CAP Development Container` | The name shown in VS Code's status bar and menus |
| `build.dockerfile` | `Dockerfile` | Build the image from the Dockerfile in this folder |
| `build.args.VARIANT` | `"24"` | Passed to `ARG VARIANT` in the Dockerfile. **Change the Node version here** |
| `customizations.vscode.extensions` | see below | Extensions installed inside the container automatically |
| `forwardPorts` | `4004, 4005, 4006, 9229` | `4004` is the default `cds watch` port; `4005` and `4006` are for extra services or apps; `9229` is the Node.js debugger port |
| `remoteUser` | `node` | VS Code and its terminals run as `node`, not root |

The extensions:

| Extension | What it's for |
|---|---|
| `sapse.vscode-cds` | CDS language support: syntax highlighting, code completion, error checking |
| `sapse.vscode-wing-cds-editor-vsc` | Graphical CDS model editor |
| `sapse.vsc-extension-odata-csdl-modeler` | Diagram view of OData service metadata |
| `saposs.xml-toolkit` | Editing XML, such as Fiori views and annotations |
| `dbaeumer.vscode-eslint` | Checking JavaScript code for problems |
| `mechatroner.rainbow-csv` | Easier reading of CSV files, which CAP uses for initial data in `db/data/*.csv` |
| `humao.rest-client` | Sending HTTP requests from `.http` files to test your services |

## Using it

### First time

1. Install VS Code, the
   [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)
   extension and Docker Desktop.
2. Open the project folder in VS Code.
3. When VS Code offers to **Reopen in Container**, accept. Otherwise run
   **Dev Containers: Reopen in Container** from the command palette
   (`Ctrl+Shift+P`).
4. The first build takes a few minutes. Later starts are much quicker.
5. Open a terminal in VS Code. You are now inside the container:

   ```sh
   node --version
   cds version
   cf version
   ```

### Creating and running a CAP project

```sh
cds init my-app          # create a new CAP project
cd my-app
cds add sample           # optional: add sample data and services
npm install
cds watch                # start the server; VS Code forwards port 4004
```

Open <http://localhost:4004> in your browser.

### Deploying to Cloud Foundry

```sh
cf login -a https://api.cf.<region>.hana.ondemand.com --sso
cds add mta              # generate mta.yaml
```

Building MTA archives with `mbt` needs one more tool, see
[Adding tools](#adding-tools) below.

### After changing the Dockerfile or devcontainer.json

Run **Dev Containers: Rebuild Container**. Changes don't take effect until you
rebuild.

## Using this in future projects

### Option 1: copy the folder into each project (recommended)

Copy this whole `.devcontainer` folder into the root of the new project's repo
and commit it:

```text
my-new-cap-project/
├── .devcontainer/
│   ├── devcontainer.json
│   ├── Dockerfile
│   └── README.md
├── app/
├── db/
├── srv/
└── package.json
```

Anyone who clones the repo, or opens it in GitHub Codespaces, gets the same
environment. Change the `name` in `devcontainer.json` so you can tell projects
apart.

### Option 2: one environment for several projects

Keep this repo as your "workbench". Open it in the container and create or
clone projects in subfolders of it. This is quicker for experiments, but
every project shares one environment, and teammates won't get the container
when they clone just one project.

### Adapting it to a project

Make these changes as needed, then rebuild.

#### Changing the Node version

In `devcontainer.json`:

```json
"args": { "VARIANT": "22" }
```

Use a Node version that the
[CAP docs](https://cap.cloud.sap/docs/get-started/) support.

#### Pinning tool versions

`@sap/cds-dk@latest` gets whatever version is newest when the image is built,
so two people can end up with different versions. For team projects, pin the
version:

```dockerfile
RUN npm install -g @sap/cds-dk@9
```

Or leave `cds-dk` out of the Dockerfile, add it to the project's
`devDependencies`, and run it with `npx cds`.

#### Adding tools

- **System packages:** add them to the `apt-get install` list.
- **Global npm tools:** add them to the `npm install -g` line, after
  `USER node`. Common ones for CAP and BTP work:

  ```dockerfile
  RUN npm install -g \
    @sap/cds-dk@latest \
    mbt \
    @ui5/cli \
    yo @sap/generator-fiori
  ```

  `mbt` builds MTA archives, `@ui5/cli` is for UI5 apps, and
  `yo @sap/generator-fiori` generates Fiori apps.
- **cf CLI plugins**, such as the MTA plugin for `cf deploy`. Add this after
  `USER node` so the plugin belongs to the user you work as:

  ```dockerfile
  RUN cf install-plugin multiapps -f
  ```

- **Dev container features** (ready-made install scripts). Add them in
  `devcontainer.json`:

  ```json
  "features": {
    "ghcr.io/devcontainers/features/github-cli:1": {},
    "ghcr.io/devcontainers/features/docker-in-docker:2": {}
  }
  ```

  Browse the full list at <https://containers.dev/features>.

#### Installing project dependencies automatically

Add this to `devcontainer.json` to run `npm install` each time the container
is created:

```json
"postCreateCommand": "npm install"
```

#### Adding extensions or ports

Add extension IDs to `customizations.vscode.extensions`. An extension's ID is
shown on its Marketplace page and in the Extensions view. Add ports your
project uses to `forwardPorts`, for example `8080` for an approuter.

### Things to avoid

- **Don't put credentials in the Dockerfile or devcontainer.json.** Both files
  are committed, and anything in the image can be read by anyone who has the
  image. Log in to Cloud Foundry after the container starts, or use
  environment variables that aren't committed.
- **Don't install tools only from the container terminal** if the project
  depends on them. They disappear on rebuild. Put them in the Dockerfile.
- **Don't run as root.** Keep `USER node` before the npm steps and
  `remoteUser: node`, or files you create can end up owned by root.

## Troubleshooting

### Container build fails at `apt-get update` (Cloud Foundry repo not signed)

#### Symptom

Building or rebuilding the dev container fails at the `apt-get update` step
with output like this:

```text
Err:3 https://cf-cli-debian-repo.s3.amazonaws.com stable InRelease
  Sub-process /usr/bin/sqv returned an error code (1), error message is:
  Signing key on C19C04748BF33B2E17863557172B5989FCD21EF8 is bad:
  The primary key is not live because: Expired on 2020-09-12T18:17:33Z
...
E: The repository 'https://packages.cloudfoundry.org/debian stable InRelease' is not signed.
ERROR: failed to build: ... did not complete successfully: exit code: 100
```

#### Cause

Earlier versions of the [Dockerfile](Dockerfile) installed the cf CLI
(`cf8-cli`) from the Cloud Foundry apt repository, and added that repository's
signing key to the image.

- The primary key of that signing key expired in 2020. Only its subkeys have
  been extended since then.
- The base image `mcr.microsoft.com/devcontainers/javascript-node` is now built
  on Debian 13 (trixie). On trixie, apt checks repository signatures with
  Sequoia's `sqv` instead of `gpgv`.
- `sqv` is stricter than `gpgv` and rejects signatures that depend on an
  expired primary key. apt then treats the whole repository as unsigned and
  `apt-get update` fails.

The key-download step can show as `CACHED` and look fine. The key itself is
the problem, not the download.

#### Fix

The Dockerfile no longer uses the Cloud Foundry apt repository. The cf CLI v8
now comes from the official release tarball:

```dockerfile
ARG TARGETARCH
RUN case "${TARGETARCH:-amd64}" in \
    amd64) CF_RELEASE=linux64-binary ;; \
    arm64) CF_RELEASE=linuxarm64-binary ;; \
    *) echo "Unsupported architecture: ${TARGETARCH}" && exit 1 ;; \
  esac \
  && curl -fsSL "https://packages.cloudfoundry.org/stable?release=${CF_RELEASE}&version=v8&source=github" \
  | tar -xz -C /usr/local/bin cf8 cf \
  && cf version
```

- The tarball has a `cf8` binary and a `cf` link to it. Both go into
  `/usr/local/bin`.
- `TARGETARCH` is set automatically by BuildKit, so the build works on both
  amd64 and arm64 hosts.
- `cf version` runs at the end, so the build stops right away if the download
  ever breaks.
- `cf8-cli` has been removed from the `apt-get install` list, and the keyring
  and apt source steps have been removed.

To apply the fix, run **Dev Containers: Rebuild Container** from the VS Code
command palette.

If Cloud Foundry publishes a new signing key with a valid primary key, you
could switch back to the apt repository. The tarball install works either way.

### "No space left on device" while installing VS Code Server

#### Symptom

Near the start of the Dev Containers log you may see something like this:

```text
Start: Run: wsl -d docker-desktop -e /bin/sh -c ...
tar: can't make dir vscode-server-linux-x64/node_modules/node-pty: No space left on device
Could not connect to WSL.
```

#### Cause

On Windows with Docker Desktop, the Dev Containers extension first tries to
put VS Code Server inside Docker Desktop's own internal WSL distro
(`docker-desktop`). That distro has very little free space, so the install can
fail.

#### Impact and fix

This error doesn't break the build. The extension moves on and builds the
container normally. If you see this error, look further down the log for the
real cause of a failed build.

If it keeps happening, free up space in Docker Desktop:

```sh
docker system df          # see what is using space
docker image prune -a     # remove images not used by any container
docker volume prune       # remove volumes not used by any container
```

Both prune commands delete data permanently, so check what they will remove
before you run them.
