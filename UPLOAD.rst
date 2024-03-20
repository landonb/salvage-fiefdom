@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
SUITABLE DIGITAL FIEFDOM BACKUP PLAN
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@

*A suitable backup plan for the frugal file custodian.*

###################################
Worst-Case Scenario Backup Strategy
###################################

This document is one of two documents that describe a thrifty,
cloud-centric backup and recovery procedure for anyone's *digital
fiefdom*, or the collection of their personal files and projects,
using Linux terminal commands and AWS Glacier.

One should also backup their data to local sources that would be easier from
which to recover, such as USB flash devices and local hard drives. But just
in case you need to recover from an extreme failure, and just in case you
cannot access the aforementioned local sources from which to recover your
data, one might also want to keep a single, encrypted archive of their data on
AWS Glacier, which is a more cost-conscious option than other backup solutions.

In practicality, the procedure outlined herein really just details uploading
a single file to AWS Glacier, at some point later possibly downloading that
file, and also replacing that file on Glacier with newer versions.

The archive file need not be your digital fiefdom, but it's for that purpose
that this document was originally curated.

See also: The Recovery Plan
===========================

Follow the procedure below regularly to keep your data
`backed up`__.

__ `Create a Backup Archive`_

Then, should you ever need to claw your way out of a catastrophe,
refer to the `recovery plan <README.rst>`_.

The example code in this file is part of ``bin/salvage-fiefdom`` 
================================================================

The `executable <bin/salvage-fiefdom>`_
script supersedes the code demonstrated this document.

The executable is actively used and represents an actual working implementation,
while the code examples in this document will not necessarily be maintained
(but they're still helpful if you're new to Glacier and want to learn).

This code shown in this document should still work (if you'd like to copy
and paste it), but you're encouraged to read (and to run) the executable
instead, e.g.,::

  TREEHASHER=path/to/treehasher \
  ./bin/salvage-fiefdom upload \
    --region "us-east-1" \
    --vault-name "My Vault" \
    --gpg-keyid "ABCD1234" \
    path/to/files

#################
Table of Contents
#################

.. contents::
   :depth: 2

############################
How Often Should You Backup?
############################

First understand that Glacier is not like normal storage you might pay
for that gives you a set amount of storage space and doesn't care how
often you read or write to that space.

Rather, Glacier charges you per file (aka archive) that you upload, such
that if you upload one file, then edit that file and upload it again,
that would really count as two archives on Glacier. (You could delete the
first archive, true, but because of the 90 day retention policy, you'd
still be charged a minimum of 90 days of storage for the first archive.)

As such, Glacier encourages you to upload as few files as possible, and to
let them sit as long as possible. So the strategy outlined in this document
is to bundle all of your files into a single archive that you upload to
Glacier, and then to replace that file at some interval (say, weekly), or
based on some trigger (e.g., after lots of edits).

So let's determine how often to backup by considering the cost of each backup
archive that gets uploaded and stored on AWS Glacier, and considering how
important it is to have the most up-to-date archive of our data stored.

First, estimate the cost of using AWS Glacier
=============================================

AWS Glacier is an inexpensive cloud storage option,
at the time of this writing costing a fraction of a
cent per GB per month ($0.004/GB/month;
see `AWS Glacier Pricing <https://aws.amazon.com/glacier/pricing/>`_).

If your goal is to backup non-media data, i.e., not photos, music, or movies,
then you probably don't have an astronomical amount of data, and it probably
doesn't grow that fast.

So let's assume you have on the order of 10GB of data. The cost to store
1 copy of that archive for 1 month is: 10 * $0.004, or 4 cents USD.

But Glacier has a 90 day retention policy, so the cost of that archive is
at least the cost of storing it for 3 months. E.g., if you
upload a new backup each month, and you delete the old backup, the old
backup will charged for the 30 days it was stored and also the remaining
60 days of the retention period, or 3 * $0.04 = 12 cents.

It's easy to see, then, that if you rotate backups every day, you'll pay
12 cents each day (12 cents for each archive),
or at most 0.12 * 31 = $3.72 per month.

If you back up a little less frequently, say, every week, you'll pay
0.12 * (31/7), or about $0.53 each month.

And note that if you backup monthly, you'll pay, obviously, 0.12 * 1 = $0.12 per month.

(You could backup once per quarter, paying just $0.12 total every 3 months,
and then it is truly just $0.04 per month!)

Next, consider the chance you'll need to recover *from AWS Glacier*
===================================================================

You're probably using AWS Glacier for peace of mind, rather than
expecting to rely on it to actually recover your data.

That is, you probably also backup your data locally, such as to a USB
flash device that you carry around, or to a backup drive that you stash
in a fire safe. And you probably have more than one development machine,
such as a laptop and a desktop, each with redundant copies of your data.

But you may also have a tiny amount of worry that something catastrophic
could still happen to all your devices (and possibly to you, too, in which
case you won't need to follow these instructions, but I digress), which is
why you also want to store a copy of your data in the cloud.

And finally, you're cheap, so the idea of paying on the order of pennies
to store your data every month is a lot more appealing than shelling out
five or 10 bucks for a more polished backup service. A backup service is also
likely to charge you for more space than you need, e.g., one popular service
currently has a free account that includes 2GB of storage, which is 8GB less
than you need for 10GB of data; but their cheapest paid option is $12.50 for
3TB per month, which is 2090GB more than you need!

And then you've got your answer
===============================

I'll conclude that you're probably using AWS Glacier as a cheap, worst-case
backup store. You want the data stored there to be somewhat up to date, and
you don't care that it takes hours to retrieve your data (because the service
is that much cheaper, and because you don't expect to ever need to retrieve
your data). And you want to brag to your friends how little you pay for your
backup solution.

So you'll probably backup every week or so, or selectively after large data
events.

And then you'll pay around four-bits per month to maintain a fairly-fresh
off-site backup of 10GB or so of data.

####################
One-time Setup Tasks
####################

1. Create an AWS account, if necessary;
   Create a new Glacier Vault; and
   Install the AWS CLI command line interface application.

  - Refer to `One-time AWS CLI Setup <README.rst#one-time-aws-cli-setup>`_.

2. Setup a GPG key for encrypting the archive.

  - The current (2019) recommendation [citation needed]
    for key generation is to use elliptic-curve cryptography.

    To generate a new key and add it to your user's GPG keychain, run::

      gpg --expert --full-gen-key

    choose elliptic curve cryptography::

      (9) ECC and ECC

    and select the first curve offered::

      (1) Curve 25519

    Also, set the key to expire after 1 year, as a best practice::

      Key is valid for? (0) 1y

    Follow the other ``gpg`` prompts to finish generating the key.

  - Set the GPG Key ID to a Bash variable for later usage, e.g.,::

      GPG_KEYID=ABCD1234

    You can see Key IDs with the command, ``gpg --list-secret-keys``

3. Install a utility to compute the tree hash of a collection
   of chunked files. Clone the project somewhere and set the path
   to a local variable for later script usage::

    cd choose/your/own/path

    git clone git@github.com:changyy/aws-glacier-sha256-tree-hash.git

    TREEHASHER=$(pwd)/aws-glacier-sha256-tree-hash/main.py

##########################
Assemble Your Backup Files
##########################

Unless you already have all of your files under one directory, you'll
probably need to assemble them all in one location, which we will then
then flatten, compress, and encrypt into a single file.

For example, the author tracks all of their data using multiple git
repositories. A script is then run which calls ``git clone --bare``
for each project, pulling data under a single, new directory path.

The new directory hierarchy is then flattened by ``tar``, compressed,
encrypted, signed, and finally copied to the AWS Glacier Vault.

In your terminal, set the ``FIEFDOM`` variable to the path of your data, e.g.,::

  FIEFDOM=${FIEFDOM:-$(mktemp --suffix='.fiefdom' -d)} || echo "FAIL!"

Then, copy files under ``$FIEFDOM/``, and you'll be prepared for the next steps.

#######################
Create a Backup Archive
#######################

Now that you've got your fiefdom prepared, flatten and compress it::

  WORKDIR=${WORKDIR:-$(mktemp --suffix='.workdir' -d)} || echo "FAIL!"

  ARCHIVE_FLAT=${WORKDIR}/archive.tar.xz

  tar -cvJf ${ARCHIVE_FLAT} ${FIEFDOM}

Use the GPG Key ID you determined earlier (and set to ``$GPG_KEYID``)
to encrypt the archive::

  CRYPTED=${WORKDIR}/upload.gpg

  gpg --encrypt \
    --recipient ${GPG_KEYID} \
    --output ${CRYPTED} \
    ${ARCHIVE_FLAT}

Calculate the archive checksum and save it to a file::

  CHECKSUM_PLN=${ARCHIVE_FLAT}.sha256sum

  shasum -a 256 ${ARCHIVE_FLAT} | awk '{print $1}' > ${CHECKSUM_PLN}

It's probably unlikely that anyone would tamper with your archive
on Glacier, but just in case, you can encrypt the checksum and
cache it somewhere else, so you can compare it to the archive if
you need to perform a recovery. Encrypt the checksum file::

  CHECKSUM_SIG=${CHECKSUM_PLN}.sig

  gpg --encrypt \
    --recipient ${GPG_KEYID} \
    --output ${CHECKSUM_SIG} \
    ${CHECKSUM_PLN}

Calculate the tree hash, which AWS also calculates,
which is needed to complete the upload operation::

  TREEHASHVAL=$( \
    py2 ${TREEHASHER} ${CRYPTED} \
    | /usr/bin/env sed -E "s/SHA-256 Tree Hash = //" \
  )

###############################
Chunk-Upload the Backup Archive
###############################

Split the archive file into chunks::

  CHUNK_SZ_4GB=$((2**32))

  CHUNKS_DIR=${WORKDIR}/put-chunks

  mkdir -p ${CHUNKS_DIR}

  SPLIT_OUTPUT=${WORKDIR}/split-chunks.out

  split -n --bytes=${CHUNK_SZ_4GB} --verbose \
    ${CRYPTED} ${CHUNKS_DIR}/chunk \
    > ${SPLIT_OUTPUT}

  SPLIT_CHUNK_CNT=$(wc -l ${SPLIT_OUTPUT})

Set variables specific to your Vault
(from the `list of variables to set <README.rst#record-these-values>`_)
e.g.,::

  VAULT_NAME=__________
  REGION=________________
  
Initiate the chunked, multipart upload::

  UPLOAD_INIT_OUT=${WORKDIR}/multipart-upload-init.out

  ARCHIVE_DESCRIPTION="fiefdom-$(date +%Y-%m-%d)"

  aws glacier initiate-multipart-upload \
    --account-id - \
    --archive-description "${ARCHIVE_DESCRIPTION}" \
    --part-size ${CHUNK_SZ_4GB} \
    --vault-name ${VAULT_NAME} \
    --region ${REGION} \
    > ${UPLOAD_INIT_OUT}

  UPLOAD_ID=$(cat ${UPLOAD_INIT_OUT} | jq -r '.uploadId')

Upload each chunk, one by one::

  local ic
  for ((ic = 0; ic < ${SPLIT_CHUNK_CNT}; ic++)); do
    local byte_0
    local byte_n
    local part_upload_out
    byte_0=$((${ic} * 2**32))
    byte_n=$(((${ic} + 1) * 2**32 - 1))
    echo "Chunk #: ${ic} / ${byte_0} to ${byte_n}"
    part_upload_out=${WORKDIR}/multipart-part-upload.out.${ic}

    aws glacier upload-multipart-part \
      --account-id - \
      --region ${REGION} \
      --vault-name ${VAULT_NAME} \
      --upload-id="${UPLOAD_ID}" \
      --body ${CHUNKS_DIR}/chunk${ic} \
      --range "bytes ${byte_0}-${byte_n}/*" \
      > ${part_upload_out}

  done

Complete the upload operation::

  ENCRYPTED_SZ=$(command du -b ${CRYPTED} | cut -f 1)
  UPLOAD_SIZE0=$((${ENCRYPTED_SZ} - 1))

  UPLOAD_COMPLETE_OUT=${WORKDIR}/multipart-upload-complete.out

  aws glacier \
    complete-multipart-upload \
      --account-id - \
      --region ${REGION} \
      --vault-name ${VAULT_NAME} \
      --upload-id="${UPLOAD_ID}" \
      --archive-size ${UPLOAD_SIZE0} \
      --checksum ${TREEHASHVAL} \
        > ${UPLOAD_COMPLETE_OUT}

  NEW_ARCHIVE_ID="$(cat ${UPLOAD_COMPLETE_OUT} | jq -r '.archiveId')"

Remember the new Archive ID.

  - The author uses config files to remember the Archive IDs of the two
    most recent archives uploaded.

    For example::

      FIEF_CFG=${HOME}/.config/salvage-fiefdom
      mkdir -p ${FIEF_CFG}
      if [[ -s ${FIEF_CFG}/.archive-id.fresh ]]; then
        mv ${FIEF_CFG}/.archive-id.fresh ${FIEF_CFG}/.archive-id.older
      fi
      echo ${NEW_ARCHIVE_ID} > ${FIEF_CFG}/.archive-id.newer

    So there are two files, ``~/.config/salvage-fiefdom/.archive-id.older``
    and ``~/.config/salvage-fiefdom/.archive-id.newer``.

    Then, 24 hours after the upload, AWS will have updated the inventory,
    and you can verify the latest upload was successful.
    And upon confirming that latest upload was successful,
    you can delete the archive identified by ``.archive-id.older``,
    and remove (or empty) that file.

#######################################
Remove the Old Archive File and Cleanup
#######################################

Verify that the new file was uploaded successfully (e.g., check back after 24 hours)::

  aws glacier list-vaults --account-id - --region ${REGION}

  aws glacier describe-vault --account-id -  --region ${REGION} --vault-name ${VAULT_NAME}

Delete the old archive -- you probably don't need to verify that the new file
shows up in the vault list first; but if you're a paranoid one, go ahead and
wait, and then delete the old archive::

  [[ ! -s ${FIEF_CFG}/.archive-id.older ]] && echo "NOOP: Nothing to do."

  OLD_ARCHIVE_ID="$(cat ${FIEF_CFG}/.archive-id.older)"

  aws glacier \
    delete-archive \
    --account-id - \
    --region ${REGION} \
    --vault-name ${VAULT_NAME} \
    --archive-id="${OLD_ARCHIVE_ID}"

  /bin/rm ${FIEF_CFG}/.archive-id.older
  mv ${FIEF_CFG}/.archive-id.newer ${FIEF_CFG}/.archive-id.fresh

You can clean up your mess, too::

  /bin/rm -rf ${FIEFDOM}
  /bin/rm -rf ${WORKDIR}

Finally, you might want to IRL record your
`secret values <README.rst#record-these-values>`_
some place where they'll be secure and also be accessible,
in case you lose access to your password manager and
all other means of recovering from a data catastrophe.
You can use those values and the
`archive-retrieval <README.rst#prepare-archive-for-download>`_
procedure to retrieve your data from AWS Glacier.

