@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
SUITABLE DIGITAL FIEFDOM RESCUE PLAN
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@

*A suitable rescue plan for the frugal file custodian.*

########
Overview
########

This is one developer's strategy for maintaining a low-cost,
off-site data backup using AWS Glacier.

- Use the project's Bash `executable <bin/salvage-fiefdom>`_
  to occasionally `upload <UPLOAD.rst>`_
  an archive of your files to Glacier;
  and to delete the previous archive you may have uploaded.

- Keep these instructions (a copy-paste walkthrough) handy in case you
  ever have a catastrophic device failure and cannot recover from other
  more up-to-date local backups that you should also be maintaining.

###################################
Be Prepared to Recall this Document
###################################

Record the location of this document (yes, the one you're reading)
in your data recovery handbook (or somewhere IRL) so you know how
to find it when you need it!

- https://github.com/landonb/salvage-fiefdom

######################################
Backup Your Secret Key Values 🌠 🌟 🌞
######################################

In addition to recording the URI of this document, there are a few
secret and private values you could/should record IRL in case you
desperately need them to recover from a disaster.

- From ``~/.aws/credentials``, which is needed to access your AWS
  account using the AWS CLI, record the two values for the two keys
  (i.e., run ``cat ~/.aws/credentials``) so you can remake the file
  if necessary::

    aws_access_key_id = ____________________
    aws_secret_access_key = ________________________________________

- The Glacier Vault Name and Region are also needed.
  You'll know these values when you setup your Vault.
  Record them IRL so you have them at the ready, too::

    VAULT_NAME=__________
    REGION=________________

- To make recovery quicker and to ensure you trust the process,
  you'll want to store both the Archive ID and the Checksum values in a
  cryptographically-signed document that you keep updated online
  somewhere (to a paste-bin, or as a code gist, etc.).
  Record the URI of the Archive ID and Checksum document here::

    LATEST_ARCHIVE_ID_AND_CHECKSUM_URI=__________________________

- Finally, backup the GPG key used to secure your archive.
  I'm sure each reader will have a different approach to how
  they want to do that, but you'll naturally need both key parts,
  e.g., record::

    GPG_PUBLIC_KEY=____________________________________________________
    GPG_SECRET_KEY=______________________________________________________

  To obtain these values, you could run, e.g.::

    $ gpg --armor --export <KEY-ID>
    $ gpg --armor --export-secret-keys <KEY-ID>

How to Record IRL
=================

As mentioned earlier, you may want to keep a copy of your secret and private
key values off-site.

- You'll need this information later if you ever need to recover your digital fiefdom!

  - You'll need the AWS credentials and Archive ID to
    `access and download <#start-the-retrieval-job>`_
    the encrypted archive that you
    `previously uploaded <UPLOAD.rst#chunk-upload-the-backup-archive>`_.

  - You'll need the GPG private key to
    `decrypt your files <#decrypt-and-verify-the-archive>`_.

  - You'll want the checksum
    `to verify <#decrypt-and-verify-the-archive>`_
    that no one tampered with the archive.

- Generally, you may have the AWS credentials already on your file system,
  ready to go, and you may have the GPG key already in your user's keyring,
  but you might want to prepare for a worst case scenario where you've lost
  access to your machines.

- Depending on what you feel is most secure or most reliable,
  you might be inclined to record a copy of the secret values
  in the clear on a physical piece of paper; or you might want
  to instead save a copy of the secret values to an encrypted
  file, such as on a USB flash device. Or you might like oral
  lore, and you can get someone else to memorize your values.

  In whatever case, you'll want to store the paper or device or person
  in a separate location from the data that you're backing up, to better
  protect yourself against anything that might happen to your main data
  store from also happening to the secret values file (such as a flood,
  fire, theft, etc.).

  Fortunately, you should only need the secret values later under
  extremely rare circumstances. With that information, you can decide
  which approach for storing these values is best for you.

#################
Table of Contents
#################

.. contents::
   :depth: 2

######################
One-time AWS CLI Setup
######################

Setup AWS.

- Create an AWS account, if necessary.

  - Visit https://aws.amazon.com to create a new AWS account.

- Create a Glacier vault.

  - Use the `AWS Glacier Web console
    <https://console.aws.amazon.com/glacier/home>`_
    to create a new Vault.

    - Or install the CLI (see below) and check out the AWS command,
      ``aws glacier create-vault``.

  - Enable application access, and update your ``~/.aws/credentials`` with:
    ``aws_access_key_id`` and ``aws_secret_access_key``.

- Subscribe to Vault notifications
  (either SMS and/or email).

  - From the `AWS S3 Glacier console
    <https://console.aws.amazon.com/glacier/home>`_:

    - Choose your Vault, click the Notifications tab,
      and choose *Enabled* to activate the SNS topic.

    - Select both *Archive Retrieval Job Complete*
      and *Vault Inventory Retrieval Job Complete*.

  - From the `AWS Simple Notification Service console
    <https://console.aws.amazon.com/sns/v3/home>`_:

    - Click *Create subscription* to create either or both
      an email and an SMS subscription to the Glacier Topic.

- Install AWS CLI locally. *Let's get terminal*::

    sudo pip3 install awscli

    mkdir -p ~/.aws

    echo "[default]" > ~/.aws/config

    cat <<EOF |
    [default]
    aws_access_key_id = <your aws_access_key_id>
    aws_secret_access_key = <your aws_secret_access_key>
    EOF
    cat > ~/.aws/credentials

- For help on the AWS CLI commands used throughout this
  document, see:

    https://docs.aws.amazon.com/cli/latest/reference/glacier/index.html

########################
Determine the Archive ID
########################

This recovery procedure assumes that you previously
`uploaded an archive file <UPLOAD.rst>`_ to your
AWS Glacier Vault.

When you upload an archive file, it's a good practice
to record the Archive ID.

  - If you followed the upload procedure,
    you might find the Archive ID lurking under
    ``~/.config/salvage-fiefdom/``, e.g.,::

      ARCHIVE_ID="$(cat ~/.config/salvage-fiefdom/.archive-id.fresh)"
      echo ${ARCHIVE_ID}

    or you might have stashed the Archive ID in a pastebin or gist,
    in which case you can `skip to the next section`__.

    __ `Prepare Archive for Download`_

However, if you do not have the Archive ID -- perhaps the worst happened,
and a catastrophic event took out your devices and other local backups --
then follow along here to determine the Archive ID.

But be aware, it may take AWS close to four hours to complete the inventory
request, before you can get the Archive ID (at least that's how long it took
when the author tested).

Start the Inventory Job...
==========================

- First, set your specific Vault values in the terminal, e.g.::

    VAULT_NAME=__________
    REGION=________________

- Next, start an inventory retrieval job::

    WORKDIR=${WORKDIR:-$(mktemp --suffix='.revival' -d)} || echo "FAIL!"
    cd ${WORKDIR}

    INVENTORY_INIT_JOB=${WORKDIR}/inventory-init-job.out
    aws glacier \
      initiate-job \
      --account-id - \
      --region ${REGION} \
      --vault-name ${VAULT_NAME} \
      --job-parameters '{"Type": "inventory-retrieval"}' \
        > "${INVENTORY_INIT_JOB}"

    cat ${INVENTORY_INIT_JOB}
    # OUTPUT, e.g.,
    # {
    #  "location": "/${AWS_ACCT_ID}/vaults/${VAULT_NAME}/jobs/${INVENTORY_JOB_ID}",
    #  "jobId": "${INVENTORY_JOB_ID}"
    # }

    INVENTORY_JOB_ID=$(cat ${INVENTORY_INIT_JOB} | jq -r '.jobId')
    echo "INVENTORY_JOB_ID=${INVENTORY_JOB_ID}"

Await Inventory Completion
==========================

Wait for the job to complete.

- You can subscribe to text message and email notifications
  from Glacier using the AWS console (see `above`__).

__ `One-time AWS CLI Setup`_

- You can poll for the job status using ``describe-job`` (see `next`__).

__ `Optional: Poll describe-job`_

- You can call ``get-job-output`` until it succeeds (see `below`__).

__ `Get the Archive ID: Call get-job-output`_

Optional: Poll ``describe-job``
-------------------------------

- Use ``describe-job`` to check the job status. E.g.,::

    aws glacier describe-job \
      --account-id -\
      --region ${REGION} \
      --vault-name Grewingk \
      --job-id "${INVENTORY_JOB_ID}"

  For the first few hours after starting the inventory job,
  you'll see an in-progress status, e.g.,::

    {
      "JobId": "${INVENTORY_JOB_ID}",
      "Action": "InventoryRetrieval",
      "VaultARN": "arn:aws:glacier:${REGION}:${AWS_ACCT_ID}:vaults/${VAULT_NAME}",
      "CreationDate": "2019-10-12T00:20:15.819Z",
      "Completed": false,
      "StatusCode": "InProgress",
      "InventoryRetrievalParameters": {
        "Format": "JSON"
      }
    }

  and then eventually (say, not quite four hours later),
  you'll see a completed status, e.g.,::

    {
      "JobId": "${INVENTORY_JOB_ID}",
      "Action": "InventoryRetrieval",
      "VaultARN": "arn:aws:glacier:${REGION}:${AWS_ACCT_ID}:vaults/${VAULT_NAME}",
      "CreationDate": "2019-10-12T00:20:15.819Z",
      "Completed": true,
      "StatusCode": "Succeeded",
      "StatusMessage": "Succeeded",
      "InventorySizeInBytes": 460,
      "CompletionDate": "2019-10-12T04:11:21.439Z",
      "InventoryRetrievalParameters": {
          "Format": "JSON"
      }
    }

Get the Archive ID: Call ``get-job-output``
-------------------------------------------

- Regardless of ``describe-job``, you'll need to run ``get-job-output``
  to get the Archive ID.

  Specify a local file wherein to store the job output, and ask for it::

    INVENTORY_GET_JOB=${WORKDIR}/inventory-get-job.out
    aws glacier \
      get-job-output \
      --account-id - \
      --region ${REGION} \
      --vault-name Grewingk \
      --job-id "${INVENTORY_JOB_ID}" \
      ${INVENTORY_GET_JOB}

  The command will either fail with an error message, if the job is still churning::

    An error occurred (InvalidParameterValueException) when calling the GetJobOutput
    operation: The job is not currently available for download: ${INVENTORY_JOB_ID}

  Or the command will succeed and show you a JSON response, e.g.,::

    {
      "status": 200,
      "acceptRanges": "bytes",
      "contentType": "application/json"
    }

- On success, the job output is saved to the file you indicated and
  will contain a JSON structure, e.g.,::

    cat ${INVENTORY_GET_JOB}
    # OUTPUT, e.g.,
    # {"VaultARN":"arn:aws:glacier:${REGION}:${AWS_ACCT_ID}:vaults/${VAULT_NAME}","InventoryDate":"2019-09-24T10:23:59Z","ArchiveList":[{"ArchiveId":"${ARCHIVE_ID}","ArchiveDescription":"Blah blah.","CreationDate":"2019-09-23T20:59:38Z","Size":7572565631,"SHA256TreeHash":"abcd1234..."}]}

- Set the Archive ID locally in your terminal::

    ARCHIVE_ID=$(cat ${INVENTORY_GET_JOB} | jq -r '.ArchiveList[0].ArchiveId')
    echo "ARCHIVE_ID=${ARCHIVE_ID}"

  *CAUTION:* The last command assumes there is only one file in the archive.
  Specifically, it reads the Archive ID of the first file in the inventory list.
  If you have more than one archive file in the vault, well, you figure it out.

HINT: Stash the Archive ID
--------------------------

*HINT:* Record the ``ARCHIVE_ID`` elsewhere
so you can skip the inventory job next time!

############################
Prepare Archive for Download
############################

Start the Retrieval Job...
==========================

Prepare the archive for download. This'll take about 4 hours
(at least it did for the author).

- First, set your specific Vault values in the terminal, e.g.::

    VAULT_NAME=__________
    REGION=________________

  and most importantly::

    ARCHIVE_ID=__________

- Next, start an archive retrieval job::

    WORKDIR=${WORKDIR:-$(mktemp --suffix='.revival' -d)} || echo "FAIL!"
    cd ${WORKDIR}

    RETRIEVE_JOB_JSON=${WORKDIR}/retrieve-request.json
    cat <<EOF |
    {
      "Type": "archive-retrieval",
      "ArchiveId": "${ARCHIVE_ID}",
      "Description": "Retrieve archive on 2019-10-11 22:27"
    }
    EOF
    cat > ${RETRIEVE_JOB_JSON}

    RETRIEVE_INIT_JOB=${WORKDIR}/retrieve-init-job.out
    aws glacier \
      initiate-job \
      --account-id - \
      --region ${REGION} \
      --vault-name ${VAULT_NAME} \
      --job-parameters file://${RETRIEVE_JOB_JSON} \
      > ${RETRIEVE_INIT_JOB}

    cat ${RETRIEVE_INIT_JOB}
    # OUTPUT, e.g.,
    # {
    #  "location": "/${AWS_ACCT_ID}/vaults/${VAULT_NAME}/jobs/${RETRIEVE_JOB_ID}",
    #  "jobId": "${RETRIEVE_JOB_ID}"
    # }

    RETRIEVE_JOB_ID=$(cat ${RETRIEVE_INIT_JOB} | jq -r '.jobId')
    echo "RETRIEVE_JOB_ID=${RETRIEVE_JOB_ID}"

Await Retrieval Completion
==========================

Wait for the job to complete.

- You can poll using ``describe-job``::

    RETRIEVE_DESCRIBE_JOB=${WORKDIR}/retrieve-describe-job.out
    aws glacier describe-job \
      --account-id -\
      --region ${REGION} \
      --vault-name ${VAULT_NAME} \
      --job-id "${RETRIEVE_JOB_ID}" \
      > ${RETRIEVE_DESCRIBE_JOB}

- While the job is chugging, you'll see, e.g.,::

    cat ${RETRIEVE_DESCRIBE_JOB}
    # OUTPUT, e.g.,
    # {
    #   "JobId": "${RETRIEVE_JOB_ID}",
    #   "JobDescription": "Retrieve archive on 2019-10-11 22:27",
    #   "Action": "ArchiveRetrieval",
    #   "ArchiveId": "${ARCHIVE_ID}",
    #   "VaultARN": "arn:aws:glacier:${REGION}:${AWS_ACCT_ID}:vaults/${VAULT_NAME}",
    #   "CreationDate": "2019-10-12T03:38:54.579Z",
    #   "Completed": false,
    #   "StatusCode": "InProgress",
    #   "ArchiveSizeInBytes": 7572565631,
    #   "SHA256TreeHash": "abcd1234...",
    #   "ArchiveSHA256TreeHash": "abcd1234...",
    #   "RetrievalByteRange": "0-7572565630",
    #   "Tier": "Standard"
    # }

  And then once it's finished, you'll see, e.g.,::

    cat ${RETRIEVE_DESCRIBE_JOB}
    # OUTPUT, e.g.,
    # {
    #   "JobId": "${RETRIEVE_JOB_ID}",
    #   "JobDescription": "Retrieve archive on 2019-10-11 22:27",
    #   "Action": "ArchiveRetrieval",
    #   "ArchiveId": "${ARCHIVE_ID}",
    #   "VaultARN": "arn:aws:glacier:${REGION}:${AWS_ACCT_ID}:vaults/${VAULT_NAME}",
    #   "CreationDate": "2019-10-12T03:38:54.579Z",
    #   "Completed": true,
    #   "StatusCode": "Succeeded",
    #   "StatusMessage": "Succeeded",
    #   "ArchiveSizeInBytes": 7572565631,
    #   "CompletionDate": "2019-10-12T07:23:32.561Z",
    #   "SHA256TreeHash": "abcd1234...",
    #   "ArchiveSHA256TreeHash": "abcd1234...",
    #   "RetrievalByteRange": "0-7572565630",
    #   "Tier": "Standard"
    # }

############################
Download the Chunked Archive
############################

Download the archive in chunks.

Prepare Download Loop Variables
===============================

- Read the archive size from the job output::

    ARCHIVE_SIZE=$(cat ${RETRIEVE_DESCRIBE_JOB} | jq -r '.ArchiveSizeInBytes')
    echo "ARCHIVE_SIZE=${ARCHIVE_SIZE}"

- Determine how many chunked download operations to perform::

    CHUNK_SZ_1GB=$((2**30))
    CHUNKS_CNT=$((${ARCHIVE_SIZE} / ${CHUNK_SZ_1GB}))
    [[ $((${ARCHIVE_SIZE} % ${CHUNK_SZ_1GB})) -ne 0 ]] && CHUNKS_CNT=$((CHUNKS_CNT + 1))
    echo "ARCHIVE_SIZE: $ARCHIVE_SIZE / CHUNK_SZ_: $CHUNK_SZ_1GB / CHUNKS_CNT: $CHUNKS_CNT"

Loop and Download Chunks of Archival
====================================

- Run the chunked download operations::

    sf_chunks_download () {
      local byte_0
      local byte_n
      local ic
      for ((ic = 0; ic < ${CHUNKS_CNT}; ic++)); do
        byte_0=$((${ic} * ${CHUNK_SZ_1GB}))
        byte_n=$(((${ic} + 1) * ${CHUNK_SZ_1GB} - 1))
        [[ ${byte_n} -ge ${ARCHIVE_SIZE} ]] && byte_n=$((${ARCHIVE_SIZE} - 1))
        echo "Chunk #: ${ic} / ${byte_0} to ${byte_n}"

        RETRIEVE_CHUNK=${WORKDIR}/get-chunk.part${ic}
        aws glacier \
          get-job-output \
          --account-id - \
          --region ${REGION} \
          --vault-name ${VAULT_NAME} \
          --job-id ${RETRIEVE_JOB_ID} \
          --range bytes=${byte_0}-${byte_n} \
          ${RETRIEVE_CHUNK}
      done
    }
    sf_chunks_download

 - Each call to ``get-job-output`` will output JSON on success, e.g.,::

    {
      "checksum": "abcd1234...",
      "status": 206,
      "contentRange": "bytes 0-1073741823/7572565631",
      "acceptRanges": "bytes",
      "contentType": "application/octet-stream",
      "archiveDescription": "blah-blah",
    }

Reassemble the Singular Archival File
=====================================

- Reassemble the chunked downloads into one file::

    WORKDIR=${WORKDIR:-$(mktemp --suffix='.revival' -d)} || echo "FAIL!"
    cd ${WORKDIR}

    sf_chunks_assemble () {
      local parts_list=''
      local ic
      for ((ic = 0; ic < ${CHUNKS_CNT}; ic++)); do
        parts_list="${parts_list} get-chunk.part${ic}"
      done

      echo "cat ${parts_list} > archive.gpg"
      eval cat ${parts_list} > archive.gpg
    }
    sf_chunks_assemble

Decrypt and Verify the Archive
==============================

- Decrypt the archive, and verify its checksum.

  - Decrypt the archive::

      gpg --output archive.tar.xz --decrypt archive.gpg

      # OUTPUT, e.g.,
      # gpg: encrypted with 256-bit ECDH key, ID ABCD1234ABCD1234, created 2019-09-23
      #       "Some Body <some@body>"

  - Find the checksum you created when you uploaded the archive.

    - If you used the included `executable <bin/salvage-fiefdom>`_
      to build and upload your archive, it created an encrypted
      metadata file for you. Decrypt it and parse out the checksum::

        gpg --output .latest-checksum --decrypt .latest-checksum.gpg
        cat ${WORKDIR}/.latest-checksum | cut -f 3 > checksum

  - Generate the checksum of the unpacked archive::

      shasum -a 256 archive.tar.xz | awk '{print $1}' > archive.tar.xz.sha256sum

    And verify it!::

      diff checksum archive.tar.xz.sha256sum && echo Ok! || echo DANGER!

      # OUTPUT, e.g.,
      # Ok!

Unpack the Verified Archive
===========================

Assuming the archive checksum checked out (and you subsequently trust the
package), expand it.

::

  tar -xvJf archive.tar.xz

And then it's up to you to figure out what to do next.
*Good luck!*

