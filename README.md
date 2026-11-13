# ros_workspace_shell

The shell setup files in the root of a ROS 2 install prefix (`setup.bash`,
`local_setup.sh` and the others) and the environment of the prefix, built
with plain CMake. Neither building nor using this package needs any ament
package.

It is the part of `ros_workspace` that a distribution without ament packages
needs, for example a Gazebo distribution that ROS distributions extend
(REP 159). `ros_workspace` depends on this package and adds the ament part:
`AMENT_PREFIX_PATH` for the prefix and the package's CMake config files.

## What it installs

| Where | Files |
|---|---|
| root of the prefix | `setup.{sh,bash,zsh,fish}`, `local_setup.{sh,bash,zsh,fish}`, `_local_setup_util.py` |
| `share/ros_workspace_shell/` | `package.dsv`, `package.xml` |
| `share/ament_index/resource_index/` | `packages`, `package_run_dependencies` and `parent_prefix_path` entries for `ros_workspace_shell` |

The resource index entries are plain files that the setup files read to
find the packages of the prefix; writing them needs no ament code.

`package.dsv` holds the environment of the prefix, one line per variable:

```
prepend-non-duplicate;LD_LIBRARY_PATH;lib
prepend-non-duplicate;LD_LIBRARY_PATH;lib/x86_64-linux-gnu
prepend-non-duplicate-if-exists;PATH;bin
prepend-non-duplicate;PYTHONPATH;lib/python3.12/site-packages
```

After sourcing `setup.bash`, the prefix is on `PATH` (if it has a `bin`
directory), `LD_LIBRARY_PATH` (`DYLD_LIBRARY_PATH` on macOS) and
`PYTHONPATH`. `AMENT_PREFIX_PATH` is set by `ros_workspace`, not by this
package.

ament packages install the same environment as shell scripts under
`share/<package>/environment/`, with a `.dsv` file next to each one, and
package level `local_setup.{sh,bash,zsh,fish}` scripts. The setup files in
the root of the prefix never run those scripts when a `.dsv` file exists,
so this package installs only the `.dsv` lines. The environment after
sourcing any of the setup files is the same.

## Templates

`templates/` holds unmodified copies of the prefix level templates of
ament_package, without the Windows `.bat` templates:

| Files in `templates/` | Copied from | Version |
|---|---|---|
| `prefix_level/*` | [ament/ament_package](https://github.com/ament/ament_package), `ament_package/template/prefix_level/` | tag `0.18.3`, commit `98089783110f7606a90e8ef9786719c3d0cd32f9` |

The setup files this package generates from them are identical to the ones
ament_cmake generates for `ros_workspace` 1.0.4. To update them, copy the
new versions over these and update the table.

## Parent distributions (REP 159)

A distribution that extends others with `binary_import` passes their install
prefixes, nearest first:

```
-DROS_WORKSPACE_PARENT_PREFIX_PATH="/opt/ros/parent_a;/opt/ros/parent_b"
```

The setup files then source each parent's `local_setup` first. A root
distribution leaves the list empty.

## Building

```
cmake -S . -B build -DCMAKE_INSTALL_PREFIX=/opt/ros/<distro>
cmake --build build
cmake --install build
```

The build needs a C compiler, used only to find the multiarch library
directory, and Python 3, whose path is written into the setup files.

## Limitations

- Linux and macOS only: the Windows `.bat` setup files are not generated.
- `setup.fish` does not source the parent distributions: in the ament_package
  template, the local `AMENT_CURRENT_PREFIX` set on its line 5 hides the
  global one it sets for each prefix, so every `local_setup.fish` it sources
  works on this prefix. `setup.sh`, `setup.bash` and `setup.zsh` are not
  affected.
