# yubikey-ssh

There are quite a few one pagers describing how to set up ssh authentication using a
Yubikey, but most seem to involve complicated other bits of software and environment, like
OpenSC or GPG. Yubico have also been deprecating some of their tools and writing new ones,
though they have plenty of docs using the old ones.

What a mess!

Here’s a much simpler way.

## Goal

I want to use ECDSA instead of RSA, have an ssh private key only on my Yubikey, and when
ssh’ing to a server I want my Yubikey to provide the credentials seamlessly.

## Getting Started

Hardware:

* Mac running macOS Monterey (12.x) or Ventura (13.x) - I’m using an M1 MacBook Pro
* Yubikey - I’m using a Yubikey 5C NFC with firmware version 5.43
* Ideally another Yubikey for backup purposes

Software:

Both solutions require that you ignore Apple's version of openssh, and instead build your
own. This has a major-sounding downside - it doesn’t support the Keychain like Apple’s
openssh does. Obviously your Yubikey won’t need it but if you have other ssh keys then
something like [Secretive](https://github.com/maxgoedjen/secretive) is a great and
secure-looking solution.

* I use [MacPorts](https://www.macports.org/) to get or build as much third party software
as possible. MacPorts will put its install directories at the start of your PATH.

## Simplest solution

This will work with any OpenSSH server newer than 8.2, which is when OpenSSH added support
for FIDO2. FIDO2 is the standard that security keys use for website authentication, which
is what Apple uses for "passkeys".

This solution requires that you build openssh with FIDO2 support. To do this:

```
$ sudo port install openssh +fido2 -xauth
```

The port command should tell you it needs to install some other things including `libfido2`.
Say “y” and wait a little. Start a new shell, insert your Yubikey and and create a new
key:

```
$ cd ~/.ssh
$ ssh-keygen -t ed25519-sk -C name-of-my-yubikey
```

This uses ed25519, which is believed to be superior to ecdsa-sha1-nistp256. The “-sk”
means do the cryptography and keep the private key on the Yubikey. That's pretty cool.

The string after `-C` is just for your benefit, and can be anything you want. The
`ssh-keygen` command will prompt you to touch your Yubikey. It will then prompt you to
password-protect your new private key file. I chose not to password-protect it, as it
is actually really just a reference to the real private key on your Yubikey.

You should now have two files:

* id_ed25519_sk
* id_ed25519_sk.pub

After adding the new public key to the `authorized_keys` file on your servers, you should be
able to ssh to the server and you will be prompted to touch your key. Job done!

```
$ ssh microserver.local
Confirm user presence for key ED25519-SK SHA256:AUB/wwL+mujgqSNhXuiur3CMQFLgKToaTcrgRATWXpc
User presence confirmed
SmartOS (build: 20211216T012707Z)
[root@microserver ~]# 
```

## Harder solution

This will work with older OpenSSH servers, but it is more fiddly. On Monterey you need to
install a newer version of openssh. I'm not sure about Ventura, but you might want a newer
openssh anyway for security reasons. On both you also need to install some software from
Yubikey.

```
# Monterey only?
$ sudo port install openssh -xauth
# Monterey and Ventura
$ sudo port install yubico-piv-tool yubikey-manager
```

The older `yubico-piv-tool` is only needed to provide their PKCS#11 library. These
instructions will use the newer tool called `ykman` which is from the `yubikey-manager`
port.

## Generate the Key

Insert your Yubikey.

You will be treating the Yubikey as a **PIV** in other words, a smart card. Don’t worry,
the Yubikey can still be used as a FIDO2 key for web authentication.

```
$ ykman piv keys generate 9a -a ECCP256 ~/.ssh/yubikey.pem
$ ykman piv certificates generate -s 'CN=Yubikey SSH' -a SHA512 9a ~/.ssh/yubikey.pem
$ ssh-keygen -D /opt/local/lib/libykcs11.dylib
```

The magic `9a` is the PIV “slot” used for holding authentication information. The `-s`
value is the X.509 certificate subject DN written using the LDAP format. It doesn’t matter
much what this is. It appears in the `ykman piv info` output but not in the SSH public key.
The `-D` argument points to the PKCS#11 library in the `yubico-piv-tool` port.
