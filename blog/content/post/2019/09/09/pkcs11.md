---
title: "New pkcs11 Security Provider"
date: 2019-09-09T9:00:00+01:00
tags: ["pkcs11", "security"]
draft: false
author: Paul Tittle
---

The latest release of Choria will have a new security provider, called `pkcs11`! This blog post will go over how to use it in various configurations. But first, a review of what pkcs11 is and how it's useful.

## What is pkcs11?

pkcs11 stands for "Public Key Cryptography Standard #11". It's a set of standards for how to interact with a cryptographic token. You may have heard of HSMs or smart cards. pkcs11 is how software interacts with these things.

## Why should I use pkcs11?

You may be compelled to use it due to the environment you work in. Yubikeys and CACs are being used more and more in large-scale environments. But it's a good idea to investigate the use of these things if you already aren't. The power of HSMs is that the sensitive cryptographic material is generated on the hardware and never leaves it. So instead of opening your private key file and signing hashes with it, you're handing the hash to your Yubikey, which signs it and returns the data. There are compliance advantages too (because of the stronger security). Some HSMs are FIPS-compliant, which some computing environments require.

<!--more-->

## How do I get started using pkcs11?

The following example is for an OS X/Linux environment. Let's go over how you might setup a yubikey (a brand of HSM) to be used with Choria. `ykman` is a utility that you will have to download to follow along.

Let's clear out the Yubikey first.

```nohighlight
$ ykman piv reset
WARNING! This will delete all stored PIV data and restore factory settings. Proceed? [y/N]: y
Resetting PIV data...
Success! All PIV data have been cleared from your YubiKey.
Your YubiKey now has the default PIN, PUK and Management Key:
	PIN:	123456
	PUK:	12345678
	Management Key:	010203040506070801020304050607080102030405060708
```

Now we need to generate a private key on the Yubikey:

```nohighlight
$ ykman piv generate-key -P 123456 -m 010203040506070801020304050607080102030405060708 --pin-policy ONCE --touch-policy CACHED 9a test.pub
```

The command above makes it so you have to enter your pin before you can send crypto requests, and when you send a crypto request, you have to touch the Yubikey. Once touched, you can perform more requests within 15 seconds before you have to touch it again.

The command also produces a public key file, which you will use to generate a CSR so that you can get a signed cert for your Yubikey.

```nohighlight
$ ykman piv generate-csr -s ptittle -P 123456 9a test.pub test.csr
```

With the above, we've generated a CSR with the Common Name `ptittle`, using the pubkey we got earlier, and producing a csr file.

Now that you have a CSR file, you can go get it signed by any Certificate Authority. Once you have, get the certificate file that they give you, and put it in `test.crt`.

```nohighlight
$ ykman piv import-certificate --pin 123456 -m 010203040506070801020304050607080102030405060708 9a test.crt
```

That command imports your certificate to the Yubikey. Now your Yubikey is configured to be used by Choria!

The above tutorial, while specific to Yubikeys, is also applicable to other HSMs. You will always need to generate a key, set a pin, create a CSR, then get the certificate back onto the HSM.

## How do I configure Choria to use my HSM?

The following assumes that you're running Choria on the same host that your HSM is plugged into. We will talk about forwarding HSM connections a little later.

First, install opensc. This is a library which lets software talk to the HSM. Then, in your _client configuration file_, add/change the following:

```nohighlight
plugin.security.provider: "pkcs11"
plugin.security.pkcs11.slot: 0
plugin.security.pkcs11.driver_file: "/usr/local/lib/opensc-pkcs11.so"
```

Your driver file location will probably be different, so adjust accordingly.

Then, configure Choria to trust your certificate's Common Name, as normal.

To test, try running `choria req rpcutil ping`. You will be prompted for your pin. Once entered, it should run as normal!

## How do I forward my HSM to my cluster's bastion host?

It's very likely that your compute environment isn't directly accessible from your workstation. In this case, you can use the [p11 kit](https://p11-glue.github.io/p11-glue/p11-kit.html) to create a socket that you can then forward via SSH to your cluster's bastion host. From there, you configure Choria as above, but using the p11-kit driver file. That driver knows how to find your forwarded socket, so it can send crypto requests back to your workstation!

Let's unpack all of that in detail and start with workstation-side changes. Add the following to your `.bash_profile` on your workstation:

```nohighlight
function proxyyubi () {
    mkdir -p ~/.xdg/
    rm -rf ~/.xdg/p11-kit/
    export XDG_RUNTIME_DIR=~/.xdg/
    token_url=$(p11tool --provider /usr/local/lib/opensc-pkcs11.so --list-token-urls)
    eval $(p11-kit server --provider /usr/local/lib/opensc-pkcs11.so "${token_url}")
    ln -sf ${P11_KIT_SERVER_ADDRESS#*=} /Users/ptittle/pkcs11
    echo "Created yubikey socket, proceed to ssh as normal"
}
```

Replace the library paths with where your opensc lib is. And also change the homedir in the relevant place. Now install the p11-kit on your workstation.

Now connect to your bastion host and install p11-kit. Run `systemd-path user-runtime`; the output should look like `/run/user/632000069`.

Go back to your workstation, edit your `~/.ssh/config`, and add an entry for your bastion host if you don't already have one. Add the following:

```nohighlight
RemoteForward /run/user/<runtime-user-id>/p11-kit/pkcs11 /Users/<username>/pkcs11
```

Now plug in your HSM, run `proxyyubi`, then ssh into your bastion. If you see any errors about forwarding a socket, you'll need to check your steps and make sure the bastion allows forwarding of sockets.

The Choria client configuration on the bastion will only differ by what library you specify. You'll want to use your p11-kit library, since it's the one that knows how to find your forwarded socket. Try running `choria req rpcutil ping`!

If you're still running into problems, check out the [p11-kit forwarding steps](https://p11-glue.github.io/p11-glue/p11-kit/manual/remoting.html).

## Summary

Using HSMs via pkcs11 is a great way to secure your computing environment while also being convenient to use. With Choria, you can use your local HSM directly, or forward an HSM-connected socket to a bastion host and use it there. Feel free to request any features in the [security provider repo](https://github.com/choria-io/go-security)!
