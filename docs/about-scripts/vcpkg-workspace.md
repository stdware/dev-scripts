# vcpkg-workspace.cmake

Prepares the vcpkg workspace of a project without git submodules: vcpkg and any number of overlay repositories are checked out at the refs that the project specifies, and the dependencies of the vcpkg manifest of the project are installed with every overlay. It runs on Windows, macOS and Linux with CMake 3.19 or later and git.

## Setup

1. Download the script into the `scripts` directory of the project.

   ```sh
   curl -fsSL -o scripts/vcpkg-workspace.cmake https://raw.githubusercontent.com/stdware/dev-scripts/main/scripts/vcpkg-workspace.cmake
   ```

   ```bat
   powershell -Command "Invoke-WebRequest -OutFile scripts/vcpkg-workspace.cmake https://raw.githubusercontent.com/stdware/dev-scripts/main/scripts/vcpkg-workspace.cmake"
   ```

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
| `fetch` | Checks out vcpkg and every overlay, and bootstraps vcpkg after a checkout of another commit |
| `install` | Runs `fetch`, then installs the dependencies with every overlay |
| `status` | Lists the specified and the checked-out commit of every repository |
| `run <task>` | Runs a task of the sources |
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

Paths in the sources are relative to the file that gives them.

- A `ref` is a full commit hash, a branch or a tag. A branch and a tag are fetched at each run.
- A `repository` that begins with `./` or `../` is relative to the remote `origin` of the project, as in `.gitmodules`, so that an overlay is fetched over the protocol by which the project was cloned.
- The dependencies are installed into `<vcpkg>/installed`.

## Local sources

`scripts/vcpkg-sources.local.json`, if it exists, takes precedence over `scripts/vcpkg-sources.json`. It has the same format, and its members replace those of vcpkg and of the overlays of the same name. A `path` uses a local directory as it is, without git:

```json
{
    "overlays": [
        { "name": "vcpkg-overlay", "path": "D:/GitHub/vcpkg-overlay" }
    ]
}
```

`--sources-file <file>` replaces both defaults and may be given more than once, an earlier file taking precedence.

## Qt

The ports that build against Qt receive it through the environment variables `QT_DIR` and `Qt6_DIR` or `Qt5_DIR`, which the script sets from `--qt-dir` and adds to `VCPKG_KEEP_ENV_VARS`. `--qt-dir` accepts the CMake directory of Qt 6 or Qt 5 or the prefix of Qt. Without the option, the environment variable `QT_DIR`, `Qt6_DIR` or `Qt5_DIR` applies.

## Continuous integration

A workflow downloads the script before it runs, for example in GitHub Actions:

```yaml
- name: Install vcpkg dependencies
  env:
    DEV_SCRIPTS_REF: main
    QT_DIR: ${{ env.QT_ROOT_DIR }}/lib/cmake/Qt6
  shell: pwsh
  run: |
    Invoke-WebRequest -OutFile scripts/vcpkg-workspace.cmake "https://raw.githubusercontent.com/stdware/dev-scripts/$env:DEV_SCRIPTS_REF/scripts/vcpkg-workspace.cmake"
    cmake -P scripts/vcpkg-workspace.cmake install
```
