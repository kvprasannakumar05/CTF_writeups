# lost+nfound Writeup

I approached this as a disk forensics challenge rather than something I needed to fully boot and interact with. The description said the user installed their favorite package and "everything turned into nonsense," and the hint explicitly said that `history` does not print the history file. That immediately suggested two things:

1. The useful artifacts were probably still on disk.
2. I should inspect the VM image offline and recover the real shell history file directly.

## Initial triage

The provided asset was a single OVA:

```bash
ls -la
rg --files .
tar -tf ctf-vm.ova
```

That showed:

- `ctf-vm.ovf`
- `ctf-vm.mf`
- `ctf-vm-file1.iso.gz`
- `ctf-vm-disk1.vmdk.gz`

So the useful part was the compressed VMDK inside the OVA. I extracted and decompressed it, then converted it to raw so I could use standard forensic tools:

```bash
mkdir -p /tmp/lostnfount
tar -xf ctf-vm.ova -C /tmp/lostnfount ctf-vm-disk1.vmdk.gz ctf-vm-file1.iso.gz
gunzip -kf /tmp/lostnfount/ctf-vm-disk1.vmdk.gz
qemu-img convert -O raw /tmp/lostnfount/ctf-vm-disk1.vmdk /tmp/lostnfount/ctf-vm-disk1.raw
```

Then I identified the partitions:

```bash
mmls /tmp/lostnfount/ctf-vm-disk1.raw
fdisk -l /tmp/lostnfount/ctf-vm-disk1.raw
```

The partition table showed three partitions:

- partition 1: small Linux boot partition
- partition 2: swap
- partition 3: main Linux filesystem

The root filesystem started at sector `3430400`.

## Finding the real shell history

Since the hint warned me not to trust the `history` builtin, I searched the root filesystem directly for shell history artifacts:

```bash
fls -pr -o 3430400 /tmp/lostnfount/ctf-vm-disk1.raw 2>/dev/null | rg 'history|root/|home/'
```

That immediately found:

```text
root/.ash_history
```

So the VM was using BusyBox `ash`, not bash. I dumped that file with `icat`:

```bash
icat -o 3430400 /tmp/lostnfount/ctf-vm-disk1.raw 131111
```

The important commands in the history were:

```text
apk add rust
apk add cargo
cargo install xor
export PATH=$PATH:/root/.cargo/bin
xor --help
cd home/
git init .
git add .
git commit -m "a bunch of nonsense"
git reflog
git log
ls 5457501C/
history
cat $HISFILE
vi .ash_history
```

This was the first big pivot point. From those commands I learned:

- the user installed a Rust tool called `xor`
- they ran it on the filesystem in some way
- they initialized a Git repository in `/home`
- they committed the modified state
- they later used `git stash`
- one visible directory name was `5457501C`, which looked like hex, not random text

At that point I stopped thinking of the nonsense filenames as corruption and started treating them as encoded data.

## Inspecting `/home`

I recursively listed the `/home` directory from the disk image:

```bash
fls -p -o 3430400 /tmp/lostnfount/ctf-vm-disk1.raw 131081
fls -pr -o 3430400 /tmp/lostnfount/ctf-vm-disk1.raw 131081
```

Everything in `/home` had turned into uppercase hex strings. One file appeared over and over with the same encoded name:

```text
08555D451D131A075A5D0E
```

That matched the history, where the user had created a `red-herring` file in every directory:

```text
echo "ajfesidpiunvzcoixuiuwjenfksdlzxjol" > */red-herring
echo "ajfesidpiunvzcoixuiuwjenfksdlzxjol" > ./*/red-herring
for f in $(find . -type d); do echo "kajdsfojczvioxjoij3" >> $f/red-herring; done
```

So I assumed:

- ciphertext filename `08555D451D131A075A5D0E`
- plaintext filename `red-herring`

If the installed `xor` tool literally XORed filenames, I could recover the keystream:

```python
ct = bytes.fromhex("08555D451D131A075A5D0E")
pt = b"red-herring"
key = bytes(a ^ b for a, b in zip(ct, pt))
```

That produced:

```text
z09huvhu33i
```

I tested that key on the known directory `5457501C` and got:

```text
.git
```

That was the second big pivot point. The nonsense directory was the Git repository metadata. That meant I did not need to recover the whole filesystem naming scheme perfectly; I mainly needed to recover enough of `.git` to reconstruct the commit history and any hidden state.

## Recovering the filename XOR keystream

I used standard `.git` filenames as known plaintext to extend the keystream. For example:

- `HEAD`
- `config`
- `description`
- `hooks/pre-commit.sample`
- `refs/heads/master`
- `logs/HEAD`
- `COMMIT_EDITMSG`

Because the encrypted names were stored as hex and the plaintext Git paths are highly predictable after `git init`, I could line up many ciphertext/plaintext pairs and extend the keystream well beyond the first 11 bytes.

That let me reliably decode the structure of `.git`, including:

- `.git/HEAD`
- `.git/config`
- `.git/index`
- `.git/COMMIT_EDITMSG`
- `.git/refs/heads/master`
- `.git/logs/HEAD`
- `.git/logs/refs/heads/master`
- `.git/objects/...`

## Recovering the content XOR keystream

The filenames were only half the problem. File contents in `.git` were also XORed.

I used a clean local `git init` as a plaintext oracle:

```bash
rm -rf /tmp/lostnfount-gitprobe
mkdir /tmp/lostnfount-gitprobe
git init /tmp/lostnfount-gitprobe
find /tmp/lostnfount-gitprobe/.git -maxdepth 3 -type f | sort
```

The stock hook samples created by `git init` are deterministic enough to use as known plaintext. I matched encrypted files from the VM against the corresponding plaintext files from my local probe repo. For example:

- `hooks/pre-rebase.sample`
- `hooks/pre-commit.sample`
- `hooks/prepare-commit-msg.sample`
- `config`
- `HEAD`
- `info/exclude`

When I XORed ciphertext with the known plaintext of one of those hook files, I got a long reusable keystream. The same keystream decrypted all the other standard Git files cleanly.

That let me read the live refs and reflogs.

## Reading the Git refs and reflogs

After decrypting `.git/refs/heads/master`, I got the current commit hash:

```text
8fadc480a8f31d2dac1099cfbb03a8bfb5de569f
```

Then I decrypted `.git/logs/HEAD` and `.git/logs/refs/heads/master`. The reflog said:

```text
0000000000000000000000000000000000000000 8fadc480a8f31d2dac1099cfbb03a8bfb5de569f root <root@localhost.localdomain> 1774732050 +0000    commit (initial): a bunch of nonsense
8fadc480a8f31d2dac1099cfbb03a8bfb5de569f 8fadc480a8f31d2dac1099cfbb03a8bfb5de569f git stash <git@stash> 1774732415 +0000    reset: moving to HEAD
```

This told me:

- there was one normal commit: `a bunch of nonsense`
- the user had also used `git stash`

The challenge was almost certainly hiding the flag in the stash rather than the visible committed tree.

## Decrypting Git objects

At this point I walked `.git/objects`, decrypted each object file with the recovered content keystream, zlib-decompressed it, and computed its SHA-1 from the decrypted object bytes. That gave me the real object identities and contents.

The important objects were:

- commit `8fadc480a8f31d2dac1099cfbb03a8bfb5de569f`
- commit `6098e011c4761ed32510191498fb273670bfd7b5`
- stash commit `55a10e0874b6d37a8b9c2d70468d91f5b8c78cf5`

The normal commit looked like this:

```text
tree 1421ec4a2c9b020f5ee3f910a4626810578fe0ee
author root <root@localhost.localdomain> 1774732050 +0000
committer root <root@localhost.localdomain> 1774732050 +0000

a bunch of nonsense
```

The tree mostly contained directories full of `red-herring` files and one extra blob:

```text
nonsuspiciousatall.txt -> "hmmm\n"
red-herring -> "kajdsfojczvioxjoij3\n"
```

That was all bait.

Then I decrypted the stash commit, and that was the win:

```text
tree 6764da9116f608795b2a2e27695ec314b7e9e9ac
parent 8fadc480a8f31d2dac1099cfbb03a8bfb5de569f
parent 6098e011c4761ed32510191498fb273670bfd7b5
author git stash <git@stash> 1774732415 +0000
committer git stash <git@stash> 1774732415 +0000

On master: You found me! UMASS{h3r35_7h3_c4rg0_vr00m}
```

So the flag was stored directly in the stash commit message.

## Flag

```text
UMASS{h3r35_7h3_c4rg0_vr00m}
```

## Final notes

The two hints were exactly on point:

- `history` was misleading because the real shell was `ash`, so the artifact I needed was `/root/.ash_history`
- the "nonsense" was not random corruption; it was a reversible XOR transformation applied to filenames, file contents, and Git metadata

The cleanest solving path was:

1. Extract the VM disk offline.
2. Read `/root/.ash_history`.
3. Notice `cargo install xor`, `git init`, and later `git stash`.
4. Use the known filename `red-herring` to recover the XOR keystream.
5. Decrypt `.git`.
6. Read the stash commit message.

That gave the flag without ever needing to boot the VM.
