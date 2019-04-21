# Speckle Development Document

### Overview

Speckle browser extension works on Chrome, Brave and Firefox. It uses ‘Web-Extension Polyfill for TypeScript’ (webextension-polyfill-ts) to abstract extension API calls for different browsers.

As a normal browser extension, it is made of different, but cohesive components. They include background scripts, a popup page, an options page, UI elements and various logic files. At the moment, the most logic of Speckle browser extension exists in the popup page that interacts with the background script. Browser local storage is also used to store necessary settings and encrypted keystore.

<p align='center'><img src='Dataflow.jpg'></p>

### Account Management

Speckle relies on Polkadot Keyring to implement account generation, and uses browser extension  storage to implement account persistence. We adopts sr25519 as our default keyring type, as we plan to implement BIP44 HD wallet in future and ed25519 is not recommended for BIP44 HD wallet. For the same reason, we use BIP39 mnemonic as seed generation mechanism.

Currently Speckle has implemented a simple wallet. Even though it will support multiple accounts (keypairs), each account will be independent, which means they have to be backed up individually. It depends on HD wallet development progress, we may replace simple wallet with HD wallet before production release.

As Polkdadot Keyring implementation does not expose private key, Speckle does not support raw private key backup like Metamask. It only supports mnemonic backup (before account generation) and keystore backup (after account generation).

### Code snippets

//background listens to the port
```
browser.runtime.onConnect.addListener(function (port) {
  if (port.name !== '__SPECKLE__') return
  port.onMessage.addListener(function (msg) {
    switch (msg.method) {
      case FUNCS.LOCK:
        keyringVault.lock()
        port.postMessage({ method: FUNCS.LOCK, result: true })
        Break
```

//popup service trigger message event
```
const port = browser.runtime.connect(undefined, { name: '__SPECKLE__' })

export function lockWallet (): Promise<boolean> {
  return new Promise<boolean>((resolve, reject) => {
    port.onMessage.addListener(msg => {
      if (msg.method !== FUNCS.LOCK) return
      if (msg.error) {
        reject(msg.error.message)
      }
      resolve(msg.result)
    })
    port.postMessage({ method: FUNCS.LOCK })
  })
}
```

// component calls service
```
  handleLockClick = () => {
    lockWallet().then(result => {
      console.log(result)
    })
  }
```


