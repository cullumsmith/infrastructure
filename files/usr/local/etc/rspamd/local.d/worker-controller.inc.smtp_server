bind_socket = "0.0.0.0:${rspamd_port}";

password = "${rspamd_ro_password_hash}";
enable_password = "${rspamd_rw_password_hash}";

keypair {
  algorithm = "curve25519";
  privkey = "${rspamd_privkey}";
  type = "kex";
  encoding = "base32";
  pubkey = "${rspamd_pubkey}";
}
