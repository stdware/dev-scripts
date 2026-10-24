# dev-scripts

This repository provides common scripts for C/C++ development. Each script is a single file under `scripts/`, which a project downloads into its own repository rather than vendoring this repository. [`docs/about-scripts`](docs/about-scripts) describes the usage of each script.

| Script | Purpose |
|---|---|
| [`vcpkg-workspace.cmake`](docs/about-scripts/vcpkg-workspace.md) | Checks out vcpkg and vcpkg overlays as a project specifies them, and installs the dependencies of the project |

## Download

A project downloads a script into its `scripts` directory and ignores it in `scripts/.gitignore`. The following command downloads a script in a POSIX shell:

```sh
curl -fsSL -o scripts/<script> https://raw.githubusercontent.com/stdware/dev-scripts/main/scripts/<script>
```

The following command downloads a script in PowerShell or in cmd:

```bat
powershell -Command "Invoke-WebRequest -OutFile scripts/<script> https://raw.githubusercontent.com/stdware/dev-scripts/main/scripts/<script>"
```

Replace `main` with a commit hash to download a fixed version.

## License

MIT. See [LICENSE](LICENSE).
