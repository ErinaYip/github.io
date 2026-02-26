---
title: Nixos手动构建grub主题
description: '在Nixos中手动构建并安装grub主题'
published: 2026-02-26
image: ''
tags: [ 'Rice', 'Nixos', 'nix' ]
category: 'Nixos'
draft: false 
---

Nixos官方pkgs内支持的grub主题不多，相比于海量的第三方主题更是寥寥无几，如果想手动安装喜欢的grub主题，就免不了自己编写脚本构建。

标准的grub主题需要`theme.txt`在主题文件夹的根目录以声明主题的配置和其他字体/图片/图标等的位置，这种文件结构的grub主题配置起来比较简单，我们可以参考[jeslie0](https://github.com/jeslie0)的项目：[NixOS Grub Themes](https://github.com/jeslie0/nixos-grub-themes)。

我这里以[yeyushengfan258/Particle-circle-grub-theme](https://github.com/yeyushengfan258/Particle-circle-grub-theme)为例，这个grub主题比较特殊，因为它的仓库不包含标准的grub主题配置文件结构，而是使用脚本生成配置，这种方法被广大现代化的grub主题采用，对于传统的结构，这样的方式显然更灵活和利好客制化，但对于我们的配置，则需要多几步操作。

废话不多说，这里先直接呈上完整配置：

```nix showLineNumber
boot.loader.grub = {
  enable = true;
  device = "nodev";
  efiSupport = true;
  useOSProber = true;
  theme =  pkgs.stdenv.mkDerivation {
    pname = "particle-circle-grub-theme";
    version = "unstable-2025-03-18";

    src = pkgs.fetchFromGitHub {
      owner = "yeyushengfan258";
      repo = "Particle-circle-grub-theme";
      rev = "f27991237562f93aacc3f333be4284430889f5bb";
      hash = "sha256-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=";
    };

    nativeBuildInputs = [ pkgs.imagemagick ];

    buildPhase = ''
      patchShebangs generate.sh
      ./generate.sh -t window
    '';

    installPhase = ''
      mkdir -p $out
      cp -r Particle-circle-window/* $out/
    '';
  };
};
```

- `mkDerivation`时常见的写法是`pkgs.stdenv.mkDerivation rec { ... };`，这里的 `rec` 是 `recrecursive attribute set（递归属性集）` 的关键字，它使得 `mkDerivation` 属性集内属性可以**互相引用自己**，因为这里没有用到 pname/version 的自我引用，所以不需要 `rec`，其次，现代 Nixpkgs 推荐避免不必要的 rec，使用 `(finalAttrs: { ... })` 形式更安全（但本例简单，可省略）。
- 第7、8行：`pname`和`version`是你构建的包的信息，可以任意填
- 第13行的`rev`可以直接填`tag`或某次`commit`，因为这个包没有任何的release，所以使用最新的commit
- 第14行哈希值不知道就乱填，然后使用`nixos-rebuild`生成一次配置，Nix就会报错并显示正确的哈希，例如：

  ```bash
  error: hash mismatch in fixed-output derivation '/nix/store/ks8wiwcb9dn0k2hxc8wr
  2akv6zin086j-source.drv':
           specified: sha256-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=
              got:    sha256-3yusy7V+ASj6vS7yPe9xhi2YY9TGFKKJpZQwZqITJ0U=
  Command 'nix --extra-experimental-features 'nix-command flakes' build --print-ou
  t-paths '.#nixosConfigurations."nixos".config.system.build.toplevel' --no-link' 
  returned non-zero exit status 1.
  ```

接下来的内容就是与一般构建不同的了，因为使用到了脚本生成配置，我们要在构建时运行脚本并复制正确的配置到目录。

这里我们需要指定`nativeBuildInputs = [ pkgs.imagemagick ];`作为构建时的工具，因为在这个仓库的README中提到：`Setting a custom background: Make sure you have imagemagick installed, or at least something that provides convert`，这个包依赖`convert`命令来转换图片的分辨率。

这里的`$out`是 Nix 在执行 **derivation** 的构建阶段时自动注入的环境变量，内容大概是`/nix/store/abc123xyz...-particle-circle-grub-theme`，即指向最终输出产物的**完整 Nix store 路径**


| 变量       | 含义                                   | 用法                                  |
|------------|----------------------------------------|---------------------------------------|
| `$out`     | 主要输出目录                           | 放最终产物                            |
| `$bin`     | 二进制/可执行文件输出                  | 放 bin/ 下的程序                      |
| `$dev`     | 开发头文件输出                         | 放 include/ 下的 .h 文件              |
| `$lib`     | 库文件输出                             | 放 lib/ 下的 .so / .a                 |
| `$src`     | 源码解压后的目录                       | 读取原始文件（如 generate.sh）        |
| `$PWD`     | 当前工作目录（通常等于 $src）          | 执行脚本时用                          |


---

**参考文献：**
- [Nixpkgs Reference Manual#chap-stdenv)](https://nixos.org/manual/nixpkgs/stable/#chap-stdenv)
- [Nixpkgs Reference Manual#sec-stdenv-phases](https://nixos.org/manual/nixpkgs/stable/#sec-stdenv-phases)
- [Fundamentals of Stdenv - Nix Pills](https://nixos.org/guides/nix-pills/19-fundamentals-of-stdenv.html)
- [Derivations - Official NixOS Wiki](https://wiki.nixos.org/wiki/Derivations)
- [What is the `patchShebangs` command in Nix build expressions? - Help - NixOS Discourse](https://discourse.nixos.org/t/what-is-the-patchshebangs-command-in-nix-build-expressions/12656)
- [Grub Theme Help - Help - NixOS Discourse](https://discourse.nixos.org/t/grub-theme-help/24384/6)
