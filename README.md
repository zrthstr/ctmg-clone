## ctmg-clone

`ctmg` clone with added container `resize` & `read-only/read-write` functionality

`ctmg` is an encrypted container manager for Linux using `cryptsetup` and various standard file system utilities. Containers have the extension `.ct` and are mounted at a directory of the same name, but without the extension. Very simple to understand, and very simple to implement; `ctmg` is a simple bash script.

About: https://git.zx2c4.com/ctmg/about/

### Usage

    Usage: ctmg [ new | delete | open | close | list ] [arguments...]
      ctmg new     container_path        container_size[units_suffix]
      ctmg resize  container_path   plus_container_size[units_suffix]
      ctmg delete  container_path
      ctmg open    container_path                       # read-only
      ctmg rwopen  container_path                       # read-write
      ctmg close   container_path
      ctmg list

Calling `ctmg` with no arguments will call `list` if there are any containers open, and otherwise show the usage screen. Calling `ctmg` with a filename argument will call `open` if it is not already open and otherwise will call `close`.

### Examples

#### Create a 100MiB encrypted container called "example"

    zx2c4@thinkpad ~ $ ctmg new example 100MiB
    [#] truncate -s 100MiB /home/zx2c4/example.ct
    [#] cryptsetup --cipher aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 5000 --batch-mode luksFormat /home/zx2c4/example.ct
    Enter passphrase:
    [#] chown 1000:1000 /home/zx2c4/example.ct
    [#] cryptsetup luksOpen /home/zx2c4/example.ct ct_example
    Enter passphrase for /home/zx2c4/example.ct:
    [#] mkfs.ext4 -q -E root_owner=1000:1000 /dev/mapper/ct_example
    [+] Created new encrypted container at /home/zx2c4/example.ct
    [#] cryptsetup luksClose ct_example

#### Open a container, add a file, and then close it

    zx2c4@thinkpad ~ $ ctmg open example
    [#] # or
    zx2c4@thinkpad ~ $ ctmg rwopen example
    [#] cryptsetup luksOpen /home/zx2c4/example.ct ct_example
    Enter passphrase for /home/zx2c4/example.ct:
    [#] mkdir -p /home/zx2c4/example
    [#] mount /dev/mapper/ct_example /home/zx2c4/example
    [+] Opened /home/zx2c4/example.ct at /home/zx2c4/example
    zx2c4@thinkpad ~ $ echo "super secret" > example/mysecretfile.txt
    zx2c4@thinkpad ~ $ ctmg close example
    [#] umount /home/zx2c4/example
    [#] cryptsetup luksClose ct_example
    [#] rmdir /home/zx2c4/example
    [+] Closed /home/zx2c4/example.ct

#### Resize a container by 654MiB

    foo@thinkpad ~/test/ctmg-clone (git)-[master] % ./ctmg.sh resize  e3 654MiB
    [#] truncate -s +654MiB /home/foo/test/ctmg-clone/e3.ct
    [#] cryptsetup luksOpen /home/foo/test/ctmg-clone/e3.ct ct_home-foo-test-ctmg-clone-e3
    Enter passphrase for /home/foo/test/ctmg-clone/e3.ct:
    [+] e2fsck 1.45.5 (07-Jan-2020)
    [+] Pass 1: Checking inodes, blocks, and sizes
    [+] Pass 2: Checking directory structure
    [+] Pass 3: Checking directory connectivity
    [+] Pass 4: Checking reference counts
    [+] Pass 5: Checking group summary information
    [+] /dev/mapper/ct_home-foo-test-ctmg-clone-e3: 11/27216 files (0.0% non-contiguous), 9089/108544 blocks
    [#] cryptsetup resize ct_home-foo-test-ctmg-clone-e3
    Enter passphrase for /home/foo/test/ctmg-clone/e3.ct:
    [#] resize2fs /dev/mapper/ct_home-foo-test-ctmg-clone-e3
    [+] resize2fs 1.45.5 (07-Jan-2020)
    [+] Resizing the filesystem on /dev/mapper/ct_home-zrth-test-ctmg-clone-e3 to 778240 (1k) blocks.
    [+] The filesystem on /dev/mapper/ct_home-foo-test-ctmg-clone-e3 is now 778240 (1k) blocks long.
    [+] Resized encrypted container /home/foo/test/ctmg-clone/e3.ct
    [#] cryptsetup luksClose ct_home-foo-test-ctmg-clone-e3
    [+] Closed /home/foo/test/ctmg-clone/e3.ct

### Installation

    # make install

Or, use the package from your distribution:

#### Gentoo

    # emerge ctmg

### Bug reports

All bugs in this repo are most likely from ctmg-clone not from ctmg - from adding the `resize` and `read/write` mode.
Report any bugs about the `read/write` or `resize` here via github issue or PR.

The original Authors readme says: Report any bugs to <jason@zx2c4.com>

### Security Considerations

This runs as root and auto-`sudo`s itself to achieve that. As such, you shouldn't run this on paths you don't trust or paths that could be controlled by malicious users.

Since `ctmg` uses `cryptsetup` and the LUKS infrastructure, it uses the Linux block device encryption APIs. The state of the art in block device encryption, as of writing, is [XTS mode](http://csrc.nist.gov/publications/nistpubs/800-38E/nist-sp-800-38E.pdf), which is what `ctmg` uses. But do note that this does not guarantee, entirely, the integrity of data, just the secrecy. As such, if a malicious user is able to modify the encrypted content, it is possible this could result in differing decrypted content without you noticing. So, `ctmg` is useful for keeping things secret, but not for guaranteeing the authenticity of the data. If your laptop gets stolen, sleep safely knowing that your `ctmg`-secured data is safe, but if an attacker is actively modifying the `.ct` file while you're using it in one way or another, you've got trouble.

In order to conserve space, `ctmg` uses `truncate` to make sparse files. This means that the file grows as it's used. An attacker can therefore see how much of the container is utilized. If you care about this, it's easy enough to replace the single call to `truncate` with a single call to `dd if=/dev/urandom` to make a completely full file containing only random data.
