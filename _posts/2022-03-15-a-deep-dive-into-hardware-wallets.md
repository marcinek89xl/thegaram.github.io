---
layout: post
title: "A Deep Dive into Hardware Wallets"
author: "Peter"
---

*Originally published at [medium.com/conflux-network](https://medium.com/conflux-network)*

As [Ledger](https://www.ledger.com) is adding support for [Conflux](https://confluxnetwork.org), it's a good chance to take an in-depth look at how hardware wallets work. If you just want to try Ledger on Conflux, check out the [Conflux Core](https://developer.confluxnetwork.org/sdks-and-tools/en/using_ledger_on_core) and [Conflux eSpace](https://developer.confluxnetwork.org/guides/en/using_ledger_on_espace) Ledger tutorials.

## What makes hardware wallets secure?

Hardware wallets are widely considered one of the most secure ways of holding crypto assets. Instead of storing your private keys in encrypted files on your computer or mobile phone, hardware wallets let you store your keys on a secure hardware device, usually on a [Secure Element](https://encyclopedia.kaspersky.com/glossary/secure-element) chip.

Crucially, **your private keys are generated on the device**. This ensures that they never leave your wallet. As long as your wallet can guarantee that its memory cannot be easily accessed from a connected computer or through side channels, your keys are secure.

> ⚠️ The motto "your private keys never leave the device" is not really true. You are asked to make a paper backup of your seed phrase when you initialize your device. If someone gains access to your seed phrase, they can sidestep all the security guarantees offered by hardware wallets.

Your computer is connected to the internet. This makes it subject to all kinds of vulnerabilities from virus infections to direct hacking. The secrets stored on your hardware wallet are never exposed, even when it is connected to your computer through USB or Bluetooth.
Some widely used hardware wallets include [Ledger](https://www.ledger.com), [Trezor](https://trezor.io), and [Keystone](https://keyst.one).

## How do hardware wallets work?

Hardware wallets often work alongside a companion wallet like Ledger Live, [MetaMask](https://metamask.io), or [Fluent](https://fluentwallet.com).

Let's say you want to use your Ledger hardware wallet with Fluent. When you initiate a transaction (e.g. click `Swap` on Uniswap), Fluent constructs or receives an \*unsigned\* transaction. Instead of using your local private key to sign this transaction, Fluent forwards it to your Ledger wallet through USB.

![Fluent Wallet sign transaction popup window](/assets/img/2022-03-15-a-deep-dive-into-hardware-wallets/fluent-popup.png "Fluent Wallet sign transaction popup window")

Upon receiving a new signing request, Ledger first decodes the transaction and displays it to the user for confirmation. Ideally, the user should check important fields, including the receiver address, the transfer amount, and the specific action that the transaction will execute. **What You See Is What You Sign** (WYSIWYS).

![Transaction confirmation on the Ledger device](/assets/img/2022-03-15-a-deep-dive-into-hardware-wallets/ledger-workflow.gif "Transaction confirmation on the Ledger device")
_Approving a Conflux transaction on the Ledger Nano S_

If the user approves the transaction, Ledger will then create a signature using the private key stored on the device and send this signature back to Fluent. Fluent attaches this signature to the original transaction and sends it to the network for execution.

> Most hardware wallets will not actually store any private keys. Instead, they only store the seed and derive private keys from it on-demand. For more info, read our upcoming article about HD wallets and BIP32.

#### Example: Signing Conflux transactions

Let's say Alice wants to send 10 CFX to Bob using her Ledger hardware wallet through Fluent.

Alice: `cfx:aap3kjaha5r13a0xayk80rym42sbhwufdaejj5sv63` <br> Bob: `cfx:aar40g1p4fcp0k608t4xyfwwgfcmhh70f2kv9c8zdp`

First, we need to construct a transaction:

```
{
  "nonce": 11,
  "gasPrice": 1000000000,
  "gasLimit": 21000,
  "to": "cfx:aar40g1p4fcp0k608t4xyfwwgfcmhh70f2kv9c8zdp",
  "value": 10 * 1e18,
  "storageLimit": 0,
  "epochHeight": 40000000,
  "chainId": 1029,
  "data": "0x"
}
```

In this JSON object, you only need to care about the to and value fields. The rest is filled by Fluent automatically.

The second step is to RLP-encode this transaction. [RLP](https://eth.wiki/fundamentals/rlp) is the binary encoding format used by Conflux and Ethereum. You can try RLP encoding and decoding using [this](https://toolkit.abdk.consulting/ethereum#rlp) tool. Make sure you use hex numbers and hex addresses.

```
RLP([
  "0xb",
  "0x3b9aca00",
  "0x5208",
  "0x1Bab1aEcd144cb2796f3F53A16523144a39FB62E",
  "0x8ac7230489e80000",
  "0x0",
  "0x2625a00",
  "0x405",
  "0x",
])
=
0xf10b843b9aca00825208941bab1aecd144cb2796f3f53a165
23144a39fb62e888ac7230489e80000008402625a0082040580
```

At this point, Fluent sends a signing request to Ledger:

```
Dear Ledger,
Please use account 44'/503'/0'/0/0 to sign the following transaction:

0xf10b843b9aca00825208941bab1aecd144cb2796f3f53a165
23144a39fb62e888ac7230489e80000008402625a0082040580

Sincerely,
Fluent
```

`44'/503'/0'/0/0` simply means that we want to use the first Conflux address derived from the on-device seed phrase. For details, see [HD wallets](https://learnmeabitcoin.com/technical/hd-wallets), [BIP32](https://en.bitcoin.it/wiki/BIP_0032), and [SLIP44](https://github.com/satoshilabs/slips/blob/master/slip-0044.md).

Ledger will then decode the transaction (`0xf10b84...`) and display the transaction details to Alice. Alice must check that it's all correct (10 CFX to Bob's address), then confirm. At this point, Ledger will derive the private key of account `44'/503'/0'/0/0` and use it to sign the transaction (`0xf10b84...`). This will result in an ECDSA signature like this:

```
0x00f9071161c2dbc19dabf54d14d42944cecacf61943a9898f4f64c8aa6d23a58
b664ea364f092d23d7a94388f2f43cf54a86fe644d221e822210fde413d406ebb6
```

Or, equivalently:

```
{
  "v": "0x0",
  "r": "0xf9071161c2dbc19dabf54d14d42944ce...",
  "s": "0x64ea364f092d23d7a94388f2f43cf54a..."
}
```

Finally, Fluent attaches the signature to the transaction, RLP-encodes it, and sends it to a node using `cfx_sendRawTransaction`.

```
{
  "to": "cfx:aar40g1p4fcp0k608t4xyfwwgfcmhh70f2kv9c8zdp",
  "value": 10 * 1e18,
  ...
  "v": "0x0",
  "r": "0xf9071161c2dbc19dabf54d14d42944ce...",
  "s": "0x64ea364f092d23d7a94388f2f43cf54a..."
}
```

Note that Alice's address is nowhere to be found in this transaction object. That is because her address can be recovered from the signature.

## Security considerations

When considering security risks, it's always useful to think from an attacker's perspective. Let's put on our attacker hat and see what a malicious dapp, a malicious wallet, or a malicious distributor could do.


#### Altering transaction data

Let's say the attacker manages to get you to install a malicious fork of MetaMask called MetaM😈sk, hoping to get you to transfer your assets on your Ledger wallet to the attacker's address. Your MetaM😈sk will then forge a malicious transaction and send it to Ledger.

In this case, the attacker has no control over Ledger, so the device will decode the transaction and show you the exact transaction details. At this point, you will detect the attack and reject the transaction.

> ⚠️ Whatever wallet you're using, it is important that you **always** carefully check the transaction details before you confirm.

Let's say you sign a valid transaction on Ledger but MetaM😈sk alters the receiver address after this. In this case, the signature will not match the transaction, and it will be rejected by the network.

> One signature only works for one specific transaction.

Instead of using a malicious fork of Metamask, the attacker could also install a virus on your system. This virus would intercept and tamper with messages between MetaMask and Ledger. This attack will run into the same issues: You will either detect the tampering on Ledger, or the signature will not match.

#### Blind signing

Let's say the attacker forks the Uniswap interface and hosts it on the domain `uniswаp.org`. (Note that "а" is U+0430 *Cyrillic Small Letter A*, not U+0061 "a" *Latin Small Letter A)*. You use this page and try to swap 10k USDT for ETH. Instead of using your address as the receiver for ETH, this malicious dapp will fill in the attacker's address.

Normally, the "action" of a transaction (e.g. swapExactTokensForTokens) is an intimidating hex string in your transaction's data field. If you approve an opaque hex string, this is called **blind signing**. With blind signing, you lose most of the security you gained by using a hardware wallet.

```
0x18cbafe5000000000000000000000000000000000000000000000000f0af123325a6c67e0
000000000000000000000000000000000000000000000000012d1cd2d74523b000000000000
00000000000000000000000000000000000000000000000000a000000000000000000000000
04d7839f543eb6fd24bb212ed2fa8d64b73c7e0590000000000000000000000000000000000
00000000000000000000006229a8e1000000000000000000000000000000000000000000000
000000000000000000200000000000000000000000057ab1ec28d129707052df4df418d58a2
d46d5f51000000000000000000000000c02aaa39b223fe8d0a0e5c4f27ead9083c756cc2
```

It is very important that your wallet interprets this hex string and the contract address and shows you what is it exactly what you're signing (WYSIWYS).

```
UniswapV2Router.swapExactTokensForETH(
  in : 17.343100700911388286 sUSD
  out: >0.005297228741890619 Ether,
  to : 0x4D7839f543eb6FD24bb212ED2fa8D64B73c7E059
)
```

Most hardware wallets are able to decode commonly used transaction types (ERC20, ERC721, Uniswap, etc.), and they disallow blind signing by default. However, decoding on hardware wallets has its [limitations](https://blog.keyst.one/blind-signing-a-security-black-hole-for-the-ethereum-community-13f909b848b6).

#### Supply chain attacks

The most obvious target for attacks on hardware wallets is the seed phrase backup. If the attacker has your seed phrase, they have your assets.

Normally, the device will generate a new random seed phrase when you first initialize it. However, there have been [cases](https://www.ledger.com/scam-second-hand-ledger-device) of users receiving devices that had already been initialized. In this case, the attacker initialized the device and obtained the seed phrase. If the user starts using this device, the attacker can transfer out their funds at any time in the future.

> ️️⚠️ It is important to **always** reset your device when you first initialize it.

Supply chain attacks can also come in different flavors. Let's say you order a device online. How can you be sure that the device you receive is authentic and not a counterfeit with a build-in backdoor? Most manufacturers offer ways to check the [authenticity](https://support.ledger.com/hc/en-us/articles/4404382029329-Check-hardware-integrity) of your hardware device.

#### Side-channel attacks

When an attacker gains physical access to your hardware wallet, they can sometimes exploit some physical characteristics of the device to break it. For instance, an attacker might track your device's power consumption while trying to guess your PIN, and gain some extra information from this. Such attacks are often called [side-channel attacks](https://en.wikipedia.org/wiki/Side-channel_attack). Hardware wallet manufacturers offer various levels of protection against such attacks.

## Conclusion

Hardware wallets store your account seed securely and sign transactions in a secure environment. By ensuring that your private keys never leave the device and that "What You See Is What You Sign", you can have much higher certainty about your assets' security.

Now it's time for you to experience this: Check out our tutorials for using Ledger hardware wallets on [Conflux Core](https://developer.confluxnetwork.org/sdks-and-tools/en/using_ledger_on_core) and [Conflux eSpace](https://developer.confluxnetwork.org/guides/en/using_ledger_on_espace).