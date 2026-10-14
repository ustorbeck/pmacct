## Test Description (306-BGP-pmbgpd-ADD_PATH-mpls_vpn)

IMPORTANT: for this test we use the pmbgpd daemon (not nfacctd)!

This test is a regression test for the fix in commit 4d58a25a ("bgp: fix
ADD-PATH NLRI parsing for labeled-unicast and MPLS-VPN SAFIs").

`bgp_nlri_parse()` used to only strip the 4-byte ADD-PATH Path Identifier
prefix from NLRI entries for SAFI_UNICAST and SAFI_MULTICAST. When ADD-PATH
is negotiated for an AFI/SAFI using SAFI_MPLS_VPN (rfc4364 BGP/MPLS VPN, as
used here) or SAFI_MPLS_LABEL (rfc3107 labeled unicast), the wire NLRI also
carries a leading 4-byte Path Identifier, which the parser never skipped.
This caused the first byte of the Path Identifier to be misread as the
prefix length, producing a bogus prefixlen, failing the psize sanity check
and silently discarding the whole BGP UPDATE message via 'goto
exit_fail_lane' - with no log message at any level, making the drop
invisible.

This test is based on test 301 (same captured BGP session, same 38 VPNv4
prefixes), with two modifications applied to the pcap:

- The BGP OPEN message advertises the ADD-PATH capability (RFC7911, code 69)
  for AFI=1 (IPv4) / SAFI=128 (MPLS-VPN) with SEND_RECEIVE=2 (send).
- Every NLRI entry in the MP_REACH_NLRI attribute of every UPDATE message is
  prefixed with a 4-byte Path Identifier (values counting up from 1000).

With the fix in place, all 38 prefixes are still parsed correctly (same
prefix/RD/label values as test 301) despite the extra 4 bytes per NLRI
entry. Without the fix, none of the 38 prefixes get logged at all (the
"as_path_id"-carrying UPDATE messages get silently discarded), which is
exactly the invisible-drop behaviour described in the bug.

### Provided files:
```
- 306_test.py                               pytest file defining test execution

- traffic-00.pcap                           pcap file (for traffic generator), derived from
                                             test 301's traffic-00.pcap: OPEN message patched
                                             to add the ADD-PATH capability for AFI 1/SAFI 128,
                                             and every NLRI entry in every UPDATE prefixed with
                                             a 4-byte Path Identifier
- traffic-reproducer-00.yml                 traffic replay function config file

- pmbgpd-00.conf                            pmbgpd daemon configuration file (same as test 301)

- pmacct_mount/pretag-00.map                pretag mapping file for pmbgpd              HINT: IPs need to match with repro_ips

- output-bgp-00.json                        desired pmbgpd kafka output [daisy.bgp topic] containing json messages
                                             (same prefixes/RD/labels as test 301, plus a non-null
                                             "as_path_id" field for every entry)
- output-log-00.txt                         log messages that need to be in the logfile (includes
                                             the extra "Capability: ADD-PATHs" log line)
```

### Test timeline:

t=0s --> traffic-reproducer startup-time

- t=5-6s: BGP packets sent

### Test execution and results:

Start traffic reproducer with provided config. When finished producing messages, the traffic reproducer will exit automatically (keep_open=false).
After pmbgpd produced to kafka, check the following:

- The pmbgpd kafka output messages in topic daisy.bgp need to match with the json messages in "output-bgp-00.json" (38 update messages, each with a non-null "as_path_id").
- The timestamp values will change between runs.
- Order of the json messages could change (this means you also have to ignore any sequence numbers when comparing the json output!)
- Log messages in "output-log-00.txt" are present in the logfile (order of appearence preserved, but there could/will be other logs in between)
- Excluding the ones present in the output-log-00.txt file, no additional ERROR or WARN messages are present in the logfile
