---
title: "Merkle树的诞生及演进"
date: "2023-11-13"
draft: false
author: "TommyLike"
authorLink: "https://github.com/TommyLike"
description:  "Merkle树的诞生及演进."
resources:
    - name: "featured-image"
      src: "featured-image.webp"
tags: ["Linux", "Merkle Tree", "SigStore", "Web PKI"]
categories: ["Linux"]
lightgallery: true
---

# Merkle 树的诞生及演进
## 诞生
在介绍Merkle树之前，先回顾下另一个更为广泛的技术——Hash(哈希也叫散列)，他能把任意长度的输入，通过散列算法变换成固定长度的输出，输出就是Hash值，自1953年诞生以来，已经衍生了很多有名的Hash算法，比如MD系列，SHA系列等算法，在理想的情况下，Hash能做到两点:

1. 相同的输入得到的结果一定相同
2. 不同的输入得到的结果一定不同
![Hash Function](/images/merkle-tree/hash-function.webp)

基于此，Hash被广泛应用在文件校验，标识，加密等领域，举个例子，openEuler每次版本发布的ISO文件都会包含其sha256的Hash文件(见下图)，当用户文件下下来后，我们可以通过对下载文件再次进行`SHA256 Hash`运算并与sha256sum文件内容比较，以此来判定文件的完整性。当然在实际的场景中，校验的情况通常会更加复杂，可能涉及多个文件，多个来源，多种时刻等，随着技术的演进，便有了今天的要介绍的Merkle树，1979年来自于斯坦福大学的美国计算机科学家Ralph Merkle在他的论文[《基于常规加密函数的数字签名》](https://people.eecs.berkeley.edu/~raluca/cs261-f15/readings/merkle.pdf)中首次提出了哈希树的概念，这便是今天Merkle树的雏形。
![openEuler社区发布的ISO文件及Hash文件(算法为sha256)](/images/merkle-tree/openeuler-example.webp)

openEuler社区发布的ISO文件及Hash文件(算法为sha256)

## 定义
按照维基百科的定义，Merkle树是一种树形结构数据，每个叶子节点以一个数据块的哈希信息为标签，而其他节点均以其子节点的标签哈希为标签，整个结构自下而上，简单直观，能够高效，安全的验证大型数据的内容。注意: Merkle树本身没有对树的类型做规定，但是一般情况下我们说的Merkle树为二叉树。
![Merkle Tree](/images/merkle-tree/merkle-tree.webp)

## 基础应用——Git
`git clone，git add...` 作为程序员每天都会跟git打交道，自2005年诞生以来，目前已是使用最为广泛的代码管理系统。在git的内部，存储文件时有个基础对象——**Tree，**其结构就是一棵Merkle树(严格来说，是一个跟Merkle树非常相似的M[erkle DAG](https://docs.ipfs.tech/concepts/merkle-dag/)，我们这里不做区分，都算作Merkle树)，这是Merkle树最基础的应用。
展开来说，Git内部在存储文件时，他会把每个文件拆解为两部分，一部分用于保存文件的文件内容，用Blob对象表示，另一部分如文件名等元数据信息会作为Node保存在我们的这颗树中:

### Blob
用于保存对应文件的内容，相比于原始的提交文件内容，blob文件会按照新格式重新组装内容并压缩，以一个内容为`Hello World` 名为`real_name.txt`的文件举例，生成的blob文件信息以下图举例，注意: git内部每个存储的文件都会以blob的形式存储，同时每个文件的多个版本也会用多个独立的blob保存。
```bash
#Hello World会以Blob形式存储，具体格式如下:
#1. Blob文件内容: blob+内容长度+原始文件内容
Blob Content = Zip("blob 11 Hello World\0") 
#2. Blob文件名: 其文件内容的SHA1摘要
Blob File Name = SHA1("blob 11 Hello World\0") #bd9dbf5aae1a3862dd1526723246b20206e5fc37
```
### Tree

Git代码仓都是由文件夹及文件构成的，有了Blob存储文件内容，剩下的文件夹结构便是用Merkle Tree的形式存储，这颗树中每个非Root节点代表一个具体的文件(Blob)或者是一个子文件夹(Tree), 要注意的是每个文件对应的文件名，文件权限等信息也会附加在这节点中，有了以上的信息，我们以一个实际的仓库状态为例：
```bash
➜  git-demo git:(main) tree -L 4
.
├── dir3
│   ├── file3.txt
│   └── file4.txt
├── file1.txt
└── file2.txt
```

他的结构非常简单，2个文件及一个包含2个文件的子文件夹，转成Tree+Blob的形式后，结构则如下，其中每个上层节点的Hash都是由下层节点的Hash计算而来，这与我们Merkle树的定义是匹配的，Git也正是借用这结构，非常方便的用一个Root Hash表示仓库的独立的状态（Snapshot）。
![Git Tree Object](/images/merkle-tree/Git-Tree-Object.webp)
一旦我们的某个文件发生变更，只需要结合变更后节点的Blob文件便能重新组装新的树（包括计算新的Root Hash），以修改`file1` 内容举例，则更新后的Tree如下图所示。
![Two Git Tree Objects](/images/merkle-tree/Two-Git-Tree-Objects.webp)
不难发现，引入Merkle树后，Git能非常灵活和高效的计算并跟踪仓库的状态，这也是Merkle树最为核心的优势，另外，这里我们只围绕Merkle树对git的相关内部细节做了介绍，实际上Git完整的设计和概念会复杂很多，如果感兴趣，可以参考Git的官方指导[ProGit](https://git-scm.com/book/en/v2)。

## 基础应用——P2P

Merkle树另一个熟悉的场景便是P2P中的下载校验，还是以我们上文提到的`openEuler ISO checksum`方式，虽然能保护文件的完整性，但是因为Hash是基于整个文件计算来的，一旦校验失败就需要全部重新下载，因此效率相对低下，同时，下载的速度也会跟服务的出口带宽，地理位置及带宽占用率相关。
为此便有了P2P的方案，他可以同时从多个对端下载同个文件，从而降低服务端压力，不过因为文件往往会被拆分为多个文件块，也由此带来了新的问题，一个是怎么验证对端数据可信，一个是怎么验证合并文件的完整性问题。早期，社区引入了Hash列表的概念，以下图为例，每个文件块均会包含独立的hash信息，同时会有一个总Hash信息确保文件块Hash未篡改，客户端在下载的时候，需要先获取下载文件的文件块Hash列表信息，下载完成后再单独计算校验，这样做的好处是在多个源同时下载，且每块可以独立校验。
![Hash list with top hash](/images/merkle-tree/Hash-list-with-top-hash.webp)早期的BitTorrent便是采用这种方式对传输的数据进行定义和校验的，我们贴一个具体的例子（CentOS 7 x86_64 DVD ISO BitTorrent文件）:
```bash
{
   "announce": "http://torrent.centos.org:6969/announce",
   "announce-list": [
      [
         "http://torrent.centos.org:6969/announce"
      ],
      [
         "http://ipv6.torrent.centos.org:6969/announce"
      ]
   ],
   "comment": "CentOS 7 x86_64 DVD ISO",
   "created by": "mktorrent 1.0",
   "creation date": 1604673869,
   "info": {
      "files": [
         {
            "length": 2495,
            "path": [
               "0_README.txt"
            ]
         },
         {
            "length": 4712300544,
            "path": [
               "CentOS-7-x86_64-DVD-2009.iso"
            ]
         },
         {
            "length": 398,
            "path": [
               "sha256sum.txt"
            ]
         },
         {
            "length": 1258,
            "path": [
               "sha256sum.txt.asc"
            ]
         }
      ],
      "name": "CentOS-7-x86_64-DVD-2009",
      #每个piece的大小
      "piece length": 524288,
      #每个piece的hash值按照序号进行字符拼装
      "pieces": "<hex>SHA1 of piece1 + SHA1 of piece2 + SHA1 of piece3...<hex>
}
```
以上的Json格式中，我们需要关注BitTorrent文件结构中的`piece length`和 `pieces` 两个字段:

1. **piece length**: 文件分块后每块的具体大小
2. **pieces**: 所有文件块的SHA1 Hash字符拼装，可以理解为如果整个文件内容包含2块pieces，其文件块的Hash分别为`063bed`和`14f7d3` 那么`pieces`的值则为: `063bed14f7d3`

这种方式解决了单一下载源速度慢和文件Hash校验低效的问题，不过仍旧不够完美——拼接后的`pieces`会根据文件大小而变化，源文件越大，种子文件就越大，比如我们引用的Centos BitTorrent文件差不多有177K(越大的BT文件意味着更长的下载准备时间)，另一个则是不支持单个文件的独立校验，比如校验上文中的README.txt。
由此，便有了基于Merkle树的V2解决方案，在新版本的协议中，作者引入完美二叉树的Merkle树([Simple Merkle Hash](http://www.bittorrent.org/beps/bep_0030.html#bep-3))用于解决以上遇到的问题。我们这里对规范的细节稍微展开:

1. 每个文件均有独立的Hash Tree: 新协议中取消了V1中的Hash列表，相反，他给每个文件设置了独立的Merkle树，BitTorrent文件中会记录每个文件Merkle树的根Hash(`pieces root`)，这样使针对每个文件的校验成为可能。
2. 引入颗粒度更小的block：相对于piece，block的单位更小且固定为16kiB，是每个文件Merkle树的叶子节点，piece本身由block构成，大小可灵活设置，比如2block，4block等，属于中间节点(如果对应文件大小小于设置的piece大小，则piece是根节点)，以上规则对应下面的示例图，可以看到因为引入了block，BT文件独立传输和校验的单位更小，同时BT文件只包含上面提到的根Hash(`piece root`)和文件对应的的Piece的Hash(`piece1和piece2`)信息，因此BT记录的hash会更精简(不绝对，跟具体的BT文件包含的文件大小，数量，及piece的设置有关系)。
![File with 4 blocks and piece size of 32Kib](/images/merkle-tree/File-with-4-blocks-and-piece-size-of-32Kib.webp)
还是以Centos ISO文件举例，如果同样的BT文件换成V2的版本，调整后如下(示例)

```bash
{
    "announce": "example.com",
    "info"：{
        "files tree": {
             "folder": {
                  "filename": {
                       #每个文件会包含Merkle树的根Hash
                       "0_README.txt": {"length": 2495, "pieces root": "<hex string>"},
                       "CentOS-7-x86_64-DVD-2009.iso": {"length": 4712300544, "pieces root": "<hex string>"},
                       "sha256sum.txt": {"length": 398, "pieces root": "<hex string>"},
                       "sha256sum.txt.asc": {"length": 1258, "pieces root": "<hex string>"},
                  },
                  ...
             },
             ...
        },
        "meta version": 2,
        "name": "CentOS-7-x86_64-DVD-2009",
        "piece length": 524288
    },
    #一个文件如果被拆分为多个Piece，这里会包含Piece的Hash拼接，不是每个文件都会有，只有文件大小超过piece大小的才会有记录。
    "piece layers": {
        "CentOS-7-x86_64-DVD-2009.iso": "SHA256 of piece1 + SHA256 of piece2 + ....",
        ...
    },
}
```

以上Merkle树的应用中有一个非常关键的点——怎么对传输的文件块做验证(比如验证上图piece1/block0是正确的)，这里我们补充介绍下Merkle树中一个比较重要的概念，包含证明(**Inclusion Proof**):

### 包含证明(Inclusion Proof)
包含证明要解决的问题就是怎么证明一个节点在当前的树中，需要全量获取这棵树的叶子节点吗？实际是不需要的，以下图的8个叶子节点的树举例:
![Inclusion Proof](/images/merkle-tree/Inclusion-Proof.webp)
假设需要验证L1在树中，我们只需要获取L1节点每层的兄弟节点(sibling)计算结果并以此往上最终比较与根节点的一致性饥渴, 整个流程如下:

1. 计算L1的Hash，得到`Hash000`
2. 获取节点`Hash001`，与步骤1的结果合并计算得到`Hash00`
3. 获取节点`Hash01`，与步骤2的结果合并计算得到`Hash0`
4. 获取节点`Hash1`，与步骤3的结果合并计算得到`Root Hash`
5. 与获取到的`Root Hash`做比较

由此我们可以看到一颗高度为4，容量为8的树，为了验证某个叶子节点的`包含` ，我们只需要获取4个节点的Hash即可，另外，在BT的规范定义中，也把当前节点的上层兄弟节点称作叔叔节点(uncle), 在BT的通信协议中明确规定了通过兄弟节点及叔叔节点完成校验的规范，我们这里仅摘要关键部分，感兴趣的话可以查看BT2原始设计中关于Simple Merkle Tree的介绍[SPEC](http://www.bittorrent.org/beps/bep_0030.html#bep-10):

`A Tr_hashpiece message consists of an index, begin, hashlist and piece. The hashlist consists of **the piece's own hash, the piece's sibling hash, and the uncles of the piece up until and including the root hash** (see above and Discussion).`

`……`

`Upon receipt of a Tr_hashpiece message, **the receiver recomputes the root hash using the hashlist and compares it to the root hash in the Merkle torrent. If they match, all the hash values are saved in the receiver's own hash tree**, such that they can be passed on to others when the piece is downloaded from this receiver. When all subpieces have come in, the piece is checked using the hash from the hash tree.`

## 基础应用——Web PKI

在我们熟知的网络传输协议中，由权威的CA机构，通过证书链给网站颁发证书，浏览器通过内嵌的CA根证书校验网站证书的安全性，并以此建立安全传输协议是最为广泛的应用模式，不过，这个流程也存在一定的安全风险的，比如2021年来自荷兰的证书机构DigiNotar遭到黑客入侵，在不知情的情况下为Google，Yahoo，Facebook等公司颁发了数百张假证书，入侵事件造成了非常广泛的影响，直接导致了DigiNotar根证书的撤销，公司也在同月破产。为了规避该问题，一种针对证书颁发机构的监控和审核机制被提了出来，这便是2011由Google提议，2013年正式发布的Certificate Transparency([RFC6962](https://datatracker.ietf.org/doc/html/rfc6962))规范。这里我们先简单科普下什么是Certificate Transparency。

### 证书透明度(Certificate Transparency)

首先CT(以下均已CT代表Certificate Transparency)不是某个单独的概念，而是一个体系，其中既包含具体的协议规范，也定义了体系中参与的角色，行为等。CT里面的核心是`Certificate`和`Transparency`，Certificate即是我们的上文提到的网站TLS证书，Transparency则是一个新的机制，目标是确保证书颁发的行为是可监控和审核的，我们贴上RFC6962中的描述：

`This document describes an experimental protocol for publicly logging the existence of Transport Layer Security (TLS) certificates as they are issued or observed, in a manner that allows anyone to audit certificate authority (CA) activity and notice the issuance of suspect certificates as well as to audit the certificate logs themselves.`

`The intent is that eventually clients would refuse to honor certificates that do not appear in a log, effectively forcing CAs to add all issued certificates to the logs.
Logs are network services that implement the protocol operations for submissions and queries that are defined in this document.`

可以看到这里面有两个关键的要素:

1. 设置公开可查询的日志服务记录颁发的证书信息
2. 有机制支持客户端对日志中的证书合法性做校验

有日志服务记录颁发的证书信息比较简单，那怎么做到公开的信息支持客户端校验呢？使用Merkle树，我们可以在日志服务通过Merkle树记录每个证书信息，有了前文提到的**Inclusion Proof** 我们能很方便的查证某个证书是否在当前的Merkle树中，其次因为每次添加一个证书节点，我们就会生成一个新的Merkle树(根Hash变更)，所以我们还需要有方法能证明当前的树确实是之前版本添加多个节点后演变的，是一致的，由此就有了第二个概念——一致性证明(**Consistency Proof)**

### 一致性证明(Consistency Proof)
![Consistency Proof](/images/merkle-tree/Consistency-Proof.webp)
上图为一棵Merkle树在添加节点中的三个状态，我们用紫色标注相较左图新增的节点，怎么证明三个状态是一致的呢？核心是递归，将一棵树的一致性的问题拆解为左子树，右子树分别一致的两个问题。举例，当我们有一颗叶子节点为M的Merkle树，同时需要证明他是叶子节点为N的子树演变而来的$(N<M)$,  假设$K=2^{X}$ 且是不超过M的最大值(可参考上图中三棵树的注释), 那算法就变成了:

1. 当$N\leqq K$ 时，M的右子树肯定不在N树中，则我们只需要证明M的左子树跟N这颗子树是一致的(不是指相等)，然后结合M右子树的根Hash计算最终Hash是否一致即可, 如图中的`Tree2`到`Tree3`。
2. 当$N\ge K$ 时，M的左子树肯定包含在N的左子树中，则我们只需要证明M与N左子树是相等的，然后结合M右子树于N的又子树是一致的(不是指相等)即可，如图中的`Tree1`到`Tree2`。
3. 按照以上算法对剩下的子树进行一致性证明，直到比较的两课子树是相等的。

按照这个逻辑，为了证明Tree1跟Tree3这两棵树是一致的，我们只需要提供节点信息[c, d, g, l] 。有了以上的基础知识，我们再以官网的[示例](https://certificate.transparency.dev/howctworks/)继续介绍certificate transparency的整体逻辑。
### 证书透明度系统(Certificate Transparency System)
![Certificate Transparency System](/images/merkle-tree/transparency.webp)
1. 网站管理员向CA机构申请证书, 这一步跟我们正常的证书申请是一致的。
2. CA机构生成带SCT信息的证书
    1. CA 机构生成一个特殊的证书(precertificate，可以理解成是一种做了特殊标记的证书，他跟真正要签发给网站的证书内容保持一致，但是没有被签名，也不能被正常验证，CA也需要对这个percertificate做签名，生成一个名为Percertificate Signing Certificate的证书），这之后，CA机构将这个特殊的证书提交给CT日志提供商(注意，这里提交的时候除了上面提到的2个证书，也需要附带从Root CA到CA机构的完整证书链，目的是为了让日志服务商也能对Percertificate Signing Certificate的合法性做校验)。
    2. CT日志服务商会根据提交的证书生成一个名为SCT(Signed Certificate Timestamp)的结构, 可以理解成是对提交的percertificate做一次签名并补充元数据，再将SCT在组装到他自己维护的Merkle树中去，他会包含以下三个关键信息：
        1. 签名信息(signed): 证明这个SCT是CT日志服务商签名过的
        2. 证书信息(certificate): 确认SCT跟证书的绑定关系
        3. 时间戳(timestampe): 日志服务商承诺会将节点信息添加到Merkle树的时间(MMD: *maximum merge delay*)

   c. CT日志服务商生成的SCT信息并返回给CA机构

3. CA机构将SCT信息附加到最终的生成证书中(在名为`SignedCertificateTimestampList`
   的扩展字段中)，并返回给网站网站管理员。
4. 网站基于CA颁发的证书构建其网站页面并运行维护。
5. 网站浏览器（客户端）在建立连接的时候，会对网站证书中的SCT信息做校验，一般在TLS握手阶段client主动向server请求SCT信息，或者是在OCSP协议中获取，一旦客户端拿到SCT就可以验证SCT的签名信息或者是从日志服务器发起请求从而验证证书的有效性， 具体可参考(链接)(https://github.com/google/certificate-transparency/blob/master/docs/SCTValidation.md)。
6. 监控机构持续观察CT日志，以识别其异常行为，比如因为黑客攻击造成的恶意签发证书（CA机构无法抵赖）。

### 证书撤销透明度(Revocation Transparency)

CT提供了一个非常便捷的方式，用于审核CA机构的异常行为，不过在完整的证书签发使用流程中，还有一个问题没有解决，假设因为秘钥泄露的原因，服务商需要撤销证书怎么办呢？CT本身是Append Only，一旦颁发证书就会一直存在CT中无法删除，这里我们马上要介绍的Revocation Transparency[1](http://www.links.org/files/RevocationTransparency.pdf)（以下简称RT）便是为了解决此类问题，RT跟我们上文提到的CT技术内核相似，也是基于Merkle树的，但是他可用作记录证书的吊销状态，开始之前，我们要介绍Merkle树的另一个变种——Sparse Merkle Tree

### 稀疏Merkle树及不包含证明(Sparse Merkle Tree & None-inclusion Proof)

上午提到的Merkle树做Inclusion Proof是很简单的，但是要证明节点不在Merkle树中（None-inclusion Proof）会非常麻烦，总不能把整个Merkle树都开放给用户下载校验，Sparse Merkle Tree（简称SMT）在Merkle树的基础上引入了2个变化：

1. 树的大小固定，他不会跟随节点的插入而变更
2. 新增节点的位置固定，由添加内容的Hash决定树中的位置。

我们以一个相对小规模的SMT树举例子:
![None-inclusion Proof](/images/merkle-tree/_None-inclusion-Proof.webp)
上图中是一个固定大小为8的SMT树，我们定义添加节点时Hash算法调整为 $digest = sha256(leaf) mod  8$  可知:

1. 通过计算hash我们能获取每个元素的具体插入位置(0x000到0x111)
2. 对于没有插入元素的叶子节点我们设置为null。

有了以上两个信息，我们要证明节点A不存在树中，假设其Hash对应的Index为0x010，我们只需要通过null， 以及节点`Hash011，Hash00，Hash1`的信息并最终与根节点比较即可(核心根Inclusion Proof一致，只是目标节点的内容从节点Hash变成看nul)。
在实际在使用的过程中SMT的规模会非常大，一般直接使用sha256为摘要算法，这也就是说整个树的叶子节点规模达到$2^{256}$次方，这是一个超级大的数字，基本不用考虑节点用完的情况。另外一点，考虑到大部分节点都是空值（取名Sparse Merkle Tree的原因）在具体实现的过程中是可以优化和压缩的，如果感兴趣，可以进一步查询这篇关于高效存储SMT树的介绍[Efficient Sparse Merkle Trees](https://eprint.iacr.org/2016/683.pdf)。
另外，SMT往往也会和我们上面提到的第一种类型的Merkle树配合使用，主要是为了监控和审计SMT树的变化，即证明SMT树本身的一致性，这里面最常见的做法就是把SMT的`Root Hash`作为一个新节点，添加到另一棵Merkle树中，示意如下:
![merkle-tree](/images/merkle-tree/complex-merkle-tree.webp)

## 基础应用——Golang Module
在go语言中，我们使用Module对go语言使用的模块做管理，在早期的版本中，当我们执行go get的时候，系统会直接从对应的VCS系统（如github）上面下载对应版本的软件包到本地，后来在1.13版本中引入了go module proxy服务，配置proxy后，同样的行为，会统一经过中间层的proxy服务，而proxy会将所有经过他的mudule文件缓存为不可变的压缩包，这样的好处是一个是后续下载速度更快，一个是避免上游出现不可用，以及中途使用出现恶意篡改导致本地Module受感染的问题，具体看下图:
![Golang Proxy](/images/merkle-tree/Golang-Proxy.webp)
在引入go proxy后我们再来看，常见的go get命令，当我们添加某个包依赖的时候，能看到go.mod和go.sum文件中会记录项目当前使用的依赖包版本及Hash信息

```bash
➜  golang-packages cat go.mod 
module golang-packages
go 1.20

require golang.org/x/text v0.12.0 // indirect
➜  golang-packages cat go.sum 
golang.org/x/text v0.12.0 h1:k+n5B8goJNdU7hSvEtMUz3d1Q6D/XW4COJSJR6fN0mc=
golang.org/x/text v0.12.0/go.mod h1:TvPlkZtksWOMsz7fbANvkp4WM8x/WCo/om8BMLbz+aE=
```

go.mod我们最为常见这里不做展开，主要看看今天的主角——`go.sum`文件，我们的示例图中，引入了版本为1.20的text包，我们在`go.sum`文件中看到他有两行记录，分别对应的是text软件包1.20整个压缩包对应的Hash和软件包中mod文件的Hash, 有了它，系统能对下载的文件做校验，保证文件的完整性，但是会存在几个问题：

1. 假设你访问的proxy服务不可信，缓存服务器中的文件被修改了怎么办？
2. 假设proxy指向的上游VCS在你使用前就已经受到入侵，源文件已经遭到恶意修改怎么办？

因为go get本身是依赖上游(Proxy和VCS)是可信任的，因此对于以上的两个问题都无法感知，为此golang团队在1.13版本同步引入了Checksum服务(https://sum.golang.org)做额外的校验，其内部也是基于Merkle树的，整体逻辑是将使用(发布)的每个软件包的hash信息都记录到公开透明的日志服务中去，开放给大家做验证和审核，补充Checksum服务后，整体的流程升级为以下图所示：
![Go Proxy with Checksum Database](/images/merkle-tree/Go-Proxy-with-Checksum-Database.webp)
还是以上面的1.20版本的text包为例，当系统下载这个包时，会优先向Checksum服务查询软件包的checksum信息(如果Checksum服务相应的记录也为空，则服务本身会先从上游获取相关信息添加到Merkle树后再返回):

```bash
➜  ~ curl https://sum.golang.org/lookup/golang.org/x/text@v0.12.0
18876902    #节点index
golang.org/x/text v0.12.0 h1:k+n5B8goJNdU7hSvEtMUz3d1Q6D/XW4COJSJR6fN0mc=   #module hash信息
golang.org/x/text v0.12.0/go.mod h1:TvPlkZtksWOMsz7fbANvkp4WM8x/WCo/om8BMLbz+aE=  #mod 文件 hash信息

go.sum database tree
19238995   #当时merkle树大小
rYNqIZewerbOSOnMYPyCJKxl86UaEBZ1mrEpfiAQhcs=    #当时merkle树的根节点hash

— sum.golang.org Az3grut2xBvodM8V9oOPG46SgC7w8BqMN9z5tsbaN9nTEEIPNjhvYZJ4aQOzCy8qW2LEaJBANgKjGlfW42nf3DmtVwc=
```

checksum服务的lookup API是专门用来查询软件包对应的hash信息的，比如这里的`lookup/golang.org/x/text@v0.12.0` 他返回的结果中有几个关键信息

1. 软件包和软件包中mod文件的hash信息，我们可以通过这个与下载文件本身的hash做对比。
2. 添加软件包时树大小
3. 添加软件包时树的根Hash

这里我们还需要介绍另一个checksum服务的API, `latest` ，通过他我们可以查询到当前Merkle树的最新状态(大小和根Hash)

```bash
➜  ~ curl https://sum.golang.org/latest
go.sum database tree
19254946
h9eoRKj4A/u1agb0ArWU744FdzF7o/QztgC64L2t81M=

— sum.golang.org Az3grvvtPUIwSm8opaNwI5N/xULaFZ0ecAWzWKqGSBGFNwg8tntSSgJui9xh4e/70QazZqhdO1n6bsR2EABgWXeQTwc=
```

有了以上两个信息，我们就可以对软件包进行`包含证明`以及`树的一致性`做校验了，校验的逻辑如下:

```bash
#假设我们要验证软件包B，lookup查询出来的他的Index是R，本地缓存的树大小为N
验证(数据B 是 记录R):
    如果 R ≥ 缓存.N:
        N, T = 服务器.Latest()  #获取树最新的信息
        如果 服务器.TreeProof(缓存.N, N) 验证不通过:   #一致性证明
            验证失败
        缓存.N, 缓存.T = N, T
    如果 服务器.RecordProof(R, 缓存.N) 验证没使用 B:  #包含证明
        验证失败
    信任 B 是 记录R
```

Checksum 服务中Merkle的应用虽然跟CT中几乎一致，不过在实现的时候，他还引入了另一点优化——**瓦片化**

### Merkle树优化——瓦片化(Tiling)

瓦片化会把树分割为固定的多个高为 $H$，宽$W=2^{H}$的瓦片，我们用$Tile(L, K)/W$ 用来表示一个具体的瓦片，其中L为瓦片所在的层级(Level)，K为瓦片所在层级从左往右的序列号，W用于表示节点总数（当前仅当区域节点不满时）。 下面是一棵拥有 27 个数据记录的哈希树，被分割为多个高度为 2 ，长度为4的瓦片，存储瓦片时规定只存储每个瓦片的叶子节点，上层的节点通过临时计算得出，这样做能有效节省存储空间，比如$Tile(1,0)$原本要存储6个节点，瓦片化后只需要存储4个节点，空间能减少1/3，同理因为操作的单位都换成了$Tile$，则我们客户端在做验证时，所需要传输的数据也变少了，从而减少网络带宽。
真正实现时，底层的设计会更为复杂，如果读者感兴趣，可以移步[设计文档](https://research.swtch.com/tlog)
![Tiling ](/images/merkle-tree/merkle-tree-with-tile.webp)
## 总结

作为一项大家耳熟能详的技术，Merkle树还被应用在了很多我们熟知的应用中，比如存储中的IPFS，数据库的Apache Cassandra和Amazon Dynamo，区块链的Ethereum，以及软件安全领域的Sigstore，这里没有一一展开。这篇文章试图从Merkle Tree的诞生及相关技术发展演进的角度，为大家提供相对完整的介绍，希望能为大家后续的工作提供参考和借鉴。