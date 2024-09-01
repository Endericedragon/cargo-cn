# 指定依赖项

您的crate可以依赖其他来自[crates.io]（或者其他的registry）、`git`仓库、或者本地文件系统中的某个子目录所包含的库。您也可以临时性地覆写（override）某个依赖项的路径位置 --- 在测试本地开发的某个依赖项的bug是否修复时就可以这么干。您也可以为不同的开发平台设置不同的依赖项，或者仅在开发时使用。下面我们将介绍如何实现这些功能。

## 指定crates.io上的依赖项

Cargo默认在[crates.io]上查找依赖项。在这种情况下，只需要依赖名称和版本号即可唯一标识一个依赖项。在[Cargo指南](../guide/index.md)中，我们指定了一个依赖项，即`time` crate：

```toml
[dependencies]
time = "0.1.12"
```

字符串`"0.1.12"`代表需要的版本号。虽然它看上去很像是制定了`time` crate的 *具体版本* ，实际上它指定的是个 *版本范围* ，并且允许[SemVer]版本号规则兼容的升级行为。

允许的升级行为必须满足：新版本号中，主版本号（major）、次要版本号（minor）、以及修订号（patch）的最左端非零数字必须与原版本号相同。在上文的情况中，若我们运行 `cargo update time`， Cargo将会把版本号升级到 `0.1.13`（如果这个版本是所有`0.1.z`中最新的一个的话），但是不会升级到 `0.2.0`。如果我们指定了`1.0`版本，Cargo将把`time` crate升级到`1.1`（同样，如果这个版本是`1.y`中最新的一个的话），但是不会升级到`2.0`。特殊地，`0.0.x`版本号不与任何其他版本号兼容。

[SemVer]: https://semver.org

如下列出了一些样例，展示了Cargo如何解析依赖项的版本范围：

```notrust
1.2.3  :=  >=1.2.3, <2.0.0
1.2    :=  >=1.2.0, <2.0.0
1      :=  >=1.0.0, <2.0.0
0.2.3  :=  >=0.2.3, <0.3.0
0.2    :=  >=0.2.0, <0.3.0
0.0.3  :=  >=0.0.3, <0.0.4
0.0    :=  >=0.0.0, <0.1.0
0      :=  >=0.0.0, <1.0.0
```

这种版本兼容性在处理`1.0.0`之前版本的处理方式和SemVer不同。SemVer认为低于`1.0.0`的版本和任何其他版本都不兼容，Cargo则认为`0.x.y`版本兼容`0.x.z`，其中`y ≥ z`且`x > 0`。

可以进一步微调版本兼容的逻辑，具体方法请参阅[版本要求语法](#version-requirement-syntax)部分。

使用默认的版本兼容逻辑版本要求策略，如`log = "1.2.3"`，以最大化兼容性。

## 版本要求语法（Version requirement syntax）

### 前导插入符要求（Caret requirements）

**前导插入符要求语法** 为默认使用的版本号要求策略。这种版本策略允许[SemVer]兼容的更新。 它们被指定为带有前导插入符（^）的版本要求。

`^1.2.3` 即是一个插入符要求的示例。

省略插入符是使用插入符要求的简化等效语法。 虽然插入符要求是默认行为，但建议尽可能使用简化语法。

`log = "^1.2.3"` 完全等同于 `log = "1.2.3"`。

### 波浪号版本要求（Tilde requirements）

**波浪号版本要求** 指定了一个具有一定更新能力的最小版本。
如果你指定了主版本、次版本和修订版本，或者只指定了主版本和次版本，则只允许修订级别的更改。如果你只指定了主版本，则允许次版本和修订级别的更改。

`~1.2.3` 即是一个波浪号要求的示例。

```notrust
~1.2.3  := >=1.2.3, <1.3.0
~1.2    := >=1.2.0, <1.3.0
~1      := >=1.0.0, <2.0.0
```

### 通配符版本要求

**通配符版本要求** 允许在通配符位置使用任何版本。

`*`, `1.*` 和 `1.2.*` 均为通配符要求的示例。

```notrust
*     := >=0.0.0
1.*   := >=1.0.0, <2.0.0
1.2.* := >=1.2.0, <1.3.0
```

> **注意**: [crates.io] 不允许仅用一个 `*` 通配符来指定依赖项（doesn't allow bare `*`）。

### 不等号版本要求

**不等号版本要求** 允许手动指定一个版本范围或精确的版本依赖。

以下是一些不等号版本要求的示例：

```notrust
>= 1.2.0
> 1
< 2
= 1.2.3
```

<span id="multiple-requirements"></span>
### 指定多个版本要求

如您在上方的示例中看到的一样，可以通过逗号分隔，指定多个版本要求，例如： `>= 1.2, < 1.5` 。

> **建议：** 当您不确定用哪个时，请使用默认的版本要求操作符。
>
> 在极少数情况下，一个带有“公共依赖”的包（重新导出了其依赖项或在其公共 API 中与其他crate存在互操作）
> 可能兼容多个SemVer规则不兼容的版本（例如，仅使用诸如`Id`这样在不同版本之间保持不变的简单数据类型）
> 从而允许用户选择使用哪个版本的“公共依赖”。在这种情况下，像 `">=0.4, <2"` 这样的版本要求可能会派上用场。
> *然而*，crate的用户可能会遇到错误，并且需要通过 `cargo update` 手动选择“公共依赖”的版本，
> 因为 Cargo 在 [解析依赖版本](resolver.md) 时可能会选择不同版本的“公共依赖”（见 [#10599]）。
>
> 避免将版本的上限约束到小于下一个语义版本不兼容的版本
> （例如，避免 `">=2.0, <2.4"`），因为依赖树中的其他包可能需要更新版本的包，导致无法正确解析依赖项的错误（见 [#9029]）。
> 此时，也许应当考虑在 [`Cargo.lock`] 中控制版本。
>
> 在某些情况下，这可能无关紧要甚至得不偿失，包括：
> - 当没有其他人依赖你的包时，例如它只有一个 `[[bin]]`
> - 当依赖预发布包并希望避免破坏性更改时，可能需要完全指定的 `"=1.2.3-alpha.3"`（见 [#2222]）
> - 当一个库重新导出一个过程宏，但该过程宏生成的代码调用重新导出库时，可能需要完全指定的 `=1.2.3`，
>   以确保过程宏的版本不高于重新导出的库，并且生成的代码使用的 API 部分在当前版本中不存在

[`Cargo.lock`]: ../guide/cargo-toml-vs-cargo-lock.md
[#2222]: https://github.com/rust-lang/cargo/issues/2222
[#9029]: https://github.com/rust-lang/cargo/issues/9029
[#10599]: https://github.com/rust-lang/cargo/issues/10599

## 指定其他registry中的依赖项

想指定其他registry中的依赖项，只需设定`registry`值为目标registry即可：

```toml
[dependencies]
some-crate = { version = "1.0", registry = "my-registry" }
```

此处的`my-registry`是指在`.cargo/config.toml`文件中配置的registry名称。
更多信息请参阅[registries文档](registries.md)。

> **注意**: [crates.io] 不允许发布包含[crates.io]以外的依赖项的软件包。

[registries文档]: registries.md

## 指定`git`仓库中的依赖项

想指定`git`仓库中的依赖项，您至少指定`git`值为仓库的网络位置：

```toml
[dependencies]
regex = { git = "https://github.com/rust-lang/regex.git" }
```

Cargo将会下载指定的仓库，并在仓库目录中遍历文件树，以查找`Cargo.toml`文件。例如，`regex-lite`和`regex-syntax`都是`rust-lang/regex`仓库中的Rust工作空间的成员，因此可以用该仓库的根URL（`https://github.com/rust-lang/regex.git`）来引用，与他们在文件树中的位置无关。

```toml
regex-lite   = { git = "https://github.com/rust-lang/regex.git" }
regex-syntax = { git = "https://github.com/rust-lang/regex.git" }
```

不过，上述规则并不适用于[`path`指定的依赖](#specifying-path-dependencies).

### 选择特定提交（commit）

Cargo 假定：如果我们只指定仓库 URL，那么我们就打算使用默认分支上的最新提交来构建我们的包，如上面的例子所示。

你可以将 `git`键与 `rev`、`tag` 或 `branch`键结合使用，以更具体地指定要使用的提交。以下是使用名为 `next` 的分支上的最新提交的示例：

```toml
[dependencies]
regex = { git = "https://github.com/rust-lang/regex.git", branch = "next" }
```

任何不是分支或标签的内容都属于 `rev`键。这可以是一个提交的哈希，例如 `rev = "4c59b707"`，或者是由远程仓库暴露的命名引用，例如 `rev = "refs/pull/493/head"`。

`rev`键可用的引用取决于仓库托管的位置。GitHub 暴露了每个拉取请求的最新提交的引用，如上面的例子所示。其他 git 托管平台可能提供类似的功能，但命名方案可能不同。

**其他 `git` 依赖项的例子：**

```toml
# .git suffix can be omitted if the host accepts such URLs - both examples work the same
regex = { git = "https://github.com/rust-lang/regex" }
regex = { git = "https://github.com/rust-lang/regex.git" }

# a commit with a particular tag
regex = { git = "https://github.com/rust-lang/regex.git", tag = "1.10.3" }

# a commit by its SHA1 hash
regex = { git = "https://github.com/rust-lang/regex.git", rev = "0c0990399270277832fbb5b91a1fa118e6f63dba" }

# HEAD commit of PR 493
regex = { git = "https://github.com/rust-lang/regex.git", rev = "refs/pull/493/head" }

# INVALID EXAMPLES

# specifying the commit after # ignores the commit ID and generates a warning
regex = { git = "https://github.com/rust-lang/regex.git#4c59b70" }

# git and path cannot be used at the same time
regex = { git = "https://github.com/rust-lang/regex.git#4c59b70", path = "../regex" }
```

Cargo 在添加 `git` 依赖时，将其使用的提交锁定在 `Cargo.lock` 文件中，并且仅在运行 `cargo update` 命令时才会检查更新。

### `version`键的作用

`version` 键总意味着软件包在registry中可用，无论是否存在 `git` 或 `path` 键。

虽然`version` 键不影响 Cargo 检索 `git` 依赖时使用的提交，但 Cargo 会检查依赖的 `Cargo.toml` 文件中的版本信息是否与 `version` 键匹配，并在检查失败时引发错误。

在这个例子中，Cargo 从 Git 中检索名为 `next` 分支的 HEAD 提交，并检查 crate 的版本是否与 `version = "1.10.3"` 兼容：

```toml
[dependencies]
regex = { version = "1.10.3", git = "https://github.com/rust-lang/regex.git", branch = "next" }
```

`version`, `git`, 和 `path` 键均是独立地解析依赖的选项。
请参见下面的 [指定多个位置](#multiple-locations) 部分以获取详细解释。

> **注意**: [crates.io] 不允许发布带有依赖于 [crates.io] 之外发布的代码的包（[dev-dependencies] 被忽略）。请参见 [指定多个位置](#multiple-locations) 部分以获取替代 `git` 和 `path` 依赖的方案。


### 访问私有Git仓库

See [Git Authentication](../appendix/git-authentication.md) for help with Git authentication for private repos.

## Specifying path dependencies

Over time, our `hello_world` package from [the guide](../guide/index.md) has
grown significantly in size! It’s gotten to the point that we probably want to
split out a separate crate for others to use. To do this Cargo supports **path
dependencies** which are typically sub-crates that live within one repository.
Let’s start by making a new crate inside of our `hello_world` package:

```console
# inside of hello_world/
$ cargo new hello_utils
```

This will create a new folder `hello_utils` inside of which a `Cargo.toml` and
`src` folder are ready to be configured. To tell Cargo about this, open
up `hello_world/Cargo.toml` and add `hello_utils` to your dependencies:

```toml
[dependencies]
hello_utils = { path = "hello_utils" }
```

This tells Cargo that we depend on a crate called `hello_utils` which is found
in the `hello_utils` folder, relative to the `Cargo.toml` file it’s written in.

The next `cargo build` will automatically build `hello_utils` and
all of its dependencies.

### No local path traversal

The local paths must point to the exact folder with the dependency's `Cargo.toml`.
Unlike with `git` dependencies, Cargo does not traverse local paths.
For example, if `regex-lite` and `regex-syntax` are members of a
locally cloned `rust-lang/regex` repo, they have to be referred to by the full path:

```toml
# git key accepts the repo root URL and Cargo traverses the tree to find the crate
[dependencies]
regex-lite   = { git = "https://github.com/rust-lang/regex.git" }
regex-syntax = { git = "https://github.com/rust-lang/regex.git" }

# path key requires the member name to be included in the local path
[dependencies]
regex-lite   = { path = "../regex/regex-lite" }
regex-syntax = { path = "../regex/regex-syntax" }
```

### Local paths in published crates

Crates that use dependencies specified with only a path are not
permitted on [crates.io].

If we wanted to publish our `hello_world` crate,
we would need to publish a version of `hello_utils` to [crates.io] as a separate crate
and specify its version in the dependencies line of `hello_world`:

```toml
[dependencies]
hello_utils = { path = "hello_utils", version = "0.1.0" }
```

The use of `path` and `version` keys together is explained in the [Multiple locations](#multiple-locations) section.

> **Note**: [crates.io] does not allow packages to be published with
> dependencies on code outside of [crates.io], except for [dev-dependencies].
> See the [Multiple locations](#multiple-locations) section
> for a fallback alternative for `git` and `path` dependencies.

## Multiple locations

It is possible to specify both a registry version and a `git` or `path`
location. The `git` or `path` dependency will be used locally (in which case
the `version` is checked against the local copy), and when published to a
registry like [crates.io], it will use the registry version. Other
combinations are not allowed. Examples:

```toml
[dependencies]
# Uses `my-bitflags` when used locally, and uses
# version 1.0 from crates.io when published.
bitflags = { path = "my-bitflags", version = "1.0" }

# Uses the given git repo when used locally, and uses
# version 1.0 from crates.io when published.
smallvec = { git = "https://github.com/servo/rust-smallvec.git", version = "1.0" }

# N.B. that if a version doesn't match, Cargo will fail to compile!
```

One example where this can be useful is when you have split up a library into
multiple packages within the same workspace. You can then use `path`
dependencies to point to the local packages within the workspace to use the
local version during development, and then use the [crates.io] version once it
is published. This is similar to specifying an
[override](overriding-dependencies.md), but only applies to this one
dependency declaration.

## Platform specific dependencies

Platform-specific dependencies take the same format, but are listed under a
`target` section. Normally Rust-like [`#[cfg]`
syntax](../../reference/conditional-compilation.html) will be used to define
these sections:

```toml
[target.'cfg(windows)'.dependencies]
winhttp = "0.4.0"

[target.'cfg(unix)'.dependencies]
openssl = "1.0.1"

[target.'cfg(target_arch = "x86")'.dependencies]
native-i686 = { path = "native/i686" }

[target.'cfg(target_arch = "x86_64")'.dependencies]
native-x86_64 = { path = "native/x86_64" }
```

Like with Rust, the syntax here supports the `not`, `any`, and `all` operators
to combine various cfg name/value pairs.

If you want to know which cfg targets are available on your platform, run
`rustc --print=cfg` from the command line. If you want to know which `cfg`
targets are available for another platform, such as 64-bit Windows,
run `rustc --print=cfg --target=x86_64-pc-windows-msvc`.

Unlike in your Rust source code, you cannot use
`[target.'cfg(feature = "fancy-feature")'.dependencies]` to add dependencies
based on optional features. Use [the `[features]` section](features.md)
instead:

```toml
[dependencies]
foo = { version = "1.0", optional = true }
bar = { version = "1.0", optional = true }

[features]
fancy-feature = ["foo", "bar"]
```

The same applies to `cfg(debug_assertions)`, `cfg(test)` and `cfg(proc_macro)`.
These values will not work as expected and will always have the default value
returned by `rustc --print=cfg`.
There is currently no way to add dependencies based on these configuration values.

In addition to `#[cfg]` syntax, Cargo also supports listing out the full target
the dependencies would apply to:

```toml
[target.x86_64-pc-windows-gnu.dependencies]
winhttp = "0.4.0"

[target.i686-unknown-linux-gnu.dependencies]
openssl = "1.0.1"
```

### Custom target specifications

If you’re using a custom target specification (such as `--target
foo/bar.json`), use the base filename without the `.json` extension:

```toml
[target.bar.dependencies]
winhttp = "0.4.0"

[target.my-special-i686-platform.dependencies]
openssl = "1.0.1"
native = { path = "native/i686" }
```

> **Note**: Custom target specifications are not usable on the stable channel.

## Development dependencies

You can add a `[dev-dependencies]` section to your `Cargo.toml` whose format
is equivalent to `[dependencies]`:

```toml
[dev-dependencies]
tempdir = "0.3"
```

Dev-dependencies are not used when compiling
a package for building, but are used for compiling tests, examples, and
benchmarks.

These dependencies are *not* propagated to other packages which depend on this
package.

You can also have target-specific development dependencies by using
`dev-dependencies` in the target section header instead of `dependencies`. For
example:

```toml
[target.'cfg(unix)'.dev-dependencies]
mio = "0.0.1"
```

> **Note**: When a package is published, only dev-dependencies that specify a
> `version` will be included in the published crate. For most use cases,
> dev-dependencies are not needed when published, though some users (like OS
> packagers) may want to run tests within a crate, so providing a `version` if
> possible can still be beneficial.

## Build dependencies

You can depend on other Cargo-based crates for use in your build scripts.
Dependencies are declared through the `build-dependencies` section of the
manifest:

```toml
[build-dependencies]
cc = "1.0.3"
```


You can also have target-specific build dependencies by using
`build-dependencies` in the target section header instead of `dependencies`. For
example:

```toml
[target.'cfg(unix)'.build-dependencies]
cc = "1.0.3"
```

In this case, the dependency will only be built when the host platform matches the
specified target.

The build script **does not** have access to the dependencies listed
in the `dependencies` or `dev-dependencies` section. Build
dependencies will likewise not be available to the package itself
unless listed under the `dependencies` section as well. A package
itself and its build script are built separately, so their
dependencies need not coincide. Cargo is kept simpler and cleaner by
using independent dependencies for independent purposes.

## Choosing features

If a package you depend on offers conditional features, you can
specify which to use:

```toml
[dependencies.awesome]
version = "1.3.5"
default-features = false # do not include the default features, and optionally
                         # cherry-pick individual features
features = ["secure-password", "civet"]
```

More information about features can be found in the [features
chapter](features.md#dependency-features).

## Renaming dependencies in `Cargo.toml`

When writing a `[dependencies]` section in `Cargo.toml` the key you write for a
dependency typically matches up to the name of the crate you import from in the
code. For some projects, though, you may wish to reference the crate with a
different name in the code regardless of how it's published on crates.io. For
example you may wish to:

* Avoid the need to  `use foo as bar` in Rust source.
* Depend on multiple versions of a crate.
* Depend on crates with the same name from different registries.

To support this Cargo supports a `package` key in the `[dependencies]` section
of which package should be depended on:

```toml
[package]
name = "mypackage"
version = "0.0.1"

[dependencies]
foo = "0.1"
bar = { git = "https://github.com/example/project.git", package = "foo" }
baz = { version = "0.1", registry = "custom", package = "foo" }
```

In this example, three crates are now available in your Rust code:

```rust,ignore
extern crate foo; // crates.io
extern crate bar; // git repository
extern crate baz; // registry `custom`
```

All three of these crates have the package name of `foo` in their own
`Cargo.toml`, so we're explicitly using the `package` key to inform Cargo that
we want the `foo` package even though we're calling it something else locally.
The `package` key, if not specified, defaults to the name of the dependency
being requested.

Note that if you have an optional dependency like:

```toml
[dependencies]
bar = { version = "0.1", package = 'foo', optional = true }
```

you're depending on the crate `foo` from crates.io, but your crate has a `bar`
feature instead of a `foo` feature. That is, names of features take after the
name of the dependency, not the package name, when renamed.

Enabling transitive dependencies works similarly, for example we could add the
following to the above manifest:

```toml
[features]
log-debug = ['bar/log-debug'] # using 'foo/log-debug' would be an error!
```

## Inheriting a dependency from a workspace

Dependencies can be inherited from a workspace by specifying the
dependency in the workspace's [`[workspace.dependencies]`][workspace.dependencies] table.
After that, add it to the `[dependencies]` table with `workspace = true`.

Along with the `workspace` key, dependencies can also include these keys:
- [`optional`][optional]: Note that the`[workspace.dependencies]` table is not allowed to specify `optional`.
- [`features`][features]: These are additive with the features declared in the `[workspace.dependencies]`

Other than `optional` and `features`, inherited dependencies cannot use any other
dependency key (such as `version` or `default-features`).

Dependencies in the `[dependencies]`, `[dev-dependencies]`, `[build-dependencies]`, and
`[target."...".dependencies]` sections support the ability to reference the
`[workspace.dependencies]` definition of dependencies.

```toml
[package]
name = "bar"
version = "0.2.0"

[dependencies]
regex = { workspace = true, features = ["unicode"] }

[build-dependencies]
cc.workspace = true

[dev-dependencies]
rand = { workspace = true, optional = true }
```


[crates.io]: https://crates.io/
[dev-dependencies]: #development-dependencies
[workspace.dependencies]: workspaces.md#the-dependencies-table
[optional]: features.md#optional-dependencies
[features]: features.md

<script>
(function() {
    var fragments = {
        "#overriding-dependencies": "overriding-dependencies.html",
        "#testing-a-bugfix": "overriding-dependencies.html#testing-a-bugfix",
        "#working-with-an-unpublished-minor-version": "overriding-dependencies.html#working-with-an-unpublished-minor-version",
        "#overriding-repository-url": "overriding-dependencies.html#overriding-repository-url",
        "#prepublishing-a-breaking-change": "overriding-dependencies.html#prepublishing-a-breaking-change",
        "#overriding-with-local-dependencies": "overriding-dependencies.html#paths-overrides",
    };
    var target = fragments[window.location.hash];
    if (target) {
        var url = window.location.toString();
        var base = url.substring(0, url.lastIndexOf('/'));
        window.location.replace(base + "/" + target);
    }
})();
</script>
