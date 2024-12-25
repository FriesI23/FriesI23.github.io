---
#  friesi23.github.io (c) by FriesI23
#
#  friesi23.github.io is licensed under a
#  Creative Commons Attribution-ShareAlike 4.0 International License.
#
#  You should have received a copy of the license along with this
#  work. If not, see <http://creativecommons.org/licenses/by-sa/4.0/>.
title: 为 Flutter 应用创建 Windows 安装包
excerpt: |
  近期需要为应用的 Windows 版本创建 MSIX 安装包，这里简单记录一下打包流程和遇到的一些问题。
author: FriesI23
date: 2024-12-25 15:00:00 +0800
category: flutter
tags:
  - flutter
  - flutter-windows
  - msix
  - msix-cert
---

近期需要为应用的 Windows 版本创建 MSIX 安装包，这里简单记录一下打包流程和遇到的一些问题。

## 1. 配置 MSIX 打包流程

首先使用 `fluter pub add msix --dev` 将 `msix` 包引入项目，该 Package 封装了 MSIX 打包流程，
我们就不需要自己编写脚本来手动打包了。

`msix` 中有很多选项，具体可以在[文档][pub-msix]中查看，下面只提及一些重要构建的选项，
签名选项将放在下一章节：

### 1.1。 display_name / --display-name / -d

应用名，就是安装后用户可以看到的应用名称。

### 1.2. identity_name / --identity-name / -i

程序的唯一标识名，类似安装上的包名，在 Windows 上包名一般使用类型 `Company.Suite.AppName` 的形式。
标识名将关系到应用存储数据的位置（），因此请务必按照规范设置。

> 如果应用需要在 Microsoft Store 中进行发布，请注意改配置需要与 Dashboard 中配置一致。

### 1.3. msix_version / --version

一般不需要配置，不配置的话会使用 `pubspec.yaml` 中的 `version` 字段。

具体生效顺序请参考[这里][msix-version].

### 1.4. capabilities / --capabilities / -e

根据 ["App capability declarations"][ms-capability] 中说明，按自己项目需要进行配置。

### 1.5. output_name / --output-name / -n

用于配置生成的安装包名称，不需要后缀。这里提供一个自己使用的安装文件名称脚本：

```cmd
set VERSION="unknown"
for /f "tokens=2 delims=:" %%a in ('findstr /R "^version:" pubspec.yaml') do set VERSION=%%a
set VERSION=%VERSION: =%

echo "Building %VERSION%"
dart run msix:create --output-name mhabit_%VERSION%_x64
```

## 2. 构建 MSIX 签名

Windows 使用 `SignTool.exe` 为安装包添加签名，我们不直接使用该工具，而 `msix` 将使用该工具进行签名。

`msix` 默认会生成一个测试签名，如果测试用的话可以直接使用而无需单独配置。
不过如果我们需要自己发布安装包的话，一个固定的签名（不管是公共或者个人）是必要的。
签名需要证书，而证书的获取方式分为：

1. 发布到 Microsoft Store，商店会自动为应用程序提供证书。
2. 购买一个微软认证证书。
3. 自签名。

发布到商店就不说，只说说后两者。

购买证书是一笔不小的花费，不过使用该证书签名的安装包相当于被微软认证，可以正常安装到其他计算机上。

自签名的安装包默认是不能安装到计算机上的，如果计算机已经安装了签名对应的证书，或者使用不签名的方式进行安装。

由于应用还没有到需要买一个证书的体量，因此这里选择了自签名。

### 2.1. 创建 Certificate

我们有多种方式创建证书，一种简单的方式是使用 Powershell 简单的创建一个自签名的证书:

```powershell
$cert = New-SelfSignedCertificate -DnsName $YourWebsite -Type CodeSigning -CertStoreLocation Cert:\CurrentUser\My
$CertPassword = ConvertTo-SecureString -String $YourPassword -Force -AsPlainText
Export-PfxCertificate -Cert "cert:\CurrentUser\My\$($cert.Thumbprint)" -FilePath "publish.pfx" -Password $CertPassword
```

可以使用 `OpenSSL` 的标准流程：

```shell
openssl genrsa -out publish.key 4096
openssl req -new -key publish.key -out publish.csr
openssl x509 -in publish.csr -out publish.crt -req -signkey publish.key
openssl pkcs12 -export -out publish.pfx -inkey publish.key -in publish.crt  # set Export-Password
```

### 2.2. 配置 Certificate

`msix` 提供了一系列选项进行配置，这里也只提出以下重要的：

#### 2.2.1. certificate_path / --certificate-path / -c

即上一步生成 `pfx` 文件。

> **注意**：该选项如果使用 `pfx` 文件时，需要与 [certificate_password] 一起使用。
>
> **注意**：如果使用 `crt`，需确保计算机上已存在并能够读取到对应的私钥。

#### 2.2.2. certificate_password / --certificate-password / -p

`pfx` 文件对应的密码，在生成时填入。

对于使用 Powershell 流程：`ConvertTo-SecureString` 中填入的密码。

对于使用 OpenSSL 流程：`pkcs12 -export` 中填入的密码。

#### 2.2.3. install_certificate / --install-certificate

如果填写 `true` 的话，这会在签名是出现一个字符交互，提示是否需要向当前计算机安装证书。

> **注意**：在 CI 流程中，**一定**要将该选项设置为 false，因为 CI 环境中无法进行交互。
> 一般打包时不需要安装证书，但如有特殊需求请使用相关命令提前进行安装：
>
> ```powershell
> Import-Certificate -FilePath "public.crt" -CertStoreLocation Cert:\LocalMachine\Root
> ```
>
> 或
>
> ```powershell
> Import-PfxCertificate -FilePath "public.pfx" -Password $YourPassword -CertStoreLocation Cert:\LocalMachine\Root
> ```

## 3. 开始打包

使用一下命令进行打包：

```shell
dart run msix:create
# 如果不需要 create 过程中自动构建 (flutter build windows --release)
# 使用 --build-windows false 跳过应用构建流程
#dart run msix:create --build-windows false
```

等待打包流程完毕，可以在 `build/windows/x64/runner/Release/<output_name>.msix` 中获取打包完毕的安装包。

至此，打包流程完毕。

## F1. 最佳实践（非常主观的）

```yaml
# pubspec.yaml
msix_config:
  display_name: Table Habit
  identity_name: Github.FriesI23.TableHabit
  enable_at_startup: false
  logo_path: ./assets/logo/macos-1024x1024.png
  capabilities: internetClient
```

```shell
# build process
echo "${{ secrets.APP_SIGN_KEY_PFX }}" | base64 --decode > windows/certificate/publish.pfx
dart run msix:create \
  --architecture x64 \
  --output-name mhabit_x64 \
  --certificate-path windows/certificate/publish.pfx \
  --certificate-password '${{ secrets.APP_SIGN_KEY_PFX_PASSWORD }}'
```

## F2. 一些问题

### Q1. 为什么使用 MSIX 而不是 MSI 或 EXE 安装包

可以参考 [Comparing MSI vs. MSIX][diff-msix-msi]。简而言之，MSI 是现在，MSIX 是未来，也是 Microsoft Store 中使用的格式，
考虑到后面会讲应用发布在 Store，直接使用 MSIX 也是更方便的选择。

至于 EXE，我没有自定义的需求，因此没有理由单独写一个安装程序。

### Q2. 为什么出现 "No certificates were found that met all the given criteria."

该错误由 `SignTool.exe` 抛出，具体原因多种多样，这里记录一个自己遇到的问题。

我在自己的计算机上使用 `OpenSLL` 创建证书后，使用 `crt` 作为 `--certificate-path` 进行打包，此时一切正常。
不过在编写 Github Action 并测试 CI 构建时，缺出现如上报错。

后面才想起来，自己本地在生成证书时，Windows 已将私钥进行存储，因此本地可以使用 `crt`。
但打包脚本所在的机器没有改私钥，因此无法直接使用。解决的方法有两个：

1. 在打包前将私钥安装到打包机上
2. 使用 `pfx` 打包

果断使用 `pfx`，毕竟手动安装证书和使用 `pfx` 没有本质区别，而后者更方便。

置于隐私问题，根据 ["Using secrets in GitHub Actions"][gh-secret-bin] 中教程，将 `pfx` 文件进行编码：

```shell
# macos
# base64 -i cert.der -o cert.base64
# linux
base64 -w 0 cert.der > cert.base64

# upload to github
gh secret set REPLACE_TO_YOUR_SECRET_NAME < cert.base64
```

<!-- refs -->

[diff-msix-msi]: https://www.techtarget.com/searchenterprisedesktop/tip/Comparing-MSI-vs-MSIX
[pub-msix]: https://pub.dev/packages/msix
[msix-version]: https://github.com/YehudaKremer/msix/blob/main/doc/msix_version.md
[ms-capability]: https://learn.microsoft.com/en-us/windows/uwp/packaging/app-capability-declarations
[gh-secret-bin]: https://docs.github.com/actions/security-guides/encrypted-secrets
