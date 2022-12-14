# cmrk 

Useful CMake modules that are reusable enough for a variety of builds.

CMRK = CMake ReinKing modules. I am an egomaniac (but also open to better names).

Requires CMake 3.22+. Subject to rapid updates and changes while on [major version 0](https://semver.org/#spec-item-4).

## Building

Building a package is straightforward since it's all pure CMake code:

```
$ cmake -S . -B build
$ cmake --install build --prefix /path/to/install
```

You can also use `cmrk` as a git submodule with `add_subdirectory` or via
`FetchContent`. More on the latter below.

## Using cmrk

### Find package

```
cmake_minimum_required(VERSION 3.22)
project(example)

find_package(cmrk REQUIRED)

# ... call cmrk_* functions here ...
```

### FetchContent

```
cmake_minimum_required(VERSION 3.22)
project(example)

include(FetchContent)

FetchContent_Declare(
  cmrk
  GIT_REPOSITORY https://github.com/alexreinking/cmrk.git
  GIT_TAG        some_git_sha_here
)

FetchContent_MakeAvailable(cmrk)

# ... call cmrk_* functions here ...
```

## API documentation

### `cmrk_git_version_string`

```
cmrk_git_version_string(
  OUTPUT_VARIABLE <var-name>
  [REQUIRED]
  [NO_CONFIGURE_DEPENDS]
  [TEMPLATE <string>]
)
```

This function computes a version string for your project from the CMake
`PROJECT_VERSION` variable and the results of `git describe --tags`.

The overall operation is as follows:

1. Check if the current project's source directory has a `.git` folder. If not, bail.
2. Find the git executable via `find_package(Git)`. If it cannot be found, bail.
3. Run `git describe --tags` in the project's source directory. If it returns an error status, bail.
4. Store the result from the previous step in a variable named `GIT_VERSION`.
5. If `GIT_VERSION` is identical to `PROJECT_VERSION`, write `${PROJECT_VERSION}` to `<var-name>` and return normally.
6. Otherwise, interpret `TEMPLATE` as an `@ONLY` configuration string and write the result to `<var-name>`.

The default `TEMPLATE` is `@PROJECT_VERSION@ (git~@GIT_VERSION@)`.

The meaning of "bail" is determined by the `REQUIRED` argument. If it is set,
then a fatal error is issued and configuration stops. Otherwise, `<var-name>`
is set to `${PROJECT_VERSION}`.

By default, this function adds dependencies to the configure step on the
directories `.git` and `.git/refs/tags`; this can be avoided with
`NO_CONFIGURE_DEPENDS`.

For an example, see [examples/git_version_string](./examples/git_version_string).

### `cmrk_copy_runtime_dlls`

```
cmrk_copy_runtime_dlls(
  <target>
)
```

Adds a `POST_BUILD` command that copies the runtime DLLs for `<target>` to the
same directory as `<target>`. The command is only added if the target is a DLL
platform and if `<target>` is a shared or module library, or an executable.

The set of runtime DLLs is determined by `$<TARGET_RUNTIME_DLLS:target>`, and
therefore only detects DLL dependencies via the CMake-target dependency graph.
Libraries that are linked via raw flags will not be detected.

This command is idempotent. It will not add the same command to the same target
twice.

Example:

```
find_package(ZLIB REQUIED)

add_executable(my_app main.cpp)
target_link_libraries(my_app PRIVATE ZLIB::ZLIB)

cmrk_copy_runtime_dlls(my_app)
```

### `cmrk_option`

```
cmrk_option(
  NAME <var-name>
  [TYPE <type>]
  [DOC <doc-string>]
  [NO_CACHE]
  [ DEFAULT <string>
  | DEFAULT_FN <fn-name> [DEFAULT_FN_ARGS <args...>] ]
)
```

This is an enhancement on the built-in `option()` and `set(CACHE)` commands.
If `<var-name>` is already defined (even to an empty string), this command
does nothing. Otherwise, it creates a cache variable named `<var-name>` of
type `<type>` and with documentation string `<doc-string>`.

If `NO_CACHE` is set, then `TYPE` and `DOC` are ignored and a normal variable
is created instead of a cache variable.

The value of `TYPE` defaults to `STRING` and the value of `DOC` defaults to a
short message indicating that documentation is missing. This is to encourage
setting a useful value.

`<var-name>` will be set to a value depending on the settings of `DEFAULT` or
`DEFAULT_FN`. Providing both arguments is a fatal error. If `DEFAULT` is set,
then the value provided there will be used.

If `DEFAULT_FN` is set, then the function it names will be called with the
arguments provided in `DEFAULT_FN_ARGS` (if set) plus the following argument:
`OUTPUT_VARIABLE <var>`. The intended semantics are that the function being
called will set `<var>` in the caller's scope. This is to simulate a return
value mechanism since CMake functions do not have structured return values.

If neither `DEFAULT` or `DEFAULT_FN` is set, then the default value is the
empty string.

This function pairs nicely with `execute_process` and `cmrk_git_version_string`,
among others.

For an example, see [examples/git_version_string](./examples/git_version_string).

### `cmrk_debug_print_variables`

```
cmrk_debug_print_variables(
  [NO_VALUES]
  [INCLUDE_REGEX <regex>]
  [EXCLUDE_REGEX <regex>]
)
```

Prints the names and values of all variables matching `INCLUDE_REGEX` and not
matching `EXCLUDE_REGEX`. By default, these are `.*` and `^$`, respectively, so
that every variable will be printed.

If `NO_VALUES` is set, then just the names (and, for cache variables, types)
will be printed.

This function displays its output in three sorted sections: (1) cache entries,
(2) exclusively normal variables, and (3) normal variables which shadow a cache
variable with a _different_ value. Normal variables that shadow a cache
variable with the _same_ value are not shown.
