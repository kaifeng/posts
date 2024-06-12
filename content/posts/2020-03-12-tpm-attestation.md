---
layout: post
title: TPM远程证明研究
author: kaifeng
date: 2020-03-12
categories: tpm
---

# 术语和缩略语

* TCG: Trusted Computing Group
* TPM: Trusted Platform Module
* EK: Endorsement Key
* AK: Attestation Key
* IAK: Initial Attestation Key
* Attestation: 证明
* Remote Attestation: 远程证明
* Verifier/Appraiser: 校验方/鉴定方，负责检查PCR，确认可信链。
* Attester: 证明方
* PCA: Privacy CA
* TAP: Trusted Attestation Protocol
* PTS: Platform Trust Services


# 远程证明概述

确保系统处于预期状态有很多方法，TPM提供了其中一种。
TCG发布了TAP信息模型（TAP Information Model），该规范描述了角色、消息流向和schema，
通过实现证明协议，可以使用TPM设备校验一个系统的完整性。

证明是指将可校验的证据提供给另一方的过程，在TPM语境下，证据通常是指PCR，
但是任何密码学可校验的关于PCR的状态都可以作为证据，所以证据也可以扩充。

## 相关角色和活动

两个主要的角色是校验方和证明方，校验方向一个系统发送挑战查询，并选择证据的范围，证明方收集指定的证明证据（Attestation Evidence）并提供给校验方以供校验。

证明证据通常包括启动状态、当前配置和身份信息。
EK证书是其中一种身份证明（如果设备身份的凭证在出厂时录入，则可以建立供应链的证据。厂商也可以根据EK（和IAK）及证书确定设备产地），然而EK可以唯一确定一个TPM设备及其关联的系统，为避免对TPM本身的攻击，
在远程证明中，通常使用AK来标识自己的身份，如果TPM未预置AK，可以通过EK生成。如果厂商在出厂前录入了AK，这个AK称为IAK，
它可以避免在系统投入使用时再生成AK。

AK对提交给验证方的证明证据进行签名，验证方收到证明证据后，可以通过密码学方法检查提供的证据是否满足验证方的可信评估策略，以及该设备是否来自批准的厂商。

## 出厂密钥

通常TPM厂商会给TPM录入一个主EK（PEK），并生成对应的证书。EK证书包含了PEK的公开部分及其它有用字段，如TPM制造商的名字，型号，版本，key id号等。
这些信息可以唯一确定一个TPM，如果某设备能安全地与该TPM绑定，也可以用作该设备的标识符。

EK证书可以保存在TPM的NVRAM的固定位置供软件读取，或放在厂商的网站。EK证书由TPM厂商的CA签名（所以校验方需要信任厂商的根证书）。
EK和AK一般不设置密码，即使设密码，也建议设为通用密码（公开的，如calvin之类），
因为EK和AK的密钥由TPM保护，外部无法得到，它的安全性是通过这种方式保证的，是否为密钥加密码对安全性没有实质影响。

## 远程证明过程

在一个系统可以被证明之前，需要先获取TPM厂商生成的EK和AK的公钥及相应的证书，设备是否有AK要看厂商的情况，有的OEM出厂并不带AK。
校验方将这些公钥和证书保存在本地数据库（还可以与内部PKI集成），证明方的预期状态一般也预先保存在校验方的数据库中。

证明操作由校验方发起，可以基于时间间隔或者基于请求。校验方请求证明方的AK和EK证书并进行校验，如果证书有效，发起一个挑战，
挑战携带key id，PCR选择范围，及一个随机的nonce。nonce用于保证数据的时效性，它应该是一个不可预测的随机值，时间戳是一个反例。

证明方使用AK向TPM发起一个TPM2_Quote命令。TPM创建一个TPMS_ATTEST结构的数据并使用AK签名，然后发送给校验方。

校验方检查AK的签名，并检查TPMS_ATTEST结构中的信息，将软件度量日志（Event Log）与本地数据库中的已知正常状态的日志进行对比。

校验方可以将本地数据库中的每一条event log记录与TPMS_ATTEST结构中的对应条目进行对比，检查bootloader, OS, 用户程序的状态，并根据检验过程的结果得到最终仲裁结果。

# 证据收集和Quote

远程证明主要采用Quote命令（也可以采用PCR绑定的key或PCR绑定的数据来进行证明，不是主流）。
Quote是指一组PCR数据及其签名，TPM2_Quote则是对一组PCR寄存器进行签名的TPM命令，该命令为指定的PCR按给定算法和bank输出Quote结果。

Quote输出的结果可以简化为如下表示：

```
| Quote | Nonce | PCR Hash | AK签名 |
```

验证Quote需要用新一些的tpm2-tools版本，这里基于Fedora 31和tpm2-tools 4.0.1版本测试。

先定义两个句柄，0x81010009给EK，0x8101000a给AK。

创建EK：
```
$ tpm2_createek -c 0x81010009 -G rsa -u ekpub.pem -f pem -p "ekpass"
```

创建AK并固化：
```
$ tpm2_createak -C 0x81010009 -c ak.ctx -G rsa -g sha256 -s rsassa \
> -u akpub.pem -f pem -n ak.name -p "akpass"
loaded-key:
    name: 000b7392c8852e3365298d23f8c1e35e104e43e16efee172e507d1784f
c027b19dbd
$ tpm2_evictcontrol -c ak.ctx 0x8101000a
```

模拟一个nonce值并执行Quote命令：
```
$ nonce=e225b230a0ff210ac13c4bc58ea01aa5e9bd4cf4
$ tpm2_quote -c 0x8101000a -l sha256:15,16,22 -q $nonce \
> -m quote.out -s quotesig.out -o quotepcr.out -g sha256 -p "akpass"
quoted: ff54434780180022000b2c2bf37ebe44f3369b499e14d3b5a948f7dec53
ebd81348e22d864fccc8498dc0014e225b230a0ff210ac13c4bc58ea01aa5e9bd4c
f400000000003646d6000000070000000001201706190016363600000001000b030
08041002051cdfd15463a712da38c49e9390d861030e28cf1f19ebe9f5a8b6901a9
df64fc
signature:
    alg: rsassa
    sig: 533d898e79721276373a526ef13d63ae5002ca9604504015afdcb9a305
cd49a0904edc96008991a68c30487ae4083a94353ab1133d77e958b331db76f76cd
4c3d17dcb2d3895ef8328b0bff2cd10d89f8fcdd81fe4a35475993b912871f8da21
3aae4f69e87d88f186813da71fa43ecc53359b5a583d69de05f9d82dcafa7531693
02d0fd4f664869f25ddb5d1eb20e31f8c1d4f9aa5f414aae277d8cc261b064fdc24
7813751bc414464d6adcf2bc65b6e22d8a9ef41bf3a5cafbc62f8ffaea75ca04f2e
2a8d35edabb58d8f4b85e3b3da9a18b6f52dc43fe4f4999f13f136291ef8c4144f4
e3b160ddc0e18fdae7c34488883d28892ce3b8575f9ac303039a
pcrs:
    sha256:
    15: 0x000000000000000000000000000000000000000000000000000000000
0000000
    16: 0x000000000000000000000000000000000000000000000000000000000
0000000
    22: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
FFFFFFF
calcDigest: 51cdfd15463a712da38c49e9390d861030e28cf1f19ebe9f5a8b690
1a9df64fc
$
```

生成的quote.out，quotesig.out及AK的公钥需要发送给验证方。
quotepcr.out是选定的PCR的原始数据，quote中已经包含了对这些PCR数据的哈希，因此可选提供，
tpm2-tools并没有强制。

# 证据解析和校验

## 用checkquote校验证据

tpm2_checkquote命令，可以简单地对quote输出进行检查：
```
$ tpm2_checkquote -u akpub.pem -m quote.out -s quotesig.out \
> -f quotepcr.out -g sha256 -q $nonce
pcrs:
    sha256:
    15: 0x0000000000000000000000000000000000000000000000000000000000
000000
    16: 0x0000000000000000000000000000000000000000000000000000000000
000000
    22: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
FFFFFF
sig: 533d898e79721276373a526ef13d63ae5002ca9604504015afdcb9a305cd49a
0904edc96008991a68c30487ae4083a94353ab1133d77e958b331db76f76cd4c3d17
dcb2d3895ef8328b0bff2cd10d89f8fcdd81fe4a35475993b912871f8da213aae4f6
9e87d88f186813da71fa43ecc53359b5a583d69de05f9d82dcafa753169302d0fd4f
664869f25ddb5d1eb20e31f8c1d4f9aa5f414aae277d8cc261b064fdc247813751bc
414464d6adcf2bc65b6e22d8a9ef41bf3a5cafbc62f8ffaea75ca04f2e2a8d35edab
b58d8f4b85e3b3da9a18b6f52dc43fe4f4999f13f136291ef8c4144f4e3b160ddc0e
18fdae7c34488883d28892ce3b8575f9ac303039a
$
```

模拟一个错的nonce值会导致校验错误：
```
$ tpm2_checkquote -u akpub.pem -m quote.out -s quotesig.out \
> -f quotepcr.out -g sha256 -q "ac"
pcrs:
    sha256:
    15: 0x0000000000000000000000000000000000000000000000000000000000
000000
    16: 0x0000000000000000000000000000000000000000000000000000000000
000000
    22: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
FFFFFF
sig: 533d898e79721276373a526ef13d63ae5002ca9604504015afdcb9a305cd49a
0904edc96008991a68c30487ae4083a94353ab1133d77e958b331db76f76cd4c3d17
dcb2d3895ef8328b0bff2cd10d89f8fcdd81fe4a35475993b912871f8da213aae4f6
9e87d88f186813da71fa43ecc53359b5a583d69de05f9d82dcafa753169302d0fd4f
664869f25ddb5d1eb20e31f8c1d4f9aa5f414aae277d8cc261b064fdc247813751bc
414464d6adcf2bc65b6e22d8a9ef41bf3a5cafbc62f8ffaea75ca04f2e2a8d35edab
b58d8f4b85e3b3da9a18b6f52dc43fe4f4999f13f136291ef8c4144f4e3b160ddc0e
18fdae7c34488883d28892ce3b8575f9ac303039a
ERROR: Error validating nonce from quote
ERROR: Verify signature failed!
ERROR: Unable to run tpm2_checkquote
$
```

tpm2_checkquote并不属于规范定义的TPM命令，它只使用了openssl库（RSA_verify），不使用TPM资源。

## 解析TPMS_ATTEST数据

所有经TPM签名的数据都使用TPMS_ATTEST结构，签名是针对该结构进行的签名（然后封装在TPM2B_ATTEST结构），结构定义如下。
```
Parameter       | Type            | Description
-----------------------------------------------
magic           | TPM_GENERATED   | 恒为TCG
type            | TPMI_ST_ATTEST  | 结构类型，Quote类型为0x8018
qualifiedSigner | TPM2B_NAME      | 签名密钥的名称
extraData       | TPM2B_DATA      | 调用者提供的外部信息（规范未指定）
clockInfo       | TPMS_CLOCK_INFO | Clock, resetCount, restartCount, Safe
firmwareVersion | UINT64          | 固件版本号，由TPM厂商定义
[type]attested  | TPMU_ATTEST     | 结构类型对应的数据
```

前面得到的quote输出如下：
```
[kaifeng@localhost-live quote-test]$ xxd quote.out
00000000: ff54 4347 8018 0022 000b e637 8e65 7f3d  .TCG..."...7.e.=
00000010: 63d5 0da9 35e1 31e3 a91c 26ed 190c 6c06  c...5.1...&...l.
00000020: e553 e32f c912 e1bb ff58 0014 e805 05e7  .S./.....X......
00000030: a0ba 2dc5 b272 5bf1 0ca6 18c4 acd8 4cf6  ..-..r[.......L.
00000040: 0000 0000 0642 cdd7 0000 0007 0000 0000  .....B..........
00000050: 0120 1706 1900 1636 3600 0000 0100 0b03  . .....66.......
00000060: 0080 4100 2051 cdfd 1546 3a71 2da3 8c49  ..A. Q...F:q-..I
00000070: e939 0d86 1030 e28c f1f1 9ebe 9f5a 8b69  .9...0.......Z.i
00000080: 01a9 df64 fc                             ...d.
[kaifeng@localhost-live quote-test]$
```

从quote.out中可以很直观地看到magic和type字段。

qualifiedSigner是一个TPM2B_NAME结构，由一个UINT16的size字段和size长度相关的字节数组的name字段组成（分三种情况，0字节表示没有数据，4字节表示是一个句柄，其它值则表示实际长度），
保存的是签名密钥的密码学名字（哈希类运算）。测试数据中size为0x22，跟随的UNIT16为0x0b表明是一个sha256算法，随后的32字节数据是sha256的哈希值。

extraData是一个TPM2B_DATA结构，与TPM2B_NAME的定义类似，测试数据中的size是0x14，后面20字节是Quote消息，一般就把nonce值当作quote数据。

clockInfo字段是一个TPMS_CLOCK_INFO结构，包含了时钟相关的信息，64位的clock字段表示TPM上电的时间（毫秒），32位的resetCount和32位的restartCount，1字节的safe字段记录与时钟复位相关的信息。

接下来8个字节的firmwareVersion是固件版本号。

attested字段是与Quote实际相关的数据，对于Quote命令，这个字段的类型是TPMS_QUOTE_INFO，定义如下。
```
Parameter | Type               | Description
--------------------------------------------
pcrSelect | TPML_PCR_SELECTION | 摘要算法和PCR范围
pcrDigest | TPM2B_DIGEST       | 使用签名密钥计算的PCR摘要
```

TPML_PCR_SELECTION结构的pcrSelect字段从0x59偏移开始，首先是一个32位的count字段，这里的数据是00 0000 01，即结构体数量为1。
接下来是TPML_PCR_SELECTION结构类型的pcrSelections字段，这个结构包含三个部分：

* hash(TPMI_ALG_HASH)，32位的哈希算法，即00 0b这两个字节，代表PCR摘要是使用sha256计算的。
* sizeofSelect(UINT8)，03表明选择了3个PCR。
* pcrSelect，字节数组，描述了所选PCR的位图，这里是00 80 41，转成对应的二进制：
```
>>> bin(0x418000)
'0b10000011000000000000000'
```

TPM2B_DIGEST结构的pcrDigest字段由一个16位的size和字节数组组成，存放AK使用的哈希算法计算的摘要值，因为AK指定的是sha256，所以这里0x20即32字节，接下来的数据正好为32个字节，消费完整个输出。

测试数据的PCR15，16和22没有度量数据，所以默认值是全0或全F。可以计算验证Quote中的PCR摘要值：
```
>>> import binascii, hashlib
>>> h = hashlib.sha256()
>>> h.update(binascii.a2b_hex('0000000000000000000000000000000000
00000000000000000000000000000000000000000000000000000000000000000
00000000000000000000000000000FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
FFFFFFFFFFFFFFFFFFFFFFFFFFFF'))
>>> binascii.b2a_hex(h.digest())
'51cdfd15463a712da38c49e9390d861030e28cf1f19ebe9f5a8b6901a9df64fc'
>>>
```

与quote.out中的字节流一致。

## checkquote代码分析

前面分析了Quote命令的输出，现在从验证方的角度来分析如何验证这些数据，最直接的方式就是分析checkquote代码。

tpm2-tools所有命令的入口在tpm2_tools.c的main函数，它先进行一些初始化操作，比如初始化上下文，记录命令行入参，然后传给具体的命令。checkquote的入口在tpm2_tool_onrun，主要调用init和verify_signature两个函数。
init的工作是初始化入参和解析Quote数据，前面已经分析过了，所以这里主要分析verify_signature，这个函数主要分为三个部分。

### 读取公钥

读取公钥，前面指定的是akpub.pem，这里读取的就是AK公钥。
tpm2_openssl_get_public_RSA_from_pem是一个辅助函数，
它先尝试按PEM格式读取，如果失败再尝试PKCS#1格式，两者都是通过openssl库实现，
读取后，转换成一个openssl定义的RSA对象。

```
// Read in the AKpub they provided as an RSA object
FILE *pubkey_input = fopen(ctx.pubkey_file_path, "rb");
if (!pubkey_input) {
    LOG_ERR("Could not open RSA pubkey input file \"%s\" error: \"%s\"",
            ctx.pubkey_file_path, strerror(errno));
    return false;
}
RSA *pub_key = tpm2_openssl_get_public_RSA_from_pem(pubkey_input,
        ctx.pubkey_file_path);
if (pub_key == NULL) {
    LOG_ERR("Failed to load RSA public key from file");
    goto err;
}
```

### 检查签名

检查签名与摘要是否相符，如注释所述，目前工具只支持rsassa格式的签名（另一个是ECDSA）。
这里使用openssl的RSA_verify函数进行校验。

tpm2_openssl_halgid_from_tpmhalg将TPM定义的哈希算法定义转换成openssl的哈希算法定义。
因为指定的是sha256算法，因此这里的处理就是用sha256计算消息摘要，用公钥解密签名，然后对比两者是否一致。
如果一致，表示Quote中的数据的确是AK所签名，便可进一步校验PCR和nonce。

```
// Get the signature ready
if (ctx.signature.sigAlg != TPM2_ALG_RSASSA) {
    LOG_ERR("Only RSASSA is supported for signatures");
    goto err;
}
TPM2B_PUBLIC_KEY_RSA sig = ctx.signature.signature.rsassa.sig;
tpm2_tool_output("sig: ");
tpm2_util_hexdump(sig.buffer, sig.size);
tpm2_tool_output("\n");

// Verify the signature matches message digest
int openssl_hash =
    tpm2_openssl_halgid_from_tpmhalg(ctx.signature.signature.rsassa.hash);
if (!RSA_verify(openssl_hash, ctx.msg_hash.buffer, ctx.msg_hash.size,
        sig.buffer, sig.size, pub_key)) {
    LOG_ERR("Error validating signed message with public key provided");
    goto err;
}
```

tpm2_quote命令输出的签名默认是TSS格式的，TSS格式里嵌入了hash算法的信息，所以不能被其它工具识别，
可以指定命令输出plain格式的签名：
```
$ tpm2_quote -c 0x8101000a -l sha256:15,16,22 -q $nonce \
> -m quote.out -f plain -s quotesig.out -o quotepcr.out -g sha256 -p "akpass"
```

plain格式不含算法信息，checkquote也需要指定-F rsassa参数，tpm2-tools内部始终使用TSS格式。
```
$ tpm2_checkquote -u akpub.pem -m quote.out -F rsassa -s quotesig.out \
> -f quotepcr.out -g sha256 -q $nonce
```

有了rsassa格式的签名，可以用openssl命令来检查：
```
[kaifeng@localhost-live ~]$ openssl dgst -sha256 -verify akpub.pem \
> -signature quotesig.out quote.out
Verified OK
[kaifeng@localhost-live ~]$
```

### 检查nonce

nonce值存放在TPMS_ATTEST的extraData字段，没有额外的处理，校验方收到Quote数据后，
与此前发送给证明方的nonce值进行对比，如果相等，表明这是验证方针对校验方的请求作出的回应。
```
// Ensure nonce is the same as given
if (ctx.flags.extra) {
    if (ctx.quote_extra_data.size != ctx.extra_data.size ||
        memcmp(ctx.quote_extra_data.buffer, ctx.extra_data.buffer,
        ctx.extra_data.size) != 0) {
        LOG_ERR("Error validating nonce from quote");
        goto err;
    }
}
```

一般不建议在nonce中携带用户数据，如果需要携带用户数据，可以使用可复位的PCR，在每次证明时进行单次度量。
nonce可以有效的阻止replay攻击，但需要保证足够的随机性。

### 检查PCR

这一步是可选的，因为Quote数据中包含了所选PCR的摘要，如果提供了PCR的原始值，则再计算一次摘要看是否相同。
```
// Also ensure digest from quote matches PCR digest
if (ctx.flags.pcr) {
    if (!tpm2_util_verify_digests(&ctx.quote_hash, &ctx.pcr_hash)) {
        LOG_ERR("Error validating PCR composite against signed message");
        goto err;
    }
}
```

PCR原始值并不在Quote数据结构中，需要以其它方式附加在证明数据里。
既然已经有PCR摘要了，是否需要PCR原始值？建议在证据中提供，因为PCR的值不可预测，
如果出现度量值不匹配，定位非常困难，因为PCR不同寄存器可以反映不同阶段的度量数据，
提供原始PCR数据可以帮助缩小排查范围。
另外从TAP协议来看，PCR原始值应该也需要提供。

## 可信仲裁

上述所有的检查都只是证明了证据是合法的，校验方需要有一个预期的状态来对度量值进行对比，
所以事先必须要设置好一个白名单。对于云主机来说，如果硬件型号和软件版本相同，基本上也可以得到相同的度量结果，
这些结果可以预先存放在校验方的数据库中。如果硬件型号更换，固件/软件升级，目前看到的都是采用预先度量的方式记录预期结果并加入白名单，
如何增强则依赖于产品的实现。

如果厂商能为自己的各类固件代码提供软件摘要并提供给校验方，那么校验方也可以在本地基于这些数据（固件、操作系统版本、软件版本等）
构造出整个信用链的计算过程，从而得到最终状态作为比对参考，对于简单的系统也是一种可行的方式。

# 验证TPM的合法性

如果厂商提供了IAK及证书，那么只要校验方存有IAK的证书便能检查证明数据是否来自一个真实的TPM。
因为证明方使用IAK对证据进行签名，而校验方可以使用IAK证书内的公钥进行校验。

有的厂商并没有录入IAK，远程证明过程就需要增加一次交互，此时的流程大致如下：

1. 校验方向证明方请求远程证明
2. 证明方通过TPM生成AK，将AK和EK公钥发送给校验方
3. 校验方使用EK公钥加密一段隐私数据（随便什么都可以），发给证明方
4. 证明方使用TPM解密这段数据，如果成功，收集证据（即Quote），然后发给校验方
5. 校验方检查AK签名和证明证据，得出可信结论。

需要注意的是，证明方与验证方通信的服务（或称为代理程序，在opencit中称为trustagent），必须要加入主机的本地可信链，
这样才可确保其行为是符合预期的，否则证明方的流程实现就有可能被篡改。加入可信链后，任何篡改将导致度量值变化，无法通过验证。
因为签名只能在TPM操作，而提取PCR数据并签名是一个原子操作。

实际上，隐私数据的内容并不是关键，它只是利用了不对称加密的特性，使证明方必须通过TPM解密的方法来证明自己对TPM的所有权。
但是在流程上可以利用这一点，向证明方发送一些证据，并在随后的校验中使用，比如加密nonce，或者加密一个证书。

## makecredential与activatecredential

tpm2-tools有两个工具可以验证隐私数据的传递：makecredential使用某个TPM公钥加密一段数据，
activatecredential使用TPM内对应的私钥进行解密，
它们分别实现了规范定义的TPM2_MakeCredential和TPM2_ActivateCredential命令。
在远程证明场景下，这里的密钥即AK。

证明方创建AK并输出AK的公钥，同时也读取TPM内的EK公钥并输出：
```
[kaifeng@localhost-live att-test]$ handle_ek=0x81010009
[kaifeng@localhost-live att-test]$ tpm2_createek -c $handle_ek -G rsa \
> -u ekpub.pem -f pem
[kaifeng@localhost-live att-test]$ tpm2_readpublic -c $handle_ek \
> -o ek.pub
name: 000bbcb036b11c7828e7a4b39764686cf9b67a45b33fcb9d5af4adac3f70e3
cc9198
qualified name: 000b5f06c80b381f35e5172c8e3958618b4e759b43f11c16e092
1ebfd759b36eeb0a
name-alg:
  value: sha256
  raw: 0xb
attributes:
  value: fixedtpm|fixedparent|sensitivedataorigin|adminwithpolicy|
restricted|decrypt
  raw: 0x300b2
type:
  value: rsa
  raw: 0x1
exponent: 0x0
bits: 2048
scheme:
  value: null
  raw: 0x10
scheme-halg:
  value: (null)
  raw: 0x0
sym-alg:
  value: aes
  raw: 0x6
sym-mode:
  value: cfb
  raw: 0x43
sym-keybits: 128
rsa: d64fda2c00008b97b04ca2d079bb4c4fa7ae71f95b40727e431ee99a97c60dd
7815ac351e67260bbb22c1239c1f63d0129b22a04bd31c4f85054db80e050b51e608
c1b98f1ee98c7fa58957480a6e9e8692353fb6cc67f0f96d62e814f5c707c80e7410
a50814d0b7b6d9260966823bd2db29c4dd95a604f8a020079dbd1721956f05d4171b
5e5040e5bb2f379c117178e9b10106842f2738f2969234cc5b8c377094fb3d06b43c
7ecd47fb77592d015edefd26116e1d7ff0516089230dfc3b51a4f1c3b74b53a90213
24e7673dc56281178fc5dd00e8d22bfecdcaf7bc5e3dc8eb19dc86599cb9aa890ce4
a8aa8fd8f6a4cc49c53a55004953b7fb65132dce3
authorization policy: 837197674484b3f81a90cc8d46a5d724fd52d76e06520b
64f2a1da1b331469aa
[kaifeng@localhost-live att-test]$ tpm2_readpublic -c ak.ctx \
> -o ak.pub
name: 000be96ded8585c0b60feffa13228f18fb07e9ba9d755942d52d7195f0fc89
462846
qualified name: 000bbad02ab43cc5ff1c1ac0a51aff90ac4dcea5cdc35f9f0530
1805345b14f50dc4
name-alg:
  value: sha256
  raw: 0xb
attributes:
  value: fixedtpm|fixedparent|sensitivedataorigin|userwithauth|
restricted|sign
  raw: 0x50072
type:
  value: rsa
  raw: 0x1
exponent: 0x0
bits: 2048
scheme:
  value: rsassa
  raw: 0x14
scheme-halg:
  value: sha256
  raw: 0xb
sym-alg:
  value: null
  raw: 0x10
sym-mode:
  value: (null)
  raw: 0x0
sym-keybits: 0
rsa: cbd4736f7e0784b13ae30f5a944388b87382b8482e4272fc2b49f1791516b84
5b37afc4506e4c559f8266d38a5974d49252203419944bd578f8c5db9eaf4d95beb6
434681e94fa365d396da5040886492e967500611d3e18248f8b237599144d4d887d3
3600cb4b4e4f11fd1b5ca11d0735cc9405f22d0a960e34a951043b8ea1249de7862c
6e8937a2ad9fafc538103b3e8655151410bcd79200b040ed3f47580a810d99be8cf7
93f7b2a59d95bb88beee3b79be6be1759750685fbd0e16a52554e5f07a880c18c518
d9331a18512b4bb9bc54bbac143f3fb3c1eba0e528f1aa823fa5698ebe60fa1a9e85
19447b7146896bf69496b55f0c99a032f3e8a72ff
[kaifeng@localhost-live att-test]$
```

证明方将EK和AK公钥发给校验方，检验方通过makecredential加密隐私数据，得到一段加密的数据：
```
[root@b7e499f82705 ~]# echo 12345678 > secret.data
[root@b7e499f82705 ~]# ls
ak.name  ek.pub  secret.data
[root@b7e499f82705 ~]# echo $loaded_key_name
000be96ded8585c0b60feffa13228f18fb07e9ba9d755942d52d7195f0fc89462846
[root@b7e499f82705 ~]# tpm2_makecredential -T none -e ek.pub \
> -s secret.data -n $loaded_key_name -o mkcred.out
```

验证方将加密数据actcred.out发送给证明方，证明方使用TPM解密数据。具体地说，EK公钥保护的是一个种子，
经TPM解密后，与AK的Key名一起计算出一个对称密钥（KDF），再用该对称密钥解密隐私数据。
证明方若能成功完成这一步，即可证明它对TPM的所有权（Proof of ownership）。

```
[kaifeng@localhost-live att-test]$ TPM2_RH_ENDORSEMENT=0x4000000B
[kaifeng@localhost-live att-test]$ tpm2_startauthsession --policy-session -S session.ctx
[kaifeng@localhost-live att-test]$ tpm2_policysecret -S session.ctx -c $TPM2_RH_ENDORSEMENT
837197674484b3f81a90cc8d46a5d724fd52d76e06520b64f2a1da1b331469aa
[kaifeng@localhost-live att-test]$ tpm2_activatecredential -c ak.ctx -C $handle_ek -i mkcred.out -o actcred.out -P "session:session.ctx"
certinfodata:31323334353637380a
[kaifeng@localhost-live att-test]$ tpm2_flushcontext session.ctx
[kaifeng@localhost-live att-test]$ cat actcred.out
12345678
[kaifeng@localhost-live att-test]$
```

这里的session是指会话，使用同一会话的命令会处于相同的执行环境和状态，TPM会提供一些安全性保证，
有些类似数据库的事务，针对这一特性目前还没有过多研究。

## Privacy CA

如果TPM出厂没有IAK，或者不希望使用IAK进行验证，那么在设备投入使用时需要通过EK生成AK，并且需要一个Privacy CA的角色。
Privacy CA也称为Attestation CA，通常负责签发AK证书和校验。
签发AK证书看起来更像是设备投入使用前执行的一种注册性工作，
但是放在远程证明流程中，看起来也没有技术上的限制，只不过每次证明时需要先生成一个AK密钥，多一些交互。

加入PCA后的流程如下：

1. 验证方生成AK密钥，并向PCA提供EK证书和AK公钥信息
2. PCA签发一个AK证书并生成一个对称密钥，用makecredential的方式加密AK证书，然后发给验证方
3. 验证方解密证书，可以保存在NVRAM用作后续远程证明流程的身份证明
4. 验证方将Quote数据连同AK证书发送给校验方，因为AK证书由PCA签发，因此校验方也可以在本地保存PCA的根证书或者向PCA验证证书。

> 原型实现：https://github.com/blaufish/Tpm2AttestationCertificationAuthority

```
+-----+ ------ ek_rsa.cert --->  +--------------+
|     | ------ ak_rsa.pub ---->  | Attestation  |
| T P | ------ ak_rsa.name --->  | Certificate  |
| R L |                          |  Authority   |
| U A | <---- credentials -----  |              |
| S T | <---- ak_cert.enc -----  +--------------+
| T F |
| E O | ( Decrypt AKCert using TPM.EKpriv / TPM2_ActivateCredential )
| D R |
|   M |
| M   |                           +--------------+
| O   | -------- ak_cert ------>  |    Remote    |
| D   | ------ tpm2_quote ----->  | Attestation  |
| U   |                           |    Server    |
| L   |                           +--------------+
| E   |
+-----+
```

依实现场景，PCA与外部CA可以交互，也可以不交互。对于私有云，不交互的情况应该居多，PCA需要事先保存好TPM厂商的根证书。
PCA也可以与校验方结合，实现上是很灵活的。

## 获取TPM证书

### 从NVRAM读取EK证书

TPM设备理应提供EK证书，取决于厂商情况，如果出厂没有EK，那就需要可信的流程来完成设备注册和EK签发，否则就没有了可信的根源。
在没有硬件情况下，可以使用软件模拟，swtpm模拟了这个功能。

swtpm启动时可以指定创建证书，EK证书存放在0x01c00002：
```
[kaifeng@localhost-live att-test]$ tpm2_nvread 0x1c00002 > ek.cert
[kaifeng@localhost-live ~]$ openssl x509 -inform der -in ek.cert -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 13 (0xd)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = swtpm-localca
        Validity
            Not Before: Mar 10 05:48:13 2020 GMT
            Not After : Mar  8 05:48:13 2030 GMT
        Subject: CN = fedora31:fdbb98a4-44c4-4ee3-a390-b2465e3e866a
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Modulus:
                    00:d6:4f:da:2c:00:00:8b:97:b0:4c:a2:d0:79:bb:
                    4c:4f:a7:ae:71:f9:5b:40:72:7e:43:1e:e9:9a:97:
                    c6:0d:d7:81:5a:c3:51:e6:72:60:bb:b2:2c:12:39:
                    c1:f6:3d:01:29:b2:2a:04:bd:31:c4:f8:50:54:db:
                    80:e0:50:b5:1e:60:8c:1b:98:f1:ee:98:c7:fa:58:
                    95:74:80:a6:e9:e8:69:23:53:fb:6c:c6:7f:0f:96:
                    d6:2e:81:4f:5c:70:7c:80:e7:41:0a:50:81:4d:0b:
                    7b:6d:92:60:96:68:23:bd:2d:b2:9c:4d:d9:5a:60:
                    4f:8a:02:00:79:db:d1:72:19:56:f0:5d:41:71:b5:
                    e5:04:0e:5b:b2:f3:79:c1:17:17:8e:9b:10:10:68:
                    42:f2:73:8f:29:69:23:4c:c5:b8:c3:77:09:4f:b3:
                    d0:6b:43:c7:ec:d4:7f:b7:75:92:d0:15:ed:ef:d2:
                    61:16:e1:d7:ff:05:16:08:92:30:df:c3:b5:1a:4f:
                    1c:3b:74:b5:3a:90:21:32:4e:76:73:dc:56:28:11:
                    78:fc:5d:d0:0e:8d:22:bf:ec:dc:af:7b:c5:e3:dc:
                    8e:b1:9d:c8:65:99:cb:9a:a8:90:ce:4a:8a:a8:fd:
                    8f:6a:4c:c4:9c:53:a5:50:04:95:3b:7f:b6:51:32:
                    dc:e3
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Extended Key Usage:
                2.23.133.8.1
            X509v3 Subject Alternative Name: critical
                DirName:/2.23.133.2.1=id:00001014/2.23.133.2.2=swtpm/2.23.133.2.3=id:20170619
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Subject Directory Attributes:
                0.0...g....1.0...2.0.......
            X509v3 Authority Key Identifier:
                keyid:0C:D1:0A:71:79:4B:C4:75:EF:02:EB:0A:3D:89:06:96:D8:DE:3F:CC

            X509v3 Key Usage: critical
                Key Encipherment
    Signature Algorithm: sha256WithRSAEncryption
         0b:8a:7c:57:ba:f2:ed:80:9a:64:9b:62:8a:d7:58:f7:b1:67:
         47:6c:da:fa:4d:d0:40:ff:6d:90:50:23:2b:ba:25:41:30:8c:
         67:c2:4a:fe:16:99:c7:95:f2:0b:5d:92:ce:70:58:56:8a:e4:
         e9:03:ce:a5:60:1d:00:aa:17:74:6f:32:8e:d8:cd:e1:58:c9:
         df:c4:85:04:25:08:c3:36:33:35:96:96:e3:9a:77:67:63:5a:
         e0:f2:63:2e:ef:cf:65:5c:ae:84:7f:1d:e8:ca:7f:0d:dc:74:
         42:51:98:a1:8b:b7:c9:0d:7d:09:30:3e:a8:75:7b:a8:08:3e:
         5a:fe:6b:fb:ac:b1:82:96:d6:ff:c4:1c:48:ef:49:90:99:a5:
         9d:b5:ca:6f:ae:04:83:d4:48:46:c3:73:98:50:fc:a4:42:a9:
         34:9a:c9:de:ce:d0:81:ad:bf:13:52:87:9b:6a:07:e8:c5:7d:
         d6:f9:90:3c:72:16:3b:25:dd:cc:c5:f2:9e:c1:3c:e9:e4:b1:
         ea:8b:2d:9e:8c:94:39:7e:9d:1e:3c:b4:b1:30:c1:69:fd:10:
         fb:06:41:7d:35:b0:77:b4:b8:ea:e1:1d:01:e1:96:94:5a:1d:
         27:47:42:d3:da:7d:b4:68:8a:8f:36:85:92:77:30:1c:86:6c:
         63:c2:11:ac:cd:5e:e5:e9:cd:4b:24:b9:d0:11:c4:46:5b:da:
         5a:2e:97:f2:6f:57:bd:19:8d:4b:7d:48:bb:9c:5a:61:0e:b2:
         84:40:da:b9:ac:c3:e3:cb:28:93:49:db:38:7f:07:62:92:47:
         59:9e:96:0d:5a:40:6a:d3:a6:f6:25:d7:45:fd:bc:4b:c9:a3:
         52:08:36:26:f8:58:c7:18:f1:34:ec:f6:78:0b:e3:9e:6b:d0:
         86:4c:ba:85:e2:28:c7:e3:ce:ff:1e:42:20:79:83:22:ba:4a:
         16:82:63:f4:8b:bb:9b:92:9a:b4:a0:c0:62:11:29:72:e1:68:
         87:6d:19:47:0e:79
-----BEGIN CERTIFICATE-----
MIIEGTCCAoGgAwIBAgIBDTANBgkqhkiG9w0BAQsFADAYMRYwFAYDVQQDEw1zd3Rw
bS1sb2NhbGNhMB4XDTIwMDMxMDA1NDgxM1oXDTMwMDMwODA1NDgxM1owODE2MDQG
A1UEAxMtZmVkb3JhMzE6ZmRiYjk4YTQtNDRjNC00ZWUzLWEzOTAtYjI0NjVlM2U4
NjZhMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA1k/aLAAAi5ewTKLQ
ebtMT6eucflbQHJ+Qx7pmpfGDdeBWsNR5nJgu7IsEjnB9j0BKbIqBL0xxPhQVNuA
4FC1HmCMG5jx7pjH+liVdICm6ehpI1P7bMZ/D5bWLoFPXHB8gOdBClCBTQt7bZJg
lmgjvS2ynE3ZWmBPigIAedvRchlW8F1BcbXlBA5bsvN5wRcXjpsQEGhC8nOPKWkj
TMW4w3cJT7PQa0PH7NR/t3WS0BXt79JhFuHX/wUWCJIw38O1Gk8cO3S1OpAhMk52
c9xWKBF4/F3QDo0iv+zcr3vF49yOsZ3IZZnLmqiQzkqKqP2PakzEnFOlUASVO3+2
UTLc4wIDAQABo4HNMIHKMBAGA1UdJQQJMAcGBWeBBQgBMFIGA1UdEQEB/wRIMEak
RDBCMRYwFAYFZ4EFAgEMC2lkOjAwMDAxMDE0MRAwDgYFZ4EFAgIMBXN3dHBtMRYw
FAYFZ4EFAgMMC2lkOjIwMTcwNjE5MAwGA1UdEwEB/wQCMAAwIgYDVR0JBBswGTAX
BgVngQUCEDEOMAwMAzIuMAIBAAICAJYwHwYDVR0jBBgwFoAUDNEKcXlLxHXvAusK
PYkGltjeP8wwDwYDVR0PAQH/BAUDAwcgADANBgkqhkiG9w0BAQsFAAOCAYEAC4p8
V7ry7YCaZJtiitdY97FnR2za+k3QQP9tkFAjK7olQTCMZ8JK/haZx5XyC12SznBY
Vork6QPOpWAdAKoXdG8yjtjN4VjJ38SFBCUIwzYzNZaW45p3Z2Na4PJjLu/PZVyu
hH8d6Mp/Ddx0QlGYoYu3yQ19CTA+qHV7qAg+Wv5r+6yxgpbW/8QcSO9JkJmlnbXK
b64Eg9RIRsNzmFD8pEKpNJrJ3s7Qga2/E1KHm2oH6MV91vmQPHIWOyXdzMXynsE8
6eSx6ostnoyUOX6dHjy0sTDBaf0Q+wZBfTWwd7S46uEdAeGWlFodJ0dC09p9tGiK
jzaFkncwHIZsY8IRrM1e5enNSyS50BHERlvaWi6X8m9XvRmNS31Iu5xaYQ6yhEDa
uazD48sok0nbOH8HYpJHWZ6WDVpAatOm9iXXRf28S8mjUgg2JvhYxxjxNOz2eAvj
nmvQhky6heIox+PO/x5CIHmDIrpKFoJj9Iu7m5KatKDAYhEpcuFoh20ZRw55
-----END CERTIFICATE-----
[kaifeng@localhost-live ~]$
```

### 从厂商网站下载TPM根证书

Infineon是TPM的制造商之一，它的证书可以从网站下载，例如根证书：

https://www.infineon.com/dgdl/Infineon-TPM_RSA_Root_CA-C-v01_00-EN.cer?fileId=5546d46253f6505701540496a5641d20

从下面的输出可以看出，其实就是一个厂商自签名的证书，在构造可信云平台时，要先将这些厂商证书放在PCA或者校验方。
在远程证明过程中，证明方将EK证书提供给校验方，校验方从EK证书取得签名厂商，再使用对应厂商的证书进行证书验证。

```
kaifeng@morph:~/TPM$ openssl x509 -inform der -in Infineon-TPM_RSA_Root_CA-C-v01_00-EN.cer -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 3 (0x3)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = DE, O = Infineon Technologies AG, OU = OPTIGA(TM) Devices, CN = Infineon OPTIGA(TM) RSA Root CA
        Validity
            Not Before: Jul 26 00:00:00 2013 GMT
            Not After : Jul 25 23:59:59 2043 GMT
        Subject: C = DE, O = Infineon Technologies AG, OU = OPTIGA(TM) Devices, CN = Infineon OPTIGA(TM) RSA Root CA
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (4096 bit)
                Modulus:
                    00:bb:13:e8:1c:d0:1e:53:ed:ac:33:bb:1e:ba:cc:
                    c3:19:31:3b:42:90:fa:86:bf:a6:b7:35:5c:7b:dc:
                    80:a0:d8:34:b0:9e:2a:45:c0:18:94:97:db:28:12:
                    87:67:db:94:95:54:de:e9:aa:6b:ca:03:68:0c:4d:
                    1e:50:7b:1b:98:4b:d3:cf:7a:b7:d1:66:b0:58:d3:
                    4c:72:17:1f:38:57:e9:88:44:f8:38:62:a5:0c:2e:
                    d6:41:70:0c:2b:47:71:8e:e9:c7:df:1e:9d:b4:48:
                    11:f9:21:3a:6e:a8:a7:22:c1:bd:2f:e6:e7:8b:76:
                    d6:47:61:13:ea:d7:e8:e0:cb:9f:08:15:8f:ef:00:
                    2c:85:fd:c7:16:67:15:12:25:13:52:7b:8a:ee:c0:
                    18:08:ec:d3:16:89:cc:62:89:64:26:cf:57:c4:dd:
                    ed:26:50:64:35:b6:ee:0c:e8:ca:59:3f:14:d9:c5:
                    6c:cd:d2:63:33:f6:7a:23:d6:82:13:65:49:ea:fd:
                    da:cd:e7:82:c7:cd:7e:39:97:ed:9b:d7:87:f9:16:
                    4b:ed:71:7c:49:ec:e0:a4:23:b9:66:58:8b:7c:b3:
                    97:c4:e0:78:62:c4:48:2c:47:64:57:e6:1c:e5:f1:
                    78:87:89:2e:ee:0f:7a:50:84:16:12:04:de:48:05:
                    b5:56:44:47:b1:d4:85:1a:b7:97:80:39:bf:40:5e:
                    39:d9:ee:2b:f1:24:a8:98:fc:19:0e:9a:b3:60:37:
                    c9:36:ee:f3:92:e0:ff:35:8b:1d:46:9d:7b:23:c8:
                    72:7a:98:eb:56:44:2f:54:1d:fb:c9:72:f3:37:53:
                    db:6e:53:ed:dd:45:f8:9b:d3:73:46:c5:23:e7:2a:
                    d7:8b:e1:23:f5:6d:d1:df:88:68:d5:dc:b2:31:cc:
                    51:ce:7d:d8:cc:d9:cb:c5:27:a8:d7:83:98:70:5c:
                    21:52:76:c4:26:e5:ed:81:7d:3d:dd:58:30:52:7d:
                    1e:21:dc:fa:e9:92:5e:9d:70:0c:9b:de:73:6d:30:
                    ad:c7:47:9c:a5:e9:00:6e:27:26:f0:f1:a7:c7:4f:
                    72:91:6f:0b:ce:1c:e0:91:d1:95:49:4e:cc:dc:94:
                    43:dc:33:73:50:77:01:65:86:a2:d2:82:12:1f:95:
                    a2:92:3e:ff:72:1d:32:9e:83:60:01:e6:af:49:48:
                    6d:a7:c1:24:eb:8c:32:91:69:bd:b6:e7:ec:c9:d3:
                    2c:a5:1f:93:70:c4:80:4b:69:51:c3:c8:01:2e:f3:
                    56:9f:fb:09:e6:f7:da:2a:82:f2:6e:0c:a4:90:1b:
                    22:df:ea:c0:9e:bb:2f:35:c1:06:0e:e3:ac:8d:6b:
                    ec:49:c1
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                DC:BB:56:AB:F1:18:FC:A6:9A:75:11:10:65:84:12:9E:D5:41:92:B9
            X509v3 Key Usage: critical
                Certificate Sign, CRL Sign
            X509v3 Basic Constraints: critical
                CA:TRUE
    Signature Algorithm: sha256WithRSAEncryption
         61:d3:05:4c:77:11:32:17:60:9b:1a:02:06:f6:a7:2c:8d:52:
         5b:55:2f:66:ca:cc:63:15:4a:c9:d3:0a:b5:d4:53:c8:4a:ac:
         34:21:de:33:48:32:b4:b4:77:a7:54:fd:f1:9a:18:9b:de:87:
         19:a6:25:f8:da:37:f2:05:58:0e:0c:05:d6:44:9e:90:1e:d9:
         f2:44:3f:cb:db:2d:af:d0:1d:57:ec:01:5b:a8:b4:b0:fa:41:
         60:2a:79:a0:b6:b7:1a:71:91:ea:ef:7a:85:76:24:0c:23:d6:
         3a:e9:ad:3e:32:ae:44:bc:4b:16:6e:29:2d:39:ad:87:65:1b:
         7d:06:8c:e2:93:ab:cd:5a:99:0b:46:7f:63:65:36:23:34:94:
         10:7f:48:a4:ed:d1:4b:f9:4c:ad:23:21:66:f3:d9:6a:19:80:
         6b:ad:ee:61:74:99:dd:6b:1d:16:ae:59:17:28:1d:07:71:59:
         24:13:09:3b:60:a1:0c:de:06:3e:08:4b:a3:77:43:14:01:29:
         5f:4b:2f:d5:16:9b:e0:8c:21:3e:d8:9e:9f:1b:48:9f:28:5b:
         85:1d:a5:2b:98:59:ed:59:c1:28:d9:cb:30:e2:4b:3a:6e:73:
         88:13:4b:a6:87:7c:c9:84:7e:85:70:0b:c1:cd:d3:7f:c7:0f:
         a0:75:2f:0a:36:0e:f1:1a:a8:0c:64:c9:af:48:61:6a:c3:aa:
         43:2b:10:4a:c9:3b:a7:2a:c7:fb:df:21:c7:d8:ff:c2:58:99:
         42:46:4e:6b:d5:95:a3:83:25:3c:18:c2:8b:ff:da:b8:eb:95:
         ae:60:47:ec:6b:4c:ce:dd:ab:c6:d9:8a:64:90:4c:ef:21:e9:
         c7:4b:ea:ef:b7:73:c9:86:ae:20:90:90:d6:48:64:46:9c:fc:
         67:67:f8:ac:32:75:a3:99:21:c9:df:45:da:77:49:a9:71:b1:
         2a:7a:71:6c:09:18:4e:30:a5:82:81:d8:29:4c:d9:01:77:c8:
         d4:28:4c:63:70:32:5a:6c:c6:75:3b:da:28:43:8b:f4:71:c9:
         a4:53:cf:d3:91:a6:e6:cd:ab:c5:ae:ab:b8:d0:52:ce:54:d3:
         4a:f2:ac:c0:99:a2:0d:5c:c4:bc:f2:7d:9c:30:09:88:7a:3b:
         60:a6:6e:f3:28:c9:d1:cd:cb:90:94:df:28:31:68:b5:af:26:
         98:19:54:73:75:d3:79:e9:54:c4:77:98:b3:5a:dd:01:3e:e4:
         c1:65:06:53:f7:32:6c:ac:b6:ef:22:54:02:89:6a:cd:fd:54:
         ee:72:e6:36:9d:e1:d4:4b:93:0b:4c:ea:c2:5a:f2:20:69:75:
         18:d9:ac:1a:79:b0:90:48
-----BEGIN CERTIFICATE-----
MIIFqzCCA5OgAwIBAgIBAzANBgkqhkiG9w0BAQsFADB3MQswCQYDVQQGEwJERTEh
MB8GA1UECgwYSW5maW5lb24gVGVjaG5vbG9naWVzIEFHMRswGQYDVQQLDBJPUFRJ
R0EoVE0pIERldmljZXMxKDAmBgNVBAMMH0luZmluZW9uIE9QVElHQShUTSkgUlNB
IFJvb3QgQ0EwHhcNMTMwNzI2MDAwMDAwWhcNNDMwNzI1MjM1OTU5WjB3MQswCQYD
VQQGEwJERTEhMB8GA1UECgwYSW5maW5lb24gVGVjaG5vbG9naWVzIEFHMRswGQYD
VQQLDBJPUFRJR0EoVE0pIERldmljZXMxKDAmBgNVBAMMH0luZmluZW9uIE9QVElH
QShUTSkgUlNBIFJvb3QgQ0EwggIiMA0GCSqGSIb3DQEBAQUAA4ICDwAwggIKAoIC
AQC7E+gc0B5T7awzux66zMMZMTtCkPqGv6a3NVx73ICg2DSwnipFwBiUl9soEodn
25SVVN7pqmvKA2gMTR5QexuYS9PPerfRZrBY00xyFx84V+mIRPg4YqUMLtZBcAwr
R3GO6cffHp20SBH5ITpuqKciwb0v5ueLdtZHYRPq1+jgy58IFY/vACyF/ccWZxUS
JRNSe4ruwBgI7NMWicxiiWQmz1fE3e0mUGQ1tu4M6MpZPxTZxWzN0mMz9noj1oIT
ZUnq/drN54LHzX45l+2b14f5FkvtcXxJ7OCkI7lmWIt8s5fE4HhixEgsR2RX5hzl
8XiHiS7uD3pQhBYSBN5IBbVWREex1IUat5eAOb9AXjnZ7ivxJKiY/BkOmrNgN8k2
7vOS4P81ix1GnXsjyHJ6mOtWRC9UHfvJcvM3U9tuU+3dRfib03NGxSPnKteL4SP1
bdHfiGjV3LIxzFHOfdjM2cvFJ6jXg5hwXCFSdsQm5e2BfT3dWDBSfR4h3Prpkl6d
cAyb3nNtMK3HR5yl6QBuJybw8afHT3KRbwvOHOCR0ZVJTszclEPcM3NQdwFlhqLS
ghIflaKSPv9yHTKeg2AB5q9JSG2nwSTrjDKRab225+zJ0yylH5NwxIBLaVHDyAEu
81af+wnm99oqgvJuDKSQGyLf6sCeuy81wQYO46yNa+xJwQIDAQABo0IwQDAdBgNV
HQ4EFgQU3LtWq/EY/KaadREQZYQSntVBkrkwDgYDVR0PAQH/BAQDAgAGMA8GA1Ud
EwEB/wQFMAMBAf8wDQYJKoZIhvcNAQELBQADggIBAGHTBUx3ETIXYJsaAgb2pyyN
UltVL2bKzGMVSsnTCrXUU8hKrDQh3jNIMrS0d6dU/fGaGJvehxmmJfjaN/IFWA4M
BdZEnpAe2fJEP8vbLa/QHVfsAVuotLD6QWAqeaC2txpxkerveoV2JAwj1jrprT4y
rkS8SxZuKS05rYdlG30GjOKTq81amQtGf2NlNiM0lBB/SKTt0Uv5TK0jIWbz2WoZ
gGut7mF0md1rHRauWRcoHQdxWSQTCTtgoQzeBj4IS6N3QxQBKV9LL9UWm+CMIT7Y
np8bSJ8oW4UdpSuYWe1ZwSjZyzDiSzpuc4gTS6aHfMmEfoVwC8HN03/HD6B1Lwo2
DvEaqAxkya9IYWrDqkMrEErJO6cqx/vfIcfY/8JYmUJGTmvVlaODJTwYwov/2rjr
la5gR+xrTM7dq8bZimSQTO8h6cdL6u+3c8mGriCQkNZIZEac/Gdn+KwydaOZIcnf
Rdp3SalxsSp6cWwJGE4wpYKB2ClM2QF3yNQoTGNwMlpsxnU72ihDi/RxyaRTz9OR
pubNq8Wuq7jQUs5U00ryrMCZog1cxLzyfZwwCYh6O2CmbvMoydHNy5CU3ygxaLWv
JpgZVHN103npVMR3mLNa3QE+5MFlBlP3Mmystu8iVAKJas39VO5y5jad4dRLkwtM
6sJa8iBpdRjZrBp5sJBI
-----END CERTIFICATE-----
kaifeng@morph:~/TPM$
```

## 证书校验

真正的证书校验需要有真实的硬件，但这个过程是一样的，下面来模拟校验方怎么校验EK证书。

### 一个假的TPM厂商

先生成密钥：
```
[kaifeng@localhost-live ca]$ openssl genrsa -aes128 -out vendor.key 2048
Generating RSA private key, 2048 bit long modulus (2 primes)
.........+++++
....................................................................
....................................................................
.................+++++
e is 65537 (0x010001)
Enter pass phrase for vendor.key:
Verifying - Enter pass phrase for vendor.key:
[kaifeng@localhost-live ca]$
```

生成自签名证书：
```
[kaifeng@localhost-live ca]$ openssl req -new -key vendor.key -out vendor.csr
```

查看自签名证书：
```
[kaifeng@localhost-live ca]$ openssl x509 -in vendor.crt -text
```

TPM出厂也录入了EK密钥，这个密钥也理应由厂商生成并且写入TPM芯片，所以再生成一个密钥用于EK：
```
[kaifeng@localhost-live ca]$ openssl genrsa -aes128 -out ek.key 2048
Generating RSA private key, 2048 bit long modulus (2 primes)
........................+++++
..............+++++
e is 65537 (0x010001)
Enter pass phrase for ek.key:
Verifying - Enter pass phrase for ek.key:
[kaifeng@localhost-live ca]$
```

为EK生成CSR，这里厂商可以填些有用的信息：
```
[kaifeng@localhost-live ca]$ openssl req -new -sha256 -key ek.key -out ek.csr
```

为EK生成证书：
```
[kaifeng@localhost-live ca]$ openssl x509 -req -in ek.csr -CA vendor.crt -CAkey vendor.key -CAcreateserial -out ek.crt -days 365 -sha256
Signature ok
subject=C = CN, ST = ***, L = ***, O = ***, OU = ***, CN = TPM001
Getting CA Private Key
Enter pass phrase for vendor.key:
[kaifeng@localhost-live ca]$
```

### 校验方验证证书

校验方拿到EK证书后，可以查看证书的信息，找到签发方，也就是Issuer，从证书信息知道这个证书是由X厂商签发的。

校验方从本地数据库中找到X厂商的根证书对EK证书进行校验：
```
[kaifeng@localhost-live ca]$ openssl verify -CAfile vendor.crt ek.crt
ek.crt: OK
[kaifeng@localhost-live ca]$
```

AK证书的验证与EK一样，区别仅在于AK证书可能由PCA签发，此时就需要PCA的证书。

# 可信证明协议TAP

TAP定义的主要是证明活动中涉及的数据定义，确保包含了需要的信息，校验方可以基于PCR的校验来确定一个系统的完整性。
所以TAP不定义信息传递的具体方法或协议（实现无关），它可以用于诸如VPN，SSH或其它能够访问网络的协议，
这些由其它的绑定协议定义，绑定协议定义哪些信息是必需的。
TAP也没有指定如何执行完整性检查，它只关心是否能收集完备的信息，并且以合适的方式传递。

TAP是对PTS1.0的重新设计，不向后兼容。TAP增加了对DICE（
Device Identifier Composition Engine，这是另一套系统）和TPM2.0的支持，
包含了一些旧TPM版本不支持的新特性。
因为TPM1.2已经在外面使用，很多环境在一段时间会有TPM1.2和TPM2.0混用的情况，
所以TAP也定义了与TPM1.2的交互，但TPM1.2不支持所有的TAP定义。

各类信息元素构成证明报告，规范使用TLV表示法：TYPE || Length || (Value)，Type占1个字节，Length一般占4个字节，Value是各类参数的组合。TLV定义只是用于描述定义，与具体实现无关。

TAP是2019年9月制定的规范，鉴于社区现状，目前可能没有可以参考的开源实现。类型的定义如下。

```
0x00 | TAP规范版本（2字节）
0x01 | AK证书链
0x02 | TPM 2.0签名的证明数据（用于隐式证明）
0x03 | PCR值（用于TPM1.2）
0x04 | PCR值（用于TPM2.0）
0x05 | PCR日志值
0x06 | 时效性（即nonce）
0x07 | nonce质量信息
0x08 | TPM 2.0时钟时间鉴定
0x09 | PCR证明
0x0a | 签名
0x0b | 上一次休眠的报告
0x0c | 追加日志报告
0x0d | DICE签名的证明（用于隐式证明）
```

## AK证书链

AK证书链包含了AK证书和其它有助于校验方验证AK的证书，每个证书有一个头部字段表示证书的长度，然后是DER格式的X.509证书数据：
```
0x01 || length || number of certificates to follow (2 bytes) ||
  (DER encoded first certificate ||
  DER encoded second certificate ||
  ...)
```

## TPM 2.0签名的证明数据（用于隐式证明）

用于隐式证明的签名密钥是指有策略绑定，只有在系统处于特定状态时才能进行签名的密钥，这个字段包含的是使用AK和TPM2_Certify生成的数据。

```
Type           | Name        | Description
TPM2B_ATTEST   | certifyInfo | 被签名的结构
TPMT_SIGNATURE | Signature   | 签名
```

TLV定义如下：
```
0x02 || length || (pcrUpdateCounter || TPML_PCR_SELECTION || TPML_DIGEST || TPM2B_ATTEST || TPMT_SIGNATURE)
```

certifyInfo字段包了密钥的策略，策略通过TPM2_PolicyPCR命令得到，因此限制了密钥的使用。
为了使校验方易于校验，这个字段也包含了PCR值。

## TPM1.2和2.0的PCR值

TPM1.2只支持SHA1，每个PCR值是20字节。
```
0x03 || length (4 bytes) || (
  number of PCRs (1 byte) ||
  first PCR (1 byte) || valueof first PCR||
  second PCR (1 byte) || value of second PCR ||
  ....)
```

TPM2.0的定义：
```
(0x04) || length || (pcrUpdateCounter || TPML_PCR_SELECTION || TPML_DIGEST)
```

## PCR日志

PCR日志是包含所有被扩展到PCR报告值的日志，规范并未定义日志的形式，取决于平台规范，规范很慷慨地给了8个字节作为长度。

```
0x05 || length (8 bytes) || (log)
```

## 证明的时效性

```
0x06 || 0x00000002 || (freshness element indicator)
```

TAP定义了三种方法来提供证明的时效性：

* 0x0000 校验方提供的nonce
* 0x0001 可信的第三方提供的nonce
* 0x0002 从TPM时钟得到的数据


前面提到的都是校验方发送nonce的场景（也是主流场景），后两个是用在基于时钟时间的证明。

## nonce质量信息

```
0x07 || length || (Qualification Number || Data)
```

在校验方不提供nonce的情况下，这个信息用于证明nonce的时效性，规范定义了两种：
* 0x0000 nonce是时间戳的一个哈希值，包含的数据为时间戳
* 0x0001 指向一个URL和取得nonce的时间，包含的数据是URL+时间

### TPM2.0时钟时间鉴定

如果证明方无法从校验方获得nonce，则可以使用时钟时间的证明流程，这个流程可以参见规范。

## 显式证明

对于TPM1.2有两种类型数据：TPM_Quote和TPM_Quote2。
TPM2.0有三种类型，一种是使用TPM2_Quote，另两种是使用审计会话，审计会话又可分为使用nonce和时钟两种。
使用审计会话的方式必须包含pcrCounter和PCR原始值两个信息。

TPM2_Quote兼容TPM1.2的两种类型，
但是PCR值和pcrCounter不在Quote数据中（在前面的分析中也可以看到，获取这个数据集合不是原子操作，如果过程中间产生了PCR扩展，
则会导致校验失败。所以规范推荐的是用审计会话的方式，但是目前还没有看到相应的实现（tpm2-tools也没有对应的命令实现，TPM2_GetSessionAuditDigest）。


## 隐式证明

如果预计一个系统在长时间内处于固定的状态，可以使用相对简单的隐式证明的方法来证明系统处于该状态。
证明方创建一个签名密钥并且与系统状态绑定，因此，任何表明证明方能够用该密钥进行签名的行为都可以证明证明方处于预期状态。

它的TLV定义也很简单：
```
0x0A || length || signature
```

## 其它项

* 前一次休眠的报告 个人感觉主要用于PC类设备。
* 追加日志报告 属于增量式报告，可以避免发送校验方已有的日志数据。

# 小结

本文主要分析了远程证明的原理、过程和交互数据。
