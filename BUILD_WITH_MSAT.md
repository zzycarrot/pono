# Pono `--with-msat` 构建记录

## 现象

在 `tools/pono` 下执行：

```bash
./contrib/setup-smt-switch.sh --with-msat
```

会在 `smt-switch` 配置阶段失败，核心报错为：

```text
Package 'bitwuzla' not found
```

## 根因

这次失败不是 `MathSAT` 本身导致的，而是 `smt-switch` 默认同时要求 `bitwuzla`、`cvc5`、`msat`。

现场排查得到的几个关键点：

1. `bitwuzla` 的前置构建没有真正完成。
   缺少 `meson` / `ninja` 时，`bitwuzla/configure.py` 会直接失败。
2. `deps/smt-switch/deps/bitwuzla/` 目录一旦残留，后续 `setup-bitwuzla.sh` 会直接退出。
   这样会留下“源码目录存在，但安装产物缺失”的半成品状态。
3. 手工重建 `bitwuzla` 时，需要让 `PKG_CONFIG_PATH` 能看到 `deps/install/lib/pkgconfig`。
   否则 `cadical.pc` 虽然存在，Meson 仍然会走联网 fallback。
4. 第一次拉取 `symfpu` 失败后，会留下残缺的 `subprojects/symfpu/` 目录。
   后续需要先清掉这个目录再重试。
5. 在 Apple Silicon 上，脚本没有显式优先使用 Homebrew 的 `bison` / `flex`，
   容易退回到 `/usr/bin/bison 2.3`，从而在 `smt-switch` 配置时报版本不够。
6. `pono` 自身的测试配置默认要求 `GTest 1.14`，会导致本机已有的 `1.11.0`
   被拒绝并尝试联网下载。

## 已做修改

### `contrib/setup-smt-switch.sh`

已增强为：

1. 自动优先使用 `tools/pono/.build-venv/bin` 里的构建工具。
2. 如果 `bitwuzla` 源码目录已存在但安装产物缺失，会自动原地重建。
3. 如果发现残缺的 `subprojects/symfpu/`，会先清理再重试。
4. 自动给 `PKG_CONFIG_PATH` 追加 `deps/install/lib/pkgconfig`。
5. 在 macOS 上自动把 Homebrew 的 `bison` / `flex` 路径传给 `smt-switch/configure.sh`。
6. 不再因为 `deps/smt-switch/build/` 已存在而拒绝重配。

### `CMakeLists.txt`

已补充 Apple Silicon 常见的 Homebrew 前缀：

```text
/opt/homebrew/opt/bison
/opt/homebrew/opt/flex
```

### `tests/CMakeLists.txt`

已做两处修正：

1. 不再强制要求 `GTest 1.14`，允许使用本机已有版本。
2. 使用系统 `GTest` 时，为测试目标补上 `BUILD_RPATH`。

## 当前可用的构建步骤

### 1. 准备本地构建工具

如果本机没有 `meson` / `ninja`，建议在 `tools/pono` 下创建本地 venv：

```bash
python3 -m venv .build-venv
./.build-venv/bin/pip install meson ninja
```

### 2. 构建 `smt-switch`

确保 `deps/mathsat/` 已经放好 MathSAT 包后执行：

```bash
./contrib/setup-smt-switch.sh --with-msat
```

### 3. 构建 `pono`

```bash
./configure.sh --with-msat
cmake --build build -j
```

## 本次验证结果

已经在当前机器上验证：

1. `./contrib/setup-smt-switch.sh --with-msat` 成功。
   安装出了：
   - `deps/smt-switch/local/lib/libsmt-switch.a`
   - `deps/smt-switch/local/lib/libsmt-switch-bitwuzla.a`
   - `deps/smt-switch/local/lib/libsmt-switch-cvc5.a`
   - `deps/smt-switch/local/lib/libsmt-switch-msat.a`
2. `./configure.sh --with-msat` 成功。
3. `cmake --build build -j` 成功，`build/pono` 已生成。
4. 抽样测试 `ctest -R '^test_ts$' --output-on-failure` 已通过。

## 备注

`smt-switch` 在当前环境下的 74 个测试已经全部通过。

`pono` 的整套 `check` 目标没有在这次记录里完整重跑到结束，只做了配置修复和单测抽样。
如果后续要继续跑全量测试，可以从：

```bash
cmake --build build --target check -j 4
```

继续。
