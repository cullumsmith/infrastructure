#!/bin/sh

set -eu -o pipefail

PERCENT=$1
USER=$2
FROM=$3

cat << EOF | /usr/libexec/dovecot/dovecot-lda -d "$USER" -o "plugin/quota=count:User quota:noenforcing"
From: ${FROM}
Subject: Mailbox quota warning

This is an automatically generated message.

Your mailbox is now ${PERCENT}% full.

When your mailbox exceeds its quota, you will no longer receive new mail.

Please delete some messages to free up space.
EOF
