# Trying out Agebox

## Lab notes

### Initializing the git repository

```
mkdir ~/Code/trying-agebox
cd ~/Code/trying-agebox

git init
git commit --allow-empty -m "initial commit"

# Created A GitHub repository

git remote add origin git@github.com:jpcenteno/trying-agebox.git
git branch -M main
git push -u origin main

echo Some plaintext 1 > note-1.txt
echo Some plaintext 2 > note-2.txt

git add note-1.txt note-2.txt
git commit -m 'Add two plaintext files'
git push
```

### Initializing Agebox

I installed Agebox, then initialized it on the repository issuing:

```sh
agebox init
```

This created a file named `.ageboxreg.yml` with content:

```yaml
file_ids: []
version: "1"
```

Then I commited it to the git repository:

```sh
git add .ageboxreg.yml
git commit -m 'chore: Initialize agebox'
git push
```

### Tracking and encrypting the text files

My goal here is to encrypt all the `.txt` files in the repository.

The first thing that I tried was to use `agebox encrypt --filter *.txt`, but it
ggreeted me with the following error:

```
$ agebox encrypt --filter *.txt

error: "encrypt" command failed: could not encrypt: could not get public keys:
lstat keys: no such file or directory
```

I thought that it would use my SSH keypair right away. Let's see how to set the
keys to use.

The readme states:

> Public keys should be on a directory relative to the root of the repository
> (by default ./keys) at the moment of invoking encryption commands, this
> simplifies the usage of keys by not requiring pgp keys or agents.

It does not detail on the key format, but I could add a keypair anyway.

```
mkdir keys
cp ~/.ssh/id_ed25519.pub keys/

git add keys
git commit -m 'chore: Add my public key to the keys/ directory'
```

I retried the `agebox encrypt` command and this time it worked:

```
$ agebox encrypt . --filter '.*.txt'

INFO[0000] Loaded public keys    keys=1 svc=storage.fs.KeyRepository version=0.7.2
INFO[0000] Secret encrypted      secret-id=note-1.txt svc=box.encrypt.Service version=0.7.2
INFO[0000] Secret encrypted      secret-id=note-2.txt svc=box.encrypt.Service version=0.7.2
```

Issuing this command deleted both `.txt` files and created two new files sharing
the same name holding the encrypted versions of them:

```
$ git status --porcelain

 M .ageboxreg.yml
 D note-1.txt
 D note-2.txt
?? note-1.txt.agebox
?? note-2.txt.agebox
```

It also modified `.ageboxreg.yml`, adding the filenames of both .txt files to
the `file_ids` field:

```diff
-file_ids: []
+file_ids:
+- note-1.txt
+- note-2.txt
 version: "1"
```

I made two separate commits, one for the changes made to `.ageboxreg.yml` and
one that replaces the plaintext files with their encrypted counterparts.

```
git add .ageboxreg.yml
git commit -m 'chore: Track the existing .txt files using Agebox'
```

```
git add .
g c -m 'chore: Encrypt both txt files'
```

And pushed the changes:

```sh
git push
```

Now my repository looks like this, both locally and remotely on GitHub:

```
.
├── keys
│   └── id_ed25519.pub
├── note-1.txt.agebox
└── note-2.txt.agebox
```

### Decrypting the files

The private counterpart to the SSH key that I used is stored using SSH agent and
I want to keep it that way. Sadly, today I learned that ssh-agent is a signing
tool and it does not support encryption and decryption. I think that this is as
far as I will go with the experiment.

## Conclusions

Agebox does not provide _transparent_ encryption, it replaces existing plaintext
with their encrypted counterparts. It looks great for versioning a few
`secrets.env` (or similar) files and managing who has access to their content.
It does not look like a solution that scales for repositories that have several
end-to-end encrypted files.

The tool works best when scripted as part of a CI / CD pipeline.

As I expected, it does not hide the names of the files, their sizes or when they
were modified.

File integrity is out of the Agebox scope. A workaround would be to verify Git commit
sign verification. Otherwise an attacker could replace an encrypted file with
another just by knowing the victims public keys.
