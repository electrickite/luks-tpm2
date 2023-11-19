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
  * [tpm2-tools](https://github.com/tpm2-software/tpm2-tools) v4
  * [cryptsetup](https://gitlab.com/cryptsetup/cryptsetup)
  * A TPM 2.0 or TPM 2.0 simulator

Update Process
--------------

The script facilitates a variety of kernel update flows. For example, you could
set a temporary passphrase interactively during the update:

  1. Kernel is updated
  2. `luks-tpm2 temp` is called, either manually or via an update hook, and sets
     a temporary LUKS passphrase
  3. The system is rebooted into the new kernel
  4. Because the TPM PCRs have changed, the old keyfile cannot be unsealed
  5. User enters the temporary passphrase to unlock the disk
  6. `luks-tpm2 reset` is called, generating a new keyfile sealed by the TPM and
     removing the temporary passphrase

Alternately, the PCR values of the new kernel can be computed in advance using
and external command. The example below uses [tpm_futurepcr](https://github.com/grawity/tpm_futurepcr):

  1. Kernel is updated
  2. `luks-tpm2 -c "tpm_futurepcr -L '::pcr::' -o '::output::'" compute` is
      called, either manually or via an update hook, which pre-computes the new
      kernel PCR values and replaces the existing TPM key with a new, random
      value
  3. The system is rebooted into the new kernel
  4. Because the pre-computed PCR values match the new kernel, unsealing the
     key succeeds
  5. The unsealed key is used to unlock the disk normally

### LUKS Key Slots

The script requires two LUKS key slots to function: one for the sealed keyfile
and one for the temporary passphrase. You are also *strongly* encouraged to
dedicate an additional slot for a recovery passphrase not managed by `luks-tpm2`.

The default key slot layout is:

  * Slot 0: Recovery passphrase (optional)
  * Slot 1: TPM keyfile
  * Slot 2: Temporary passphrase

Usage
-----

    luks-tpm2 [OPTION]... [DEVICE] ACTION

### Options

    -h         Print help
    -v         Print version
    -m PATH    Mount point for the tmpfs file system used to store TPM keyfiles
               Default: /root/keyfs
    -p PATH    Sealed keyfile path
               .priv will be added to the path for the private section and .pub
               will be added for the public portion
               Default: /boot/keyfile  (/boot/keyfile.priv /boot/keyfile.pub)
    -H HEX     The TPM handle of the parent object for the sealed key
               Default: 0x81000001
    -K         Prompt for parent key password
    -k PATH    Path to a file containing the parent key password
    -x HEX     Index of the TPM NVRAM area holding the key
    -s NUMBER  Key size in byes
               Default: 32
    -f PATH    LUKS keyfile used instead of prompting for LUKS passphraes
    -o PATH    Optionally write the new LUKS keyfile sealed in the tpm to this path
    -t NUMBER  LUKS slot number for the TPM key
               Default: 1
    -r NUMBER  LUKS slot number for temporary reset passphrase
               Default: 2
    -L STRING  List of PCR banks used to seal LUKS key
               Default: sha256:0,2,4,7
    -l STRING  List of PCR banks used to unseal LUKS key
               Default: <value of -L>
    -T STRING  TCTI module used to communicate with the TPM
               Default: device:/dev/tpmrm0
    -c COMMAND The command used to precompute PCR values
               Ex: tpm_futurepcr -L '::pcr::' -o '::output::'

### Actions

#### init

Initialize the LUKS TPM key slot, by default in LUKS slot 1. This action will
prompt for an existing LUKS passphrase and remove any existing key in slot 1.
It will then generate a random key, seal it with the TPM against the current
PCR values, and store the sealed key on disk or in NVRAM depending on the
options specified.

#### temp

Set a temporary LUKS passphrase. The TPM will be used to unseal the passphrase
for LUKS slot 1, which will be used to set a temporary passphrase in slot 2.
The user will be interactively prompted to enter this temporary passphrase.

#### reset

Prompts the user for the temporary passphrase (if needed) and uses it to set a
new passphrase in slot 1. The slot one key is then sealed by the TPM using the
current PCR values, and LUKS slot 2 is cleared.

#### replace

The `replace` action allows a TPM-sealed LUKS key to be replaced (overwritten)
by a new, randomly generated key. By default, LUKS slot 1 will be replaced.
This action will not prompt for a passphrase, so the current key must be both
"unsealable" by the TPM and a valid LUKS key.

#### compute

Pre-computes the PCR values for the kernel that will be used on next boot. Use
the precomputed values to replace the current LUKS passphrase with a new,
random value.

Pre-computing the PCR values is accomplished using an external command specified
by the -c option or using the defaults file. The command is executed using the
system `sh` POSIX shell. The command is expected to accept the PCR bank
specification used to seal the passphrase, compute the PCR values for the next
system boot, and write their binary values to the supplied output path.

Two placeholders can be used in the command string:

  * `::pcr::` will be replaced by the PCR bank specfication
  * `::output::` will be replaced by the output path for the binary PCR values

In addition, the command environment will contain `PCR_LIST` and `OUTPUT_PATH`
variables that contain the same information as the placeholders.

### Default configuration

The script will read default configuration values by sourcing
`/etc/default/luks-tpm2` if it exists. The location of this file can changed by
setting the `CONFFILE` environment variable. Variables read from the config
file will override hard coded defaults, but will not override command line
arguments.

### LUKS device detection

If the path to a LUKS block device is not provided `luks-tpm2` will use the
first device with a `crypto_LUKS` filesystem.

How-to
------

`luks-tpm2` can protect LUKS keys using the TPM in one of two ways:

  * On disk as a pair of "sealed" files that can only be decrypted by the TPM
  * In TPM non-volatile memory (NVRAM)

In either case, the data is only accessible when certain Platform Configuration
Registers (PCRs) have not changed. This indicates that the system has not been
altered since the data was sealed.

Note that all TPM objects will be created in the owner hierarchy.

Before working with the TPM, consider setting the `TPM2TOOLS_TCTI` environment
variable for your TPM resource manager. For example, to use the in-kernel RM:

    $ export TPM2TOOLS_TCTI=device:/dev/tpmrm0

If you want to be able to access the TPM as a normal user, add yourself to the
`tss` group, otherwise you will have to run all the following `tpm2_` commands
as root.

## On-disk

Before storing sealed key files on disk, you must create a parent encryption key
on the TPM. In this example, we create a primary RSA key in the owner hierarchy
and make it persistent at handle `0x81000001`:

    $ tpm2_createprimary -c primary.ctx
    $ tpm2_evictcontrol -c primary.ctx 0x81000001

Next, call `luks-tpm2` with appropriate options:

    $ sudo -E luks-tpm2 -p /boot/keyfile -H 0x81000001 /dev/sdaX init

Two sealed files will be generated (in `/boot` for this example):
`/boot/keyfile.priv` and `/boot/keyfile.pub`.

### Parent key password

The parent encryption key can optionally be created with a password. This
password will need to be supplied during operations that require the parent key.
The `-K` option will cause `luks-tpm2` to display an interactive password
prompt. `-k PATH` will instead attempt to read the password from a file at PATH.

    $ tpm2_createprimary -c primary.ctx -p MyPassword
    $ tpm2_evictcontrol -c primary.ctx 0x81000001
    $ sudo -E luks-tpm2 -p /boot/keyfile -H 0x81000001 -K /dev/sdaX init

## NVRAM

Most TPMs provide a small amount of user-configurable non-volatile memory
(NVRAM) that will perisist between reboots. Note that NVRAM often has a limited
number of writes, so it may not be a good option if frequent updates are
required.

Before initializing NVRAM storage, locate a free index:

    $ tpm2_getcap handles-nv-index

And then call `luks-tpm2` with appropriate options:

    $ sudo -E luks-tpm2 -x 0x1500001 /dev/sdaX init

License and Copyright
---------------------

Copyright 2018-2020 Corey Hinshaw <corey@electrickite.org>

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
