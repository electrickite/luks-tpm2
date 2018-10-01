LUKS TPM2
=========

A small utility script to manage LUKS keyfiles sealed by a TPM 2.0.

This script assumes you will be using a sealed keyfile or a key stored in the
TPM during boot to unlock the root file system. It is intended to be used as
part of your kernel update process to generate a key sealed against the new
kernel's PCR values.

Requirements
------------

This script requires:

  * [bash](https://www.gnu.org/software/bash/)
  * [tpm2-tools](https://github.com/tpm2-software/tpm2-tools) v3.1
  * [cryptsetup](https://gitlab.com/cryptsetup/cryptsetup)
  * A TPM 2.0 or TPM 2.0 simulator

Update Process
--------------

The script facilitates the following kernel update process:

  1. Kernel is updated
  2. `luks-tpm2 temp` is called, either manually or via an update hook, and sets
     a temporary LUKS passphrase
  3. The system is rebooted into the new kernel
  4. Because the TPM PCRs have changed, the old keyfile cannot be unsealed
  5. User enters the temporary passphrase to unlock the disk
  6. `luks-tpm2 reset` is called, generating a new keyfile sealed by the TPM and
     removing the temporary passphrase

### LUKS Key Slots

The script requires two LUKS key slots to function: one for the sealed keyfile
and one for the temporary passphrase. You are also *strongly* encouraged to
dedicate an additional slot for a recovery passphrase not managed by `luks-tpm2`.

The default key slot layout is:

  * Slot 0: Recovery passphrase (optional)
  * Slot 1: TPM keyfile
  * Slot 2: Temporary passphrase

### Replace Key

The `replace` action allows a TPM-sealed LUKS key to be replaced (overwritten)
by a new, randomly generated key. By default, LUKS slot 1 will be replaced.
This action will not prompt for a passphrase, so the current key must be both
"unsealable" by the TPM and a valid LUKS key.

Usage
-----

    luks-tpm2 [OPTION]... DEVICE ACTION

### Actions

  * `init`: Initialize the LUKS TPM key slot
  * `temp`: Set a temporary LUKS passphrase
  * `reset`: Regenerate the LUKS TPM key and remove the temporary passphrase
  * `replace`: Replace (overwrite) a LUKS TPM key

### Options

    -h         Print help
    -m PATH    Mount point for the tmpfs file system used to store TPM keyfiles
               Default: /root/keyfs
    -p PATH    Sealed keyfile path
               .priv will be added to the path for the private section and .pub
               will be added for the public portion
               Default: /boot/keyfile  (/boot/keyfile.priv /boot/keyfile.pub)
    -H HEX     The TPM handle of the parent object for the sealed key
               Default: 0x81000001
    -x HEX     Index of the TPM NVRAM area holding the key
    -s NUMBER  Key size in byes
               Default: 32
    -t NUMBER  LUKS slot number for the TPM key
               Default: 1
    -r NUMBER  LUKS slot number for temporary reset passphrase
               Default: 2
    -L STRING  List of PCR banks used to seal LUKS key
               Default: sha1:0,2,4,7
    -T STRING  TCTI module used to communicate with the TPM
               Default: device:/dev/tpmrm0

### Default configuration

The script will read default configuration values by sourcing
`/etc/default/luks-tpm2` if it exists. The location of this file can changed by
setting the `CONFFILE` environment variable. Variables read from the config
file will override hard coded defaults, but will not override command line
arguments.

How-to
------

`luks-tpm2` can protect LUKS keys using the TPM in one of two ways:

  * On disk as a pair of "sealed" files that can only be decrypted by the TPM
  * In TPM non-volatile memory (NVRAM)

In either case, the data is only accessible when certain Platform Configuration
Registers (PCRs) have not changed. This indicates that the system has not been
altered since the data was sealed.

Note that all TPM objects will be created in the owner hierarchy.

## On-disk

Before storing sealed key files on disk, you must create a parent encryption key
on the TPM. In this example, we create a primary RSA key in the owner hierarchy
and make it persistent at handle `0x81000001`:

    $ sudo tpm2_listpersistent -T device:/dev/tpmrm0
    $ sudo tpm2_createprimary -H o -g sha1 -G rsa -T device:/dev/tpmrm0
    $ sudo tpm2_evictcontrol -A o -H 0x80000000 -S 0x81000001 -T device:/dev/tpmrm0

Next, call `luks-tpm2` with appropriate options:

    $ sudo luks-tpm2 -p /boot/keyfile -H 0x81000001 /dev/sdaX init

Two sealed files will be generated (in `/boot` for this example):
`/boot/keyfile.priv` and `/boot/keyfile.pub`.

## NVRAM

Most TPMs provide a small amount of user-configurable non-volatile memory
(NVRAM) that will perisist between reboots. Note that NVRAM often has a limited
number of writes, so it may not be a good option if frequent updates are
required.

Before initializing NVRAM storage, locate a free index:

    $ sudo tpm2_nvlist -T device:/dev/tpmrm0

And then call `luks-tpm2` with appropriate options:

    $ sudo luks-tpm2 -x 0x1500001 /dev/sdaX init

License and Copyright
---------------------

Copyright 2018 Corey Hinshaw <coreyhinshaw@gmail.com>

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.
