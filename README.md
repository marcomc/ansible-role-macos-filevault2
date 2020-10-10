[![Build Status](https://travis-ci.com/marcomc/ansible-role-macos-filevault2.svg?branch=master)](https://travis-ci.com/marcomc/ansible-role-macos-filevault2)

# FileVault2 Ansible role for macOS

This Ansible role implements a subset of commands to `enable` (only) FileVault2 via `fdesetup` present on macOS v10.7 or newer systems.

Used in [Splinter, an opinionated provisioning tool for macOS](https://github.com/marcomc/splinter).

## Example Playbook
----------------
    - vars:
        filevault_additional_users_and_passwords:
        - { username: "testuser", password: "test_password" }
        filevault_certificate: yes
        filevault_certificate_file: "/path/to/my/DER.cer"
        filevault_showrecoverykey: yes
        filevault_user_password: "password of the user for which activate FileVaul"

    - hosts: localhost
      roles:
      - marcomc.macos_filevault2

## Variables
The subset of `enable` options implement is

     fdesetup enable
        [-outputplist]
        [-forcerestart]
        [-authrestart]
        [-keychain | [-certificate path_to_cer_file]]
        [
          [-defer file_path]
          [-forceatlogin max_cancel_attempts]
          [-dontaskatlogout]
          [-showrecoverykey]
        ]
        [-norecoverykey]
        [-verbose]

each option has been mirrored to an ansible variable (and then there are a few extra variables for more features)

    verbose: no
    filevault_user: "{{ ansible_user_id }}"
    filevault_user_password: "{{ ansible_become_pass }}"
    filevault_additional_users_and_passwords: []
      # - { username: "testuser6", password: "testest" }
      # - { username: "testuser7", password: "testest2" }
    filevault_input_plist: ''                 # useful if you want to deploy a static plist file
    filevault_keychain: no
    filevault_keychain_file: no               # ignored if institutional_type is 'certificate'

    filevault_keychain_file_override: no      # Overwrite any existing copy '/Library/Keychains/FileVaultMaster.keychain'
    filevault_certificate: no
    filevault_certificate_file: ""            # ignored if institutional_type is 'keychain'
    filevault_norecoverykey: no               # 'yes' specify that only the FileVaultMaster keychain is used as the recovery key
                                              # 'no' will generate a Personal Recovery Key
    filevault_recovery_key_output_file: "~/Desktop/{{ ansible_hostname }}-presonal-recovery-key.txt" # the path where to save the Personal Recovery Key generate by FileVault2
    filevault_outputplist: no
    filevault_defer: no
    filevault_defer_file: "/dev/null"
    filevault_showrecoverykey: no
    filevault_dontaskatlogout: no
    filevault_forcerestart: no
    filevault_authrestart: no
    filevault_forceatlogin: no
    filevault_max_cancel_attempts: '-1' # (-1: ignore this option, 0=next time, 9999=only request, not force).
    filevault_forcerestart: no

you can obtain your preferred installation method toggling the options as if you where using the command line tool directly:


    # Example: Enable FileVault2 using a certificate without a generating a personal recovery key
    filevault_user: "{{ ansible_user_id }}"
    filevault_user_password: "{{ ansible_become_pass }}"
    filevault_additional_users_and_passwords:
      - { username: "testuser6", password: "testest" }
    filevault_certificate: yes
    filevault_certificate_file: "~/Documents/certificate.cer"
    filevault_norecoverykey: yes

which corresponds to:

    fdesetup enable -certificate ~/Documents/certificate.cer -norecoverykey -inputplist < imput_plist


## Input List
You can specify your own input plist to further customise your installation or if you need another process to generate such file.

If you do not specify your own input plist (which is supposed to be the default behaviour) then a plist will generated dynamically putting together `filevault_user` and `filevault_user_password` and the list of `filevault_additional_users_and_passwords`.

## Set an `institutional` recovery key for computers in your organization
you can chose to:
* deploy a pre-generated Keychain recovery key
* deploy a DER certificate that will be added to Keychain a recovery key generated on-the-fly

### Certificate
Automatically create the institutional recovery key with the supplied certificate file
> This is my favourite option because the preparation task are minimal and the outcome is the same as per the `Keychain` option

The common name of the certificate must be "FileVault Recovery Key"

You can generate a DER certificate manually:
  1. [Create a FileVault master keychain](https://support.apple.com/en-us/HT202385#create)
  2. export the ONLY the public certificate element to FileVaultRecoveryKey.cer

Alternatively you can use the [splinter-tools/filevault-recovery-key-generator.sh](https://github.com/marcomc/splinter-tools/blob/master/recovery_key_cert_generator.sh) script
> Make sure to save both the keychain and its password in a safe place (Bitwarden, LastPass, 1Password)

The generate self-signed certificate will be named `FileVault Recovery Key (<your_hostname?)`.
your hostname is set as the certificate description and cannot be changed.

If you want the description between brackets to be something different than your hostname you have to trick keychain and temporarily set your computer name to the desired certificate `description` and then switch your hostname to the original value.

    #!/usr/bin/env bash

    # Store the original hostname
    ORIGINAL_HOSTNAME=$(eval hostname)

    # Change the hostname to the desired description
    sudo scutil --set HostName "Institutional"

    # create the keychain with the certificate
    sh filevault-recovery-key-generator.sh FileVaultMaster

    # restore the original hostnam
    sudo scutil --set HostName "${ORIGINAL_HOSTNAME}"


### Keychain
If you select keychain institutional recovery key make sure to previously generate the keychain file `FileVaultMaster.keychain` containing your recovery key, __remove the private key__, and deploy it on your machine:

1. [Create a FileVault master keychain](https://support.apple.com/en-us/HT202385#create)
2. [Remove the private key from the master keychain](https://support.apple.com/en-us/HT202385#update)
3. [Deploy the updated master keychain on each Mac](https://support.apple.com/en-us/HT202385#deploy)


## Unlock a user's startup disk
If a user forgets their macOS user account password and can't log in to their Mac, you can use the private key that was contained in the original `FileVaultMaster.key` file, to unlock the disk

* [Use the private key to unlock a user's startup disk](https://support.apple.com/en-us/HT202385#unlock)
1. Start up from macOS Recovery by holding Command-R during startup.
2. Connect the external drive that contains the `private-recovery-key` or the original `FileVaultMaster.keychain` that contains the private key.
3. Choose `Utilities > Terminal` from the menu bar in macOS Recovery.
4. unlock the FileVault master keychain

        security unlock-keychain /path/to/FileVaultMaster.keychain

5. Unlock the encrypted startup disk

        # APFS disks
        diskutil ap unlockVolume "Name of the Encrypted Drive" -recoveryKeychain /path/to/FileVaultMaster.keychain

        # Mac OS Extended (HFS Plus) disks
        diskutil cs list # find the UUID
        diskutil cs unlockVolume {UUID} -recoveryKeychain /path/to/FileVaultMaster.keychain


License
-------

[MIT](LICENSE)

Author : Marco Massari Calderone (c) 2020 - marco@marcomc.com
