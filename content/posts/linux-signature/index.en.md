---
title: "操作系统领域签名系统构建"
date: "2023-11-11T09:28:31+08:00"
draft: false
author: "TommyLike"
authorLink: "https://github.com/TommyLike"
description:  "Building a sign system in Rust language."
resources:
    - name: "featured-image"
      src: "featured-image.webp"
tags: ["Linux", "Rust", "openPGP", "X509", "TEE"]
categories: ["Linux"]
lightgallery: true
---
社区签名系统自构建以来，经常会遇到签名阻塞的问题，这个问题在一直存在，但一直没有一个比较好的解决方案，今年我们在社区中引入了一个新的签名系统，整个系统基于Rust语言开发，采用CSP模型，异步框架，TEE等技术，相比现有的签名系统，整个系统在性能，安全，管理等方面都有很大的提升，本文将介绍整个系统的设计思路，技术实现和未来规划。
<!--more-->
# 开源社区中构建签名系统的探索

## 背景和痛点
在openEuler社区中，软件包和ISO等文件的签名是一个相对细分但非常重要的场景。自社区成立以来，我们主要依赖openSUSE社区的OBS Sign项目来支持这一功能。然而，在长期的项目维护期间，我们发现该服务存在一些改进的机会。 首先，我们发现当前
签名服务的场景支持有限。社区的签名需求涵盖了openPGP和X509两个密钥体系，然而，OBS Sign主要针对openPGP体系，对X509的支持有所欠缺，并且还未实现相关文件格式的支持。为了满足不同密钥体系的需求以及更多的文件格式，我们需要进一步完善和扩展。 其
次，目前签名服务的管理和维护相对不便。缺乏集中且易用的管理界面使得密钥的更新和迁移流程变得繁琐。此外，当前设计中采用的本地密钥存储方式和http的传输方式也间接增加了安全风险。为了提升管理的便利性和安全性，我们需要引入一个集中化的管
理界面，并考虑更安全的密钥存储方案。最后，签名性能也存在一些不足之处。当前签名服务无法多实例运行，并且后端签名程序的并发性能有限。这导致在进行版本集中签名时出现长时间的卡顿甚至失败情况。为了改善签名的效率和稳定性，我们需要优化签名服务的架构，
使其支持多实例运行，并提升后端签名程序的并发处理能力。
![obs-sign](/images/linux-signature/obs-sign.webp)
基于以上问题，我们决定在社区引入一个新的签名系统，其中最简单的方法是对接公司现有的签名平台，能力和安全性都具备，不过最大的问题是运行界面在公司外网，且短期内外溢能力也不太现实，另外，我们也尝试直接采购硬件加密机或云服务作为基础的签名解决方案，但最终放弃了这些选择，主要是基于以下两个原因：
1. 部署和运维成本：社区基础设施主要依赖云服务，采购硬件加密机会增加设备托管和维护的工作量。在没有实验室的情况下，设备托管变得复杂，而且也间接增加了集群部署的地域限制。
2. 对openPGP体系支持不足：openPGP作为一个独立的体系，无论是在线还是离线服务，都不直接支持，需要我们在实际使用过程中进行解耦和转换。这种不足降低了解决方案的通用性。
由此，我们提议在社区里面构建一个相对独立且通用的面相操作系统领域的签名服务，同时做到系统性能更好，安全性更高，管理界面更友好
## 业界趋势

说起签名系统，不得不提到Linux Foundation下的SigStore项目，该项目主要围绕容器镜像文件提供了全流程的签名验签能力，相比很多其他方案，他实现方案更完备，更便捷，我们这里简单介绍下基于SigStore的签名流程:
![sigstore](/images/linux-signature/sigstore.webp)

1. 开发人员使用Cosign通过 OIDC（Google、Github、Microsoft 等账号）进行身份认证
2. 认证通过后，Fulcio服务给开发者颁发关联邮箱身份的短期证书，也同步给Rekor
3. 开发人员利用Cosign工具使用对应的短期密钥对二进制文件进行签名，并发布，Cosign会将签名和验证证据(包含文件摘要，签名信息，证书)同步给 Rekor服务
4. 使用者在使用二进制文件前可以通过 Cosign使用Rekor数据验证签名的有效性

Sigstore安全的核心在与临时的秘钥对与公开防篡改的记录，我们在项目初期也做过调研，不过使用Sigstore会存在一个比较关键的问题，操作系统社区中签名验签主要基于openPGP体系，包括RPM文件格式，DNF等基础设施都是基于openPGP体系构建的，而Sigstore核心是X509体系，短期也没有支持openPGP的计划，如果要从openPGP切换到X509体系短期来看不仅是工具实现的问题，也是定义和更新标准的问题。

## 解决思路
新的服务主要基于围绕以下几个核心点做改进:

### Rust

在语言选择方面，我们的首要考虑是高性能、安全和跨平台性（特别是客户端）。结合社区的主要开发语言趋势，我们决定尝试使用Rust语言进行开发。在实际的开发过程中，我们发现Rust带来了许多便利，包括程序低内存占用、内存并发安全以及强大的编译错误提示等特性。
然而，在具体的开发过程中我们也遇到了一些问题，包括不限于底层依赖包功能缺失，云服务SDK支持不足等。为了解决这些问题，我们积极参与了Rust生态系统的发展，并做出了一些贡献，包括：

1. efi-signer：这是一个我们自行开发的针对EFI镜像文件签名的命令行工具，支持签名、验签和证书格式转换等功能，目前已经在Rust的crate上发布。
2. kernel-module-signer：这是一个针对Kernel Module文件签名的命令行工具，其功能与Kernel中的sign-file工具相同。同样地，它也在Rust的crate上发布，为开发者提供了一个便捷的KO签名工具。
3. rpm-rs：这是一个针对rpm包体系的文件解析和组装库，通过我们的推动，目前已经实现了对RSA/EDDSA算法的支持，这为使用Rust的开发者提供了在处理rpm包时的更多灵活性和功能性。

![rust](/images/linux-signature/rust.webp)

### CSP模型+异步框架

签名场景中有两个特点最为明显，一个是大量待签名文件内容的二进制传输，一个是频繁的文件读写操作，基于以上两点，我们在新系统设计中引入了CSP模型，端到端的异步框架和gRPC Stream，基于channel的CSP模型能有效减少依赖，每个过程相互独立，从而提高系统并发度，同时抽象FileHandler负责具体待签名内容拆解和签名信息组装的功能。

```rust
pub trait FileHandler: Send + Sync {
    fn validate_options(&self, sign_options: &HashMap<String, String>) -> Result<()>;
    async fn split_data(&self, path: &PathBuf, sign_options: &mut HashMap<String, String>) -> Result<Vec<Vec<u8>>>;
    async fn assemble_data(&self, path: &PathBuf, data: Vec<Vec<u8>>, temp_dir: &PathBuf, sign_options: &HashMap<String, String>,) -> Result<(String, String)>;
}
```

整个签名过程客户端的运行流程示意如下:
![async](/images/linux-signature/async.webp)
结合全流程的异步框架和GRPC Stream，整个系统在降低内存占用的同时，能非常明显的提升签名性能，我们以签名openEuler21.09版本中的所有源码包举例(大概有4100个软件包，空间占用10GB左右)，在同样的规格，同样并发的情况下，整体会有5到20倍的性能提升，客户端越多效果提升越明显。
![performance](/images/linux-signature/performance.webp)
另外考虑到大文件内容在网络传输占用带宽的问题，下一步我们计划将整个签名信息计算的过程做深入梳理，核心是拆解openPGP和X509签名的内部逻辑，做到在本地完成部分签名数据准备后，只用传输文件的摘要到服务端做最后一步签名计算，预计将进一步减少网络带宽及服务端计算压力。

### HSM+TEE
为了保证私钥的绝对安全，整个系统在设计的过程中对接了云服务中基于硬件加密模块(HSM)的秘钥管理服务，整个系统被设计为三层秘钥结构，其中Master Key存储在云服务的硬件加密模块中，负责整个集群中敏感数据的加解密，而Cluster Key和 DateKey则保存在本地，每次集群启动的时候，会经由Master Key完成一次性的解密工作。
![keys](/images/linux-signature/key-hierarchy.webp)
为了保证整个运行过程的安全，整个秘钥操作的过程也被设计成独立的签名模块，支持整体迁移至TEE环境中，数据出入TEE环境均是加密数据，同时Master Key的加解密调用将严格限制在TEE环境中，从而提升安全等级，整个服务最终的组件架构如下:
![sign system](/images/linux-signature/sign_system.webp)
## 技术实现
### 文件及签名格式解析
针对RPM， EFI等不同文件的识别和签名封装，需要梳理现有文件规范，并参考现有代码逻辑，最终拆解到系统流程中去，我们以RPM举例，整个RPM文件包含Identifier，Signature，Header，Payload四个区域，完整的签名信息包含多个Tag，分别用于存储文件摘要，签名值，内容长度等，每个Tag的定义及解释如下
![rpm](/images/linux-signature/rpm.webp)
1. **RPMSIGTAG_SIZE**: Header和Payload两个区域加一起的大小。
2. **RPMSIGTAG_PAYLOADSIZE**: Payload压缩前的大小。
3. **RPMSIGTAG_SHA1**: Header Section的SHA1摘要。
4. **RPMSIGTAG_MD5**: Header和Payload区域的128位MD5值大小。
5. **RPMSIGTAG_DSA**: Header section的DSA签名信息，跟GPG Header需要同时使用
6. **RPMSIGTAG_RSA**: Header section的RSA签名信息，跟PGP Header需要同时使用
7. **RPMSIGTAG_PGP**: Header section和Payload section的RSA签名信息, 跟RSA Header需要同时使用。
8. **RPMSIGTAG_GPG:** Header section和Payload section的DSA签名信息，跟DSA Header需要同时使用。

我们以RSA的openPGP秘钥签名为例，要最终存储Signature的信息如下:
![rpm](/images/linux-signature/rpm2.webp)
对比KernelModule文件来看，KO签名信息的识别和组装会更加简单，整个文件的签名信息会通过追加的方式补充到原始文件中，包括1. 二进制X509签名CMS信息，2. 签名的元数据`module_signature`以及3. 签名的魔法数字`~Module signature appended~`三个部分。
![kernel-module](/images/linux-signature/kernel-module.webp)
EFI体系会相对复杂，首先，EFI可执行文件是一个 PE/COFF 格式文件，其次整个过程会遵循 Microsoft Authenticode 规范来操作，整个签名的过程如下：
1. 使用`Microsoft Authenticode`中的方法计算EFI文件的摘要（当前只支持sha256）
2. 使用X509秘钥对对上面的摘要进行签名，并将其嵌入一个 `PKCS #7 SignedData` 结构中
3. 以上签名将嵌入到一个`WIN_CERTIFICATE`结构体并最终嵌入到EFI文件中，另外，签名的偏移位置和大小在EFI文件的`optional header` 补充
![efi](/images/linux-signature/efi.webp)

### 多秘钥体系支持
签名系统需要至少支持X509及openPGP两种体系，为保证扩展性，数据存储，加解密后端及具体的秘钥插件均被抽象成独立的组件，可根据需求在后续的迭代中升级替换，以具体秘钥插件举例，实例插件只需要实现SignPlugin中，秘钥生成和导入，签名执行，秘钥文件解析等操作即可:
```rust
pub trait SignPlugins: Send + Sync {
    fn new(db: SecDataKey) -> Result<Self>
        where
            Self: Sized;
    fn validate_and_update(key: &mut DataKey) -> Result<()>
        where
            Self: Sized;
    fn parse_attributes(
        private_key: Option<Vec<u8>>,
        public_key: Option<Vec<u8>>,
        certificate: Option<Vec<u8>>,
    ) -> HashMap<String, String>
        where
            Self: Sized;
    fn generate_keys(&self, key_type: &KeyType, infra_configs: &HashMap<String, String>) -> Result<DataKeyContent>;
    fn sign(&self, content: Vec<u8>, options: HashMap<String, String>) -> Result<Vec<u8>>;
}
```

### 集群与客户端负载均衡
签名系统实际运行的生产环境通常都需要部署多个客户端对接不同的构建机器，为提升系统支撑的集群规模，后端服务实例在设计时也支持水平扩展，这本是一种最为常见的设计，不过在叠加HTTP2.0(gRPC基于HTTP2.0)的多路复用及Kubernetes原生Service场景后，负载均衡会失效，造成局部节点过热的问题，为避免类似问题，我们在结合了基于DNS的客户端RoundRobin路由及Nginx的反向代理的两种负载均衡模式，能实现高效简单的gRPC负载均衡，同时支持集群内外2种访问模式。
![loadbalancer](/images/linux-signature/loadbalancer.webp)

### 客户端身份校验
客户端的身份校验，复用了现有X509秘钥体系中私有Certificate Authority的设计，我们在客户端的配置和调用中引入了基于CRL Distribution Points DP的有效性校验，服务端可以根据实际情况为不同客户端分分发独立的客户端证书，也可随时吊销客户端证书。
```bash
X509v3 extensions:
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Subject Key Identifier:
                F7:40:68:5F:E6:68:13:A6:2E:85:4E:2D:8C:32:98:93:EC:17:27:E7
            X509v3 Authority Key Identifier:
                keyid:E6:87:3C:CF:F5:3C:40:BB:AF:AA:6E:5D:EA:F1:4F:5E:31:A6:0D:7C
                DirName:/CN=Infra/OU=Infra/O=openEuler/L=ShenZhen/ST=GuangDong/C=CN
                serial:A1:DA:F9:9B:F6:C8:2B:42:C8:9F:59:DC:E8:C4:AE:0F

            X509v3 Key Usage: critical
                Digital Signature, Certificate Sign, CRL Sign
            X509v3 CRL Distribution Points:

                Full Name:
                  URI:https://signatrust.osinfra.cn/api/v1/keys/default-x509ica/crl
```

## 项目价值
社区签名系统相对来说场景比较单一，功能简单明了， 整个系统开发的难点来自于跨领域底层知识的研究和应用，其次，遵循来源社区，回馈社区的理念，这个项目自一开始就以开源项目的方式在[社区](https://gitee.com/openeuler/signatrust)运作，我们也欢迎有更多的开发者能参与进来，试用，回馈。最后，签名系统是社区IT基础设施中一环，在19年社区成立之初，社区基础设施服务大多都基于外部开源组件或采购商业服务构建，为了保证服务的自主可控和安全，整个团队做了不少尝试，包括逐步替换国产商业服务；参与上游开源社区，持续改进和维护开源组件并引入openEuler社区；本次的签名服务则是一种新的探索，即从零开始构建一套适配社区的特有服务，为后续其他同类型工作的开展提供了参考价值，当前该服务已持openPGP，X509CA，X509ICA，X509EE秘钥的/证书的生命周期管理(含吊销)，同时支持多种二进制文件文件的批量签名，并已接入EUR，社区SBOM等服务。

## 未来规划
下一步我们将进一步加强服务在全链路安全，签名通用性上面的能力，比如在现有框架基础上新增国密算法支持，引入零信任框架解决客户端服务端身份互认的问题等。同时，上文提到的业界签名方案SigStore，虽然跟我们当期的关注重点存在不少差异，但不可否认的是基于公开防篡改的记录的方式正在被逐渐应用到更多的场景，我们也看到像Archlinux，Redhat等社区也在融合对接的方向做了不少尝试，因此积极推动社区中签名验签标准及周边工具的向前演进也是下一步的重点。

## 仓库地址
Signatrust: https://gitee.com/openeuler/signatrust