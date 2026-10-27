# vcpkg-workspace.cmake

vcpkg-workspace.cmake prepares the vcpkg workspace of a project without git submodules. It checks out vcpkg and any number of overlay repositories at the refs that the project specifies, and installs the dependencies of the vcpkg manifest of the project with every overlay. It runs on Windows, macOS and Linux with CMake 3.19 or later and git.

## Setup

1. Download the script into the `scripts` directory of the project.

   ```sh
   curl -fsSL -o scripts/vcpkg-workspace.cmake https://raw.githubusercontent.com/stdware/dev-scripts/main/scripts/vcpkg-workspace.cmake
   ```

   ```bat
   powershell -Command "Invoke-WebRequest -OutFile scripts/vcpkg-workspace.cmake https://raw.githubusercontent.com/stdware/dev-scripts/main/scripts/vcpkg-workspace.cmake"
   ```

   The first command runs in a POSIX shell, and the second runs in PowerShell and in cmd. A later version replaces the downloaded script with the following command:

   ```sh
   cmake -P scripts/vcpkg-workspace.cmake self-update
   ```

   The script accepts the download only if the download has a version section. A script of a version before 1.0.0 has no command self-update and is downloaded again.

2. Ignore the script and the local sources in `scripts/.gitignore`, and the dependency directory in the `.gitignore` of the project.

   ```gitignore
   # scripts/.gitignore
   /vcpkg-workspace.cmake
   /vcpkg-sources.local.json
   ```

   ```gitignore
   # .gitignore
   /deps
   ```

3. Write the sources of the project in `scripts/vcpkg-sources.json`, beside the script.

   ```json
   {
       "manifest-dir": "vcpkg-manifest",
       "vcpkg": {
           "repository": "https://github.com/microsoft/vcpkg.git",
           "ref": "master"
       },
       "overlays": [
           {
               "repository": "../../stdware/vcpkg-overlay.git",
               "ref": "main"
           }
       ]
   }
   ```

## Usage

```sh
cmake -P scripts/vcpkg-workspace.cmake install --qt-dir /path/to/Qt/6.11.2/gcc_64
cmake -S . -B build -DCMAKE_TOOLCHAIN_FILE=deps/vcpkg/scripts/buildsystems/vcpkg.cmake
```

| Command | Effect |
|---|---|
| `fetch` | Checks out vcpkg and every overlay, and bootstraps vcpkg if its vcpkg tool is absent or of a release other than the release that the checkout requires |
| `install` | Runs `fetch`, then installs the dependencies with every overlay |
| `status` | Lists the specified and the checked-out commit of every repository |
| `paths` | Prints `VCPKG_ROOT`, `VCPKG_EXECUTABLE`, `VCPKG_INSTALLED_DIR`, `VCPKG_MANIFEST_DIR` and `VCPKG_TOOLCHAIN_FILE` as lines of `<name>=<path>`, and nothing else |
| `run <task>` | Runs a task of the sources |
| `self-update [<ref>]` | Replaces the script with the script at `<ref>` of this repository, a branch, a tag or a commit, `main` by default |
| `version` | Prints the version of the script |
| `help` | Prints the full reference |

Every command accepts `--dry-run`, which prints the git and vcpkg commands without running them. The arguments after `--` are appended to the command of `vcpkg install` or of the task. `cmake -P scripts/vcpkg-workspace.cmake help` prints every option and rule.

## Sources

| Member | Meaning |
|---|---|
| `vcpkg` | The vcpkg repository, with `repository` and `ref`, or a local `path` |
| `overlays` | The overlays in their precedence, each with `repository` and `ref`, or a local `path`. `name` defaults to the last component of the repository or of the path. `ports` and `triplets` name the directories of the overlay, `ports` and `triplets` by default. |
| `manifest-dir` | The directory of `vcpkg.json`, the directory of the sources by default |
| `deps-dir` | The directory into which vcpkg and the overlays are checked out, `deps` in the project by default |
| `clone-args`, `fetch-args` | The arguments of `git clone` and `git fetch`, `["--quiet", "--depth", "1"]` by default, also accepted by vcpkg and by each overlay |
| `tasks` | Named lists of arguments for `run`, in which `${VCPKG_ROOT}`, `${VCPKG_EXECUTABLE}`, `${VCPKG_INSTALLED_DIR}`, `${VCPKG_MANIFEST_DIR}` and `${VCPKG_OVERLAY_ARGS}` are replaced |

Paths in the sources are relative to the file that specifies them.

- A `ref` is a full commit hash, a branch or a tag. A branch and a tag are fetched at each run.
- A `repository` that begins with `./` or `../` is relative to the remote `origin` of the project, as in `.gitmodules`, so that an overlay is fetched over the protocol by which the project was cloned.
- The dependencies are installed into `<vcpkg>/installed`.
- vcpkg is bootstrapped if it has no vcpkg executable, or if the release of the executable differs from `VCPKG_TOOL_RELEASE_TAG` of `scripts/vcpkg-tool-metadata.txt` in vcpkg. The rule also applies to a local vcpkg, for example a submodule.

## Local sources

`scripts/vcpkg-sources.local.json`, if it exists, takes precedence over `scripts/vcpkg-sources.json`. It has the same format, and its members replace those of vcpkg and of the overlays of the same name. A `path` specifies a local directory, which the script uses directly without clone, fetch or checkout:

```json
{
    "overlays": [
        { "name": "vcpkg-overlay", "path": "D:/GitHub/vcpkg-overlay" }
    ]
}
```

`--sources-file <file>` replaces both defaults and may be given more than once, an earlier file taking precedence.

## Qt

The ports that build against Qt find it through the environment variables `QT_DIR` and `Qt6_DIR` or `Qt5_DIR`. `--qt-dir` accepts the prefix of Qt 6 or Qt 5:

```sh
cmake -P scripts/vcpkg-workspace.cmake install --qt-dir /path/to/Qt/6.11.2/gcc_64
```

It also accepts the CMake directory of Qt:

```sh
cmake -P scripts/vcpkg-workspace.cmake install --qt-dir /path/to/Qt/6.11.2/gcc_64/lib/cmake/Qt6
```

Both commands set the following environment variables for vcpkg:

```
QT_DIR=/path/to/Qt/6.11.2/gcc_64/lib/cmake/Qt6
Qt6_DIR=/path/to/Qt/6.11.2/gcc_64/lib/cmake/Qt6
VCPKG_KEEP_ENV_VARS=<VCPKG_KEEP_ENV_VARS of the environment>;QT_DIR;Qt6_DIR
```

`VCPKG_KEEP_ENV_VARS` names the variables that vcpkg keeps in the environment of a port build. On Windows, vcpkg removes the other variables from that environment, apart from a fixed set. Without `--qt-dir`, the environment variable `QT_DIR`, `Qt6_DIR` or `Qt5_DIR`, in this order, takes its place. Without either, the script passes no Qt.

## GitHub Actions

The action `actions/vcpkg-workspace` runs `install` with the script of the same commit as the action, therefore the ref of the action also fixes the version of the script. The action restores the binary packages of vcpkg from the cache of GitHub Actions before the installation and saves them after it. The following example runs the action after [install-qt-action](https://github.com/jurplel/install-qt-action), which sets `QT_ROOT_DIR` to the prefix of Qt:

```yaml
- name: Install vcpkg dependencies
  id: vcpkg
  uses: stdware/dev-scripts/actions/vcpkg-workspace@main
  with:
    qt-dir: ${{ env.QT_ROOT_DIR }}

- name: Configure
  run: cmake -S . -B build -DCMAKE_TOOLCHAIN_FILE=${{ steps.vcpkg.outputs.toolchain-file }}
```

| Input | Meaning |
|---|---|
| `sources-file` | The files of sources, one per line, an earlier file taking precedence. The default is `scripts/vcpkg-sources.json`. |
| `qt-dir` | The value of `--qt-dir`. No Qt is passed if it is empty. |
| `triplet` | The value of `--triplet`. vcpkg uses its default triplet if it is empty. |
| `install-args` | The arguments after `--`, one per line |
| `cache` | Whether the binary packages are restored and saved, `true` by default. If it is `false`, `VCPKG_BINARY_SOURCES` is set to `clear` for the job. |
| `cache-key` | A key that separates the caches of jobs on the same operating system and architecture, for example of different triplets |

| Output | Meaning |
|---|---|
| `vcpkg-root` | The directory of vcpkg |
| `vcpkg-executable` | The vcpkg executable |
| `installed-dir` | The directory into which the dependencies are installed |
| `toolchain-file` | The CMake toolchain file of vcpkg |

The binary packages are cached under a key of the operating system, the architecture, `cache-key` and the commit. A run restores the cache of the latest key with the same prefix, because vcpkg identifies a binary package by its ABI and the packages of an earlier commit remain valid.

## Continuous integration without the action

A workflow can also download the script before it runs the script, for example in GitHub Actions after install-qt-action:

```yaml
- name: Install vcpkg dependencies
  env:
    DEV_SCRIPTS_REF: main
  shell: pwsh
  run: |
    Invoke-WebRequest -OutFile scripts/vcpkg-workspace.cmake "https://raw.githubusercontent.com/stdware/dev-scripts/$env:DEV_SCRIPTS_REF/scripts/vcpkg-workspace.cmake"
    cmake -P scripts/vcpkg-workspace.cmake install --qt-dir $env:QT_ROOT_DIR
```
