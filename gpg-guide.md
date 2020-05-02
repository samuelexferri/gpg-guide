# GPG Guide

## Introduction

### What are keys?

In public key cryptography, a key is actually a pair: a public key, and a
private key. You use the private key to digitally sign files, and others
use the public key to verify the signature. Or, others use the public key
to encrypt something, and you use the private key to decrypt it.

As long as only you have access to the private key, other people can
rely on your digital signatures being made by you, and you can rely on
nobody else being able to read messages encrypted for you.

### What are subkeys?

**OpenPGP** further supports subkeys, which are like the normal keys,
except they're
bound to a master keypair. A subkey can be used for signing or for
encryption. The really useful part of subkeys is that they can
be revoked independently of the master keys, and also stored separately
from them.

In other words, subkeys are like a separate keypair, but automatically
associated with your main keypair.

### Why?

Subkeys make key management easier!

The master keypair is quite important:
it is the best proof of your identity online and if
you lose it, you'll need to start building your reputation from scratch.

Subkeys make life easier: you already have an automatically created encryption subkey and you create another subkey for signing, and you keep those on your main computer. You publish the
subkeys on the normal keyservers, and everyone else will use them instead of the master keys for encrypting messages or verifying your message signatures. Likewise, you will use the subkeys for decrypting and signing messages.

You will need to use the master keys only in exceptional circumstances, namely when you want to modify your own or someone else's key.  More specifically, you need the master private key:

-   when you sign someone else's key or revoke an existing signature.
-   when you add a new UID or mark an existing UID as `primary`.
-   when you create a new subkey.
-   when you revoke an existing UID or subkey.
-   when you change the preferences (e.g., with `setpref`) on a UID.
-   when you change the expiration date on your master key or any of its subkey.
-   when you revoke or generate a revocation certificate for the complete key.

(Because each of these operation is done by adding a new self- or revocation signatures from the private master key).

Since each link of the Web of Trust is an endorsement of the binding between a public key and a user ID, OpenPGP certification signatures (from the signer's private master key) are relative to a UID and are irrelevant for subkeys.  In particular, subkey creation or revocation does not affect the reputation of the master key.  So in case your subkey gets stolen while your master key remains safe, you can revoke the compromised subkey and replace it with a new subkey without having to rebuild your reputation and without reducing reputation of other people's keys signed with your master key.

### Multiple Subkeys per Machine vs. One Single Subkey for All Machines

One might be tempted to have one subkey per machine so that you only need to exchange the potentially compromised subkey of that machine. In case of a single subkey used on all machines, it needs to be exchanged on all machines in case of a compromising.

But this only works for signing subkeys. If you have multiple encryption subkeys, gpg is said to encrypt only for the most recent encryption subkey and not for all known and not revoked encryption subkeys.

## Creating new keys

First create a master keypair (private and public keys):

```bash
$ gpg --expert --full-gen-key
```

The user will be prompted to answer several questions:

1.  Algorithm: `RSA (set your own capabilities)`.
2.  Capability: only `Certify (C)`; disable the default capabilities by entering the related letter, one capability after the other.
3.  Size: `4096 bits`.
4.  Expiration date: a period of a year is enough most of the time; it's possible to change it afterwards.
5.  Details: real name, email address and comment for the key's purpose.
6.  Passphrase (also blank space admitted).

In another terminal, in order to create entropy, run a disk write performance benchmark using:

```bash
$ dd bs=1M count=1024 if=/dev/zero of=test conv=fdatasync
```

When the master keys were created, it will display:

    gpg: key 9016AAAFCA7213FE marked as ultimately trusted
    gpg: revocation certificate stored as '/home/user/.gnupg/openpgp-revocs.d/F4123A7E12AFACB0CEBB584E9016AAAFCA7213FE.rev'
    public and secret key created and signed.

    pub   rsa4096 2020-05-01 [C] [expires: 2021-05-01]
          F4123A7E12AFACB0CEBB584E9016AAAFCA7213FE
    uid   Mario Rossi (Hello World!) <mariorossi@gmail.com>

Where:

-   Key ID is `9016AAAFCA7213FE`.

-   Key Fingerprint is `F412 3A7E 12AF ACB0 CEBB 584E 9016 AAAF CA72 13FE`.

The master key is your `Certify (C)` key for certification. It is the key used to sign other people's keys and to manage your own subkeys and identities.

Let's create subkeys, it is important to have one dedicated to each task:

-   `Authenticate (A)`: authenticating is used to sign other people's public keys, as you "certify" that the key belongs to them, so if you know Bob, and you know Alice, but Alice doesn't know Bob, but you have signed both their keys, when Alice looks up Bob's public key and sees your signature on the public key, she knows she's got the right one.
-   `Sign (S)`: signing is affixing a digital signature to a message or file, verifying that the message/file must have come from you, and only you.
-   `Encrypt (E)`: encrypting is to use your private key to encrypt a message to your public key, so you can encrypt a file/message that you and only you can decrypt; otherwise, someone use the public key to encrypt something, and you use the private key to decrypt it.

First list the available keys:

```bash
$ gpg --list-keys
```

Edit it to add subkeys: to do this, you will need to switch to expert mode:

```bash
$ gpg --expert --edit-key <key-id>
```

You are now in edit mode. Add the authentication key with the `addkey` command and repeat the procedure for the encryption and signing keys.

```bash
# gpg> addkey
# (follow prompts)
# gpg> save
```

Now we have an OpenPGP keypair with its identity and three subkeys with each a capability.

    pub   rsa4096 2020-05-01 [C] [expires: 2021-05-01]
          F4123A7E12AFACB0CEBB584E9016AAAFCA7213FE
    uid   [ultimate] Mario Rossi (Hello World!) <mariorossi@gmail.com>

    sub   rsa4096 2020-05-01 [A] [expires: 2021-05-01]
    sub   rsa4096 2020-05-01 [S] [expires: 2021-05-01]
    sub   rsa4096 2020-05-01 [E] [expires: 2021-05-01]

## Key administration

List public keys:

```bash
$ gpg --list-keys
$ gpg --list-sigs
$ gpg --fingerprint
```

List private (secret) keys:

```bash
$ gpg --list-secret-keys
```

Delete a public key:

```bash
$ gpg --delete-key <key-id>
```

Delete a private key:

```bash
$ gpg --delete-secret-key <key-id>
```

**Note:** `<key-id>` refers to a key by the name of its owner, email address,
the key's fingerprint, by its 8-digit hex ID or similar.

### Edit a key

To open a menu for editing key related tasks, run:

```bash
$ gpg --edit-key <key-id>
# gpg> key 1 (refers to a sub-key (e.g. the first one))
# gpg> (command)
```

Note: `key <num>` toggle a sub-key, all selected keys are displayed with `sub*`; by default primary key is selected. Similar is `uid <num>` to toggle a user IDs.

Useful commands:

-   `help`: display all commands.
-   `pref`: show preferences.
-   `keyserver`: set a default keyserver.
-   `adduid`: add user ID to this key.
-   `addphoto`: add a photo to this key.
-   `addkey`: add a subkey to this key.
-   `enable/disable`: enable/disable selected keys.
-   `passwd`: change passphrase.
-   `clean`: compact any user ID that is no longer usable (revoked or expired).
-   `revkey`: revoke a key.
-   `expire`: change expiration date of key.

### Revocation certificate

If for instance the private key bas been stolen, the UID has been changed or you
forget the passphrase, it is important to notify others that the public key
should no longer be used. After generating a new key it is recommended to
immediately generate a **revocation certificate**:

```bash
$ gpg --output revoke.asc --gen-revoke <key-id>
```

Store the file `revoke.asc` somewhere safe (it is smart to print this certificate). It can be used to revoke the key
later when the private key is compromised.

To revoke your key, import the revocation certificate:

```bash
$ gpg --import revoke.asc
```

If a keyserver is used, update the keyserver as well:

```bash
$ gpg --keyserver <keyserver-name> --send <key-id>
```

If you've got a revocation certificate and are sure you never might lose access to both your private key and revocation certificate at the same time (consider fire, (physical) theft, official institutions searching your house), there is absolutely no use in setting an expiry date apart from possible confusion and more work extending it.

Even worse, expiry dates might provide a false sense of security. The key on the keyservers expired, so why bother to revoke it? There is a large number of well-connected RSA 512 bit keys on the keyserver network, and probably a comparabily large number of weak DSA keys (because of the Debian RNG problems). With faster processors and possibly new knowledge on algorithm weaknesses, an attacker might in future be able to crack the expired, but non-revoked key and use it!

See [backup](bakcup) section and [renew an expired key](renew-an-expired-key) section.

### Backup

Let’s save all keys:

```bash
$ gpg --export --armor <key-id> > <key-id>.pub.asc
$ gpg --export-secret-keys --armor <key-id> > <key-id>.priv.asc
$ gpg --export-secret-subkeys --armor <key-id> > <key-id>.subpriv.asc
```

By default GnuPG writes to STDOUT if no file is specified with the
`--output <file>` option.
If no `<key-id>` has been entered, all present keys will be exported.

Keep your primary (private) key entirely offline! This is tricky to do but helps in protecting the very important primary key. If your primary key is stolen, the attacker can create new identities, revoke existing ones and completely impersonate you. Storing keys “offline” is therefore a good way to protect against such attacks.

Import backup of a private key:

```bash
$ gpg --allow-secret-key-import --import <key-id>.priv.asc
```

The public key can be freely distributed by sending it to friends, publishing on
websites or registering it with public [keyservers](keyservers).
In order to encrypt a documents for another user as well as to verify their
signatures, we need their public key.

Import someone else's public key:

```bash
$ gpg --import <key-id>.pub.asc
```

### Daily use

Let’s delete all private keys:

```bash
$ gpg --delete-secret-key <key-id>
```

Then, we import only the private keys of the subkeys:

```bash
$ gpg --import <key-id>.sub_priv.asc
```

Let’s check that we have only the private keys of the subkeys:

    sec#  rsa4096 2020-05-01 [C] [expires: 2021-05-01]
          F4123A7E12AFACB0CEBB584E9016AAAFCA7213FE
    uid           [ultimate] Mario Rossi (Hello World!) <mariorossi@gmail.com>
    ssb   rsa4096 2020-05-01 [E] [expires: 2021-05-01]
    ssb   rsa4096 2020-05-01 [S] [expires: 2021-05-01]
    ssb   rsa4096 2020-05-01 [A] [expires: 2021-05-01]

The small `#` after `sec` indicates that the secret key of the master key no longer exists, it’s a stub instead.

Your computer is now ready for normal use!

When you need to use the master keys, mount the encrypted USB drive,
and set the `GNUPGHOME` environment variable:

```bash
$ export GNUPGHOME=/media/something
$ gpg -K
```

or use the `--homedir` command-line argument:

```bash
$ gpg --homedir=/media/something -K
```

The latter command should now list your private key with `sec` and not
`sec#`.

### Set a default key

It is a good practice to configure this key as the default key within the `~/.bashrc` file, in order to specify its use as automatic with other applications that use the GnuPG system. To do this just insert the line in the `~/.bashrc` file:

```bash
export GPGKEY=<key-id>
```

**Note:** `<key-id>` is a primary key ID, not a subkey ID.

Now you need to restart the encryption service. Depending on your system, you may need to end one of the following two processes:

```bash
# gpg-agent:

$ killall -q gpg-agent
$ eval $(gpg-agent --daemon)

# seahorse-agent:

$ killall -q seahorse-agent
$ eval $(seahorse-agent --daemon)
```

Finally, run this command:

```bash
$ source ~/.bashrc
```

**Alternative:**

To choose a default key without having to specify `--default-key` on the command-line every time, create a configuration file (if it doesn't already exist), `~/.gnupg/gpg.conf`, and add a line containing

```bash
default-key <key-id>
```

Replacing `<key-id>` with the key ID you want to use by default.

### Renewal an expired key

The expiration date of a key can be changed any time, even after it expired:

```bash
$ gpg --edit-key <key-id>
# gpg> key 1 (only if you need to update a sub-key (e.g. the first one), by default primary key is selected)
# gpg> expire
# (follow prompts)
# gpg> save
```

Then you may send your key to the [keyservers](keyservers) to publish this change:

```bash
$ gpg --send-key <key-id>
```

**Note:** Private keys never expire, only public keys does.

You can always extend your expiration date, even after it has expired! This “expiration” is actually more of a safety valve or “dead-man switch” that will automatically trigger at some point. If you have access to the secret key material, you can untrigger it. The point is to setup something to disable your key in case you lose access to it (and have no revocation certificate).

Setting an expiration date means that you will need to extend that expiration date sometime in the future. That is a small task that you will need to remember to do (set a calendar event to remind you about your expiration date).

For subkeys, the effect is rather simple: after a given time frame, the subkey will expire. This expiry date can only be changed using the primary key. If an attacker gets hold of your subkey (and only this), it will automatically be inactivated after the expiry date.The expiry date of a subkey is a great tool to announce you switch your subkeys on a regular base, and that it's time for others to update your key after a given time.

For primary keys, the situation is different. If you have access to the private key, you can change the expiry date as you wish. This means, if an attacker gets access to your private key, he can extend the validity period arbitrarily. Worst case, you lose access to the private key at the same time, then even you cannot revoke the public key any more (you do have a printed or otherwise offline and safely stored revocation certificate, do you?). An expiry date might help in the case you just lose control over the key (while no attacker has control over it). The key will automatically expire after a given time, so there wouldn't be an unusable key with your name on it sitting forever on the keyservers.

### Replacement of a compromised key

When replacing one uncompromised key with a newer (typically longer) one, using a transition period when both keys are trustworthy and participate in the Web of Trust uses trust transitivity to use links to the old key to trust signatures and links created by the new key. During a transition, both keys are trustworthy but you only use the newer one to sign documents and certify links in the web of trust.

If you use smartcards (or plan to do so) then having more (encryption) keys creates a certain inconvenience (a card with the new key cannot decrypt old data encrypted with previous keys).

Adding new keys leads to an increase in the length of the public key.

### Add a photo

A photo can be added to the key:

```bash
$ gpg --edit-key <key-id>
# gpg> addphoto
# (follow prompt)
# gpg> save
```

The image must be a JPEG file. Remember that the image is stored within your public key. If you use a very large picture, your key will become very large as well. Keeping the image close to 240x288 is a good size to use.

### Add an additional UID

Additional email addresses can be added to the key:

```bash
$ gpg --edit-key <key-id>
# gpg> adduid
# (follow prompt).
# gpg> save
```

If more UIDs were added to a key, we can set a primary UID:

```bash
$ gpg --edit-key <key-id>
# gpg> uid 2
# gpg> primary
# (follow prompt)
# gpg> save
```

## Keyservers

Send public key to keyserver, so that others can retrieve the key:

```bash
$ gpg --send-keys <key-id> --keyserver <keyserver-name>
```

Find details about a key on the keyserver w/o importing it:

```bash
$ gpg --search-keys <key-id> --keyserver <keyserver-name>
```

Import key from a keyserver:

```bash
$ gpg --recv-keys <key-id>
```

The [sks keyservers pool][sks-pool] is often recommended. The communication with
the keyserver is established using a protocol called **hkps**.

Other keyservers:

-   pool.sks-keyservers.net
-   keys.openpgp.org (Default)
-   pgp.mit.edu
-   zimmermann.mayfirst.org
-   keyring.debian.org
-   keyserver.ubuntu.com

The `--keyserver` option is not required, when the keyserver is specified
in `~/.gnupg/dirmngr.conf`.

To fetch keys automatically from a keyserver as needed, add the following to
`~/.gnupg/gpg.conf`:

    keyserver-options auto-key-retrieve

More details on how to setup a keyserver, see [GPG Best
Practices][best-practices]. Note that since GnuPG 2.1 some options have been
moved to `dirmngr` and must be added to `~/.gnupg/dirmngr.conf`.

#### Tip: Ensure that all keys are refreshed through the keyserver you have selected

When creating a key, individuals may designate a specific keyserver to use to pull their keys from. It is recommended that you use the following option to `~/.gnupg/gpg.conf`, which will ignore such designations:

    keyserver-options no-honor-keyserver-url

This is useful because it prevents someone from designating an insecure method for pulling their key and if the server designated uses hkps, the refresh will fail because the ca-cert will not match, so the keys will never be refreshed. Note also that an attacker could designate a keyserver that they control to monitor when or from where you refresh their key.

#### Tip: Don't blindly trust keys from keyservers

Anyone can upload keys to keyservers and there is no reason that you should trust that any key you download actually belongs to the individual listed in the key. You should therefore verify with the individual owner the full key fingerprint of their key. You should do this verification in real life or over the phone.

Once you have verified the key fingerprint that you need, you may download the key from the keyserver pool:

```bash
$ gpg --recv-key '<fingerprint>'
```

The next step is to confirm that you actually got the correct key from the keyserver. The keyserver might have given you a different key than the one you just asked for. If you have GnuPG with version less than 2.1, then you must manually confirm the fingerprint after you have downloaded the key (versions 2.1 and later will refuse to accept incorrect keys from the keyserver).

You can confirm the key fingerprint in one of two ways:

Option 1: Check the fingerprint is now in your keyring:

```bash
$ gpg --fingerprint '<fingerprint>'
```

Option 2: Attempt to (locally) sign a key with that fingerprint:

```bash
$ gpg --lsign-key '<fingerprint>'
```

If you are confident you have the right fingerprint from the owner of the key, the preferred method is to locally sign the key. If you want to publicly advertise your connection to the person who owns the key, you can do a publicly exportable `--sign-key` instead.

Note the single quote marks above (’), which should surround your full fingerprint and are necessary to make this command work. Double-quotes (") also work.

#### Tip: Don't rely on the key ID

Short OpenPGP key IDs, for example `0×2861A790`, are 32 bits long. They have been shown to be easily spoofed by another key with the same key ID. Long OpenPGP key IDs (for example `0xA1E6148633874A3D`) are 64 bits long. They are trivially collidable, which is also a potentially serious problem.

If you want to deal with a cryptographically-strong identifier for a key, you should use the full fingerprint. You should never rely on the short, or even long, key ID.

You should probably at least set keyid-format `0xlong` and with-fingerprint gpg options (put them in `~/.gnupg/gpg.conf`) to increase the key ID display size to 64 bit under regular use, and to always display the fingerprint.

 You can always find your primary key’s fingerprint (for example, if you want to give your fingerprint to someone to verify at a keysigning party), you can display the fingerprints of all of your secret keys by running this:

```bash
$ gpg --with-fingerprint --list-secret-key
```

Check key fingerprints before importing.

If you received or downloaded a key you can and should display its fingerprint before importing it into your keyring, in that way you can verify the fingerprint without possibly spoiling your keyring and adding a compromised key:

```bash
$ gpg --with-fingerprint <key-file>
```

## Encryption

**GnuPG** supports both symmetric key encryption and public key encryption:

1.  **Symmetric key encryption:** The same key is used for both encryption and
    decryption. Two parties communicating using a symmetric cipher must agree on
    the key beforehand. Once they agree, the sender encrypts a document using the
    key, sends it to the receiver, and the receiver decrypts it using the same
    key. The primary problem with symmetric key encryption is the key exchange.
    If there are `n` people who need to communicate, then `n(n-1)/2` keys are
    needed for each pair of people to communicate privately.

2.  **Public key encryption:** This involves creation of a public and private key
    pair. The private key should never be shared with anyone, while the public
    key is supposed to be shared with people who want to send you encrypted data.
    Documents are encrypted using the public key. Later the encrypted file is
    decrypted with the private key and a passphrase that was set during key
    generation. As opposed to symmetric key encryption, only `n` keypairs are
    needed for `n` people to communicate privately.

For a more thorough discussion see for instance [The GNU Privacy
Handbook][gnu-handbook].

### Public key encryption

After a public key has been imported, we can encrypt a file or message to that
recipient:

```bash
$ gpg [--output <out-file>] --recipient <key-id> --encrypt <file>
```

By default the encrypted file will be appended a `.gpg` suffix. This can be
changed with the `--output <out-file>` option.

To encrypt a file for personal use, `<key-id>` is simply the name or email
address (or anything else) that was used during key generation.

When encrypting or decrypting a document, it is possible to have more than one
private key in use. In this case, we need to select the active key with the
option `--local-user <key-id>`. Otherwise the default key is used.

If the recipient is not specified, it will self-encrypt using its public key. If you choose a recipient indicating someone else's public key, you will not be able to decrypt it but only the destinatary with his private key can.

Decrypt a file that has been encrypted with our own public key:

```bash
$ gpg [--output <out-file>] --decrypt <file>.gpg
```

Further options:

-   `--armor, -a`: encrypt file using ASCII text.
-   `--recipient-file <key-file>`: using a public key from a file.
-   `--hidden-recipient <user-id>, -R <user-id>`: put recipient key IDs in the
      encrypted message to hide receivers of message against traffic analysis.
-   `--no-emit-version`: avoid printing version number in ASCII armored output.

**Note:** Each private subkey is independent of the others, each one decrypts the messages addressed to it. It is therefore better to have only one subkey per capability at a time.

### Signing and checking signatures

To avoid the risk that someone else claims to be you, it's useful to sign every
document that is encrypted. If the encrypted document is modified in any way, a
verification of the signature will fail.

Sign a file with your own key:

```bash
$ gpg --output <file.sig> --sign <file>
```

Note that the file is compressed before being signed. The output will be in
binary format and thus won't be _human-readable_. To make a clear text
signature, run:

```bash
$ gpg --output <file.sig> --clearsign <file>
```

This causes the document to be wrapped in an ASCII-armored signature but
otherwise does not modify the document. Hence, the content in a clear text
signature is readable w/o any special software. OpenPGP software is only
required to verify the signature.

A disadvantage of above methods is that users who received the signed documents
must edit the files to recover the original (since signature is now part of
document). It is also possible to make a detached signature to a file:

```bash
$ gpg --armor --output <file.sig> --detach-sign <file>
```

This is highly recommended when signing binary files (like tar archives).

Sign and encrypt a file:

```bash
$ gpg --recipient <user-id> [--local-user <key-id>] [--armor] --sign --encrypt <file>
```

**Note:** GnuPG first signs a message, then encrypts it.

String `--local-user <key-id>` specifies the key to sign with, it overrides
`--default-key <key-id>` option, which is usually specified in
`~/.gnupg/gpg.conf`.

When an encrypted file has been signed, the signature is usually checked when
the file is decrypted using the `--decrypt` option:

```bash
$ gpg --output <file> --decrypt <file.gpg>
```

To just check the signature use the `--verify` option:

```bash
$ gpg --verify <pgp-file/sig-file>
```

This assumes the signer's public key has already been imported.

Assuming we downloaded a file `archive.tar.gz` and a corresponding detached
signature `archive.tar.gz.asc`, we can verify the signature after downloading
the signer's public key as follows:

```bash
$ gpg --verify archive.tar.gz.asc archive.tar.gz
```

### Symmetric key encryption

Documents can be encrypted with a symmetric cipher using a passphrase. The
default cipher used is AES-128 but can be changed with the `--cipher-algo`
option.

Encrypt a file using a symmetric key:

```bash
$ gpg --symmetric <file>
```

This will create an encrypted file with a .gpg appended to the old file name.

Decrypt the encrypted file:

```bash
$ gpg [--output myfile] --decrypt myfile.gpg
```

The user will be prompted to enter the passphrase used to encrypt.

## GPG configuration files

The home directory where **GnuPG** and its helper tools look for configuration
files defaults to `~/.gnupg/` (see also `--homedir` option). By default the
directory has its permissions set to `700`, and the files it contains have their
permissions set to `600`. It is very important to keep `~/.gnupg/` private and
have a secure backup stored on a seperate disk as it contains all files
generated by the `gpg` command including the private keys.

For most users the following files are sufficient:

-   `gpg.conf`: standard configuration file read by `gpg` on startup. It may
    contain any long options that are available for `gpg`. A skeleton
    configuration file is generated on the very first run of `gpg`.
-   `gpg-agent.conf`: standard configuration file read by `gpg-agent` on startup.
    The `gpg-agent` is a daemon to request and cache passphrases used by `gpg`.
-   `dirmngr.conf`: standard configuration file read by `dirmngr` on startup.
    `dirmngr` takes care of accessing the OpenPGP keyservers and is also used for
    managing and downloading certificate revocation lists. A skeleton
    configuration file is generated on the very first run of `gpg`.

Example configuration files are included in this repository.

## References

-   [GPG Guide (GitHub)][gpg-guide]
-   [The GNU Privacy Handbook][gnu-handbook]
-   [Debian subkeys][debian-subkeys]
-   [OpenPGP -The almost perfect keypair][eleven-labs]
-   [OpenPGP best practices][best-practices]

[gpg-guide]: https://github.com/bfrg/gpg-guide

[gnu-handbook]: https://www.gnupg.org/gph/en/manual.html

[debian-subkeys]: https://wiki.debian.org/Subkeys

[eleven-labs]: https://blog.eleven-labs.com/en/openpgp-almost-perfect-key-pair-part-1/

[best-practices]: https://riseup.net/en/gpg-best-practices

[sks-pool]: https://sks-keyservers.net/overview-of-pools.php#pool_hkps

TODO:

-   TODO, Link alle section
-   Reogranize (See gpg-guide vero)
-   GitHub Key
-   Sito (My old key # is lost!, My fingerprint is #, Every year at 01/05 my publi key change! (Expiration of subkeys modify my pub))
-   Controllare tutti le cartelle e i file delle config

You're trying to delete a user ID, not a subkey. Use key [n] and delkey instead. From the help comand inside gpg --edit-key:

uid         select user ID N
key         select subkey N
deluid      delete selected user IDs
delkey      delete selected subkeys

If you already shared your key with others, better revoke the key instead of deleting it. By deleting it, other's will not be able to realize you're not using it any more (you can't delete it on key servers and other's computers!), by revocation you're signalling "don't use this (sub)key any more".

Private don't change. Master pub changed if i change expiration date or i add or do somtihign to subkeys.

Pub si allunga solo per l'aggiunta di nuove chiavi/sottochiavi non della data di scadenza

Se cifro un messaggio con una subkey mi servirà la chiave privata della subkeys per decifrarlo.
La master privata serve solo per gestire le subkes non per decifrare.
Ogni chiave decifra ciò che ha criptato sè stessa.

Se ho due sub E come selezioni quella di default?
gpg --default-key <yourKeyID> --sign-key <YourFriendsKeyID>

Per vedere gli ID delle subkey bisogna nadare in edit

These options are used to change the configuration and are usually found in the option file.
\--default-key name
    Use name as the default key to sign with. If this option is not used, the default key is the first key found in the secret keyring. Note that -u or --local-user overrides this option. This option may be given multiple times. In this case, the last key for which a secret key is available is used. If there is no secret key available for any of the specified values, GnuPG will not emit an error message but continue as if this option wasn’t given.

    By default, GnuPG will use the most recently created

Regarding how keys are selected with GnuPG: simply import the keys (gpg --import), GnuPG will select the proper key automatically. Usually, the required key is stored in the crypto message's headers, otherwise GnuPG will try all of the private keys.

Quindi quando qualcuno usa la mia pub per criptare un file e inviarmelo di default usa la  pubblica della mia sottochiave E più recente. Io con il comando decrypt delego a GPG di trovare la privata (sarà quella E recente) per decifrare.
