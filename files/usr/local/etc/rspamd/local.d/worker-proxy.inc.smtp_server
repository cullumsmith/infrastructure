bind_socket = "${rspamd_milter_sock} owner=${rspamd_user} group=${postfix_user} mode=0660";

count = ${rspamd_processes};

upstream "local" {
  self_scan = yes;
}
