# 斑点(Speckle)开发文档

[English](index.md)

### 概述

Speckle浏览器扩展适用于Chrome，Brave和Firefox。它使用'Web-Extension Polyfill for TypeScript'(webextension-polyfill-ts)来抽象不同浏览器的扩展API调用。

作为普通的浏览器扩展，它由不同组件组成。它们包括后台脚本，弹出页面，选项页面，UI元素和各种逻辑文件。目前，Speckle浏览器扩展的主要逻辑存在于与后台脚本交互的弹出页面中。浏览器扩展存储还用于存储必要的设置和加密的密钥库。

<p align ='center'> <img src ='Dataflow.jpg'> </p>

### 帐户管理

Speckle依靠Polkadot Keyring实现帐户生成，并使用浏览器扩展存储来实现帐户持久性。我们采用sr25519作为我们的默认密钥环类型，因为我们计划在未来实施BIP44 HD钱包，不建议将ed25519用于BIP44 HD钱包。出于同样的原因，我们使用BIP39助记符作为种子生成机制。

目前，Speckle实现了一个简单的钱包。即使它支持多个帐户(密钥对)，每个帐户也将是独立的，这意味着它们必须单独备份。这取决于HD钱包的开发进度，我们可能会在生产发布之前用HD钱包替换简单的钱包。

由于Polkdadot Keyring实现不公开私钥，因此Speckle不支持Metamask等原始私钥备份。它仅支持助记符备份(在帐户生成之前)和密钥库备份(在帐户生成之后)。

用户可以通过两种方式导入帐户，12个字种子短语和密钥库文件。按密钥库文件导入帐户时，必须提供密码。 ed25519和sr25519帐户均受支持。在Polkadot PoC-2及之前生成的帐户也可以导入。

### 安全机制

* 扩展存储而不是本地存储用于保留加密帐户。
* 密码强制为至少8个字符
* 在保存到存储之前，将使用用户密码对从未加密密钥库导入的帐户进行加密
* 密码或助记符只在浏览器扩展分配的内存中。

### 余额组建

Speckle通过订阅来自Polkadot(或任何Substrate)节点的Websocket rpc端点的freeBalance更新来显示帐户余额。一旦地址发生变化，它就会取消更新。这种方式余额回自动更新。

### 代码阅读指南

代码按文件夹组织，文件夹名称描述其中的内容，例如组件，服务等
帐户管理服务位于`/ src / ts / background / services / keyring-vault.ts`文件中。它负责助记符生成，帐户生成，密钥库文件生成，帐户持久性和存储中的帐户加载。

`src / ts / services / keyring-vault-proxy.ts`是`keyring-vault.ts`的服务代理，用于隐藏前端的消息传递复杂性。

`/ src / ts / routes / RouteWithLayout.tsx`是一个HOC(高阶组件)，可用于定义使用不同布局的路由。 `/ src / ts / layouts`包含两个Layout组件，`LoginLayout.tsx`是一个容器组件，它在用户解锁钱包之前为屏幕定义一个公共页眉和页脚，`DashboardLayout.tsx`是一个定义公共内容的容器组件。用户解锁钱包后，仪表板屏幕的页眉和页脚。

### 代码片段

//后台侦听端口
```
browser.runtime.onConnect.addListener(function(port){
  if(port.name！=='__ SPECKLE__')return
  port.onMessage.addListener(function(msg){
    switch(msg.method){
      case FUNCS.LOCK：
        keyringVault.lock()
        port.postMessage({ method：FUNCS.LOCK，result：true })
        break
...
```

//弹出页面发送消息到后台并监听返回消息
```
const port = browser.runtime.connect(undefined，{name：'__ SPECKLE__'})

export function lockWallet()：Promise <boolean> {
  return new Promise <boolean>((resolve，reject)=> {
    port.onMessage.addListener(msg => {
      if(msg.method！== FUNCS.LOCK) return
      if(msg.error){
        reject(msg.error.message)
      }
      resolve(msg.result)
    })
    port.postMessage({ method：FUNCS.LOCK })
  })
}
```

//组件调用服务
```
  handleLockClick =()=> {
    lockWallet()。then(result => {
       console.log(result)
    })
  }
```
