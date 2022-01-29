# 说明

此项目只是为了简单验证一下rust往openwrt(redmi ac2100)交叉编译的可能性。

前提：
1. 一份编编译过的面向redmi ac 2100 openwrt 21.02源码，并成功编译了如下项目以便具备一份可用于rust编译的openssl发行包

https://github.com/trojan-gfw/openwrt-trojan

其他：
1. 对trojan-r简单粗暴进行了修改（最近因为放假，学习了2天rust代码），去掉了server模式，替换rustls为native-tls（底层是openssl）
2. trojan-r不支持nat模式，所以对我来说，只是测试一下可行性而已，另外，实测既便关闭debug模式，在看ytb时，仍会导致trojan-r内存占用到25M左右。因此相比前述基于c++的项目常驻是
   4M内存而言，还是更适合一点。
3. shadowsocks-rust项目里，对于x86/arm机型，可以采用ring，而对于非x86/arm机型，采用了自有的密码库，这样解决了mips支持的问题。

https://github.com/shadowsocks/crypto2

anyway，rust尽管还是比较复杂，但在交叉编译、包管理方面，另外就是没有GC机制这点，也同样比较吸引我，但在这此实际操作来看，暂时我还没有体会到在内存上相比go的明显优势，毕竟redmi
ac 2100只有128M内存，很容易被oom kill掉。但这个原因可能不是rust的，而是rust可能与openssl library交互这部分，但现在来看，还有一些路要走。

期待未来ring项目支持mips，trojan-r这个项目也可以继续更新。

前提完成之后，主要需要做如下修改：

cargo配置：
```
[target.mipsel-unknown-linux-musl]
linker = "/home/xxx/code/openwrt/staging_dir/toolchain-mipsel_24kc_gcc-8.4.0_musl/bin/mipsel-openwrt-linux-musl-gcc"
rustflags = ["-C", "target-feature=+crt-static", "-C", "link-arg=-s"]
```

编译配置：
```
xxx@amdpc-ubuntu:~/code/trojan-r$ PATH=$PATH:/home/xxx/code/openwrt/staging_dir/toolchain-mipsel_24kc_gcc-8.4.0_musl/bin/  OPENSSL_DIR=/home/xxx/code/openwrt/staging_dir/target-mipsel_24kc_musl/usr/ OPENSSL_STATIC=yes cargo build --target mipsel-unknown-linux-musl  
```

另外，因为native-tls的原因需要在/home/xxx/code/openwrt/staging_dir/toolchain-mipsel_24kc_gcc-8.4.0_musl/bin/目录下增加到mipsel-unknown-linux-musl-gcc到mipsel-openwrt-linux-musl-gcc的符号链接。


# Trojan-R

高性能的 Trojan 代理，使用 Rust 实现。为嵌入式设备或低性能机器设计。R 意为 **R**ust / **R**apid。

**Trojan-R 目前为实验性项目，仍处于重度开发中，协议、接口和配置文件格式均可能改变，请勿用于任何生产环境。**

## 特性

- 极致性能

    牺牲部分灵活性，采用激进的性能优化策略以极力减少不必要的开销。采用[更高效](https://jbp.io/2019/07/01/rustls-vs-openssl-performance.html)的 `rustls` （相较 openssl）建立 TLS 隧道以提升加解密的性能表现。

    使用 tokio 异步运行时，允许 `Trojan-R` 同时使用所有 CPU 核心，保证低时延和高效的吞吐能力。

    > 需要更多 benchmark 数据和更多优化

- 低内存占用

    Rust 无 GC 机制，内存占用可被预计。简化的握手和连接流程，仅使用极少的堆内存和复制。

    > 需要更多 benchmark 数据和更多优化

- 简易配置

    使用 toml 格式配置，仅需数行配置即可启动完整客户端或服务器。

- 内存安全

    使用 Rust 语言实现，可证明的内存安全性。在语法层面保证所有内存操作安全可靠。无竞争条件，无悬挂指针，无 UAF，无 Double Free。

- 密码学安全

    使用 `rustls` 建立 TLS 加密安全信道，过时的或不安全的密码学套件[均被禁用](https://docs.rs/rustls/0.18.1/rustls/#non-features)。`Trojan-R` 强制开启服务器证书校验以防止中间人攻击。

- 隐蔽传输

    `Trojan-R` 使用 TLS 建立代理隧道，难以从正常 TLS 流量中被区分。支持协议回落，在遭到主动探测时将与普通 TLS 服务器表现一致。

- 跨平台支持

    `Trojan-R` 可被交叉编译，支持 Android， Linux，Windows 和 MacOS 等操作系统，以及 x86，x86_64，armv7，aarch64 等硬件平台。

## 非特性

由于与项目的设计原则冲突，下列特性不计划实现

- 统计功能，包括 API 和数据库对接等

- 路由功能

- 用户自定义协议栈

- 透明代理

如果需要实现上述功能，请使用其他类似工具与 `Trojan-R` 组合实现。

## 设计原则

- 安全性

    `Trojan-R` 不涉及底层操作，且目前的性能瓶颈与其无关，无使用 unsafe rust 的必要。协议回落和 TLS 配置等安全敏感代码经过仔细考虑和审计，同时也欢迎更多来自开源社区的安全审计。

    目前 `Trojan-R` 使用 `#![forbid(unsafe_code)]` 禁用 unsafe rust。如未来有必要使用 unsafe rust 时，必须经过严格审计和测试。

- 使用静态分发而非动态分发

    协议实现使用统一的 trait。协议嵌套使用静态分发，以保证嵌套协议栈的函数调用关系在编译时被确定，使编译器可以进行内联和更好的优化。

- 低内存分配

    减少热点代码的内存分配，用引用替换复制，以实现更高的性能和更低的内存开销。

- 简洁

    保持最简洁干净的实现，以保证最低的代码复杂度，尽可能少的性能开销，并增加可靠性和减少攻击面。

## 部署和使用

`Trojan-R` 使用 toml 进行配置，参考 `config` 文件夹下配置文件。

## 编译

```shell
cargo build --release
```

交叉编译基于 `cross` 完成，编译前请确认已经安装 `cross` (`cargo install cross`)

```shell
make armv7-unknown-linux-musleabihf
```

编译默认开启链接时优化，以提升性能并减小可执行文件体积，因此编译耗时可能较其他项目更长。

编译完成后可以使用 `strip` 去除调试符号表以减少文件体积。

## TODOs

- [ ] 更完善的交互接口和文档

- [ ] 更多的单元测试和集成测试

- [ ] 性能调优

- [ ] 可复现的 benchmark 环境

- [ ] 实现 lib.rs 和导出函数

- [x] 分离客户端和服务端 features

- [ ] Github Actions

## 致谢

- [trojan](https://github.com/trojan-gfw/trojan)

- [shadowsocks-rust](https://github.com/shadowsocks/shadowsocks-rust)
