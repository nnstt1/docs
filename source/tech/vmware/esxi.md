# ESXi

## SNMP

ssh で ESXi にログイン

```sh
esxcli system snmp set --communities public
esxcli system snmp set --enable true
esxcli network firewall ruleset set --ruleset-id=snmp --allowed-all true
esxcli network firewall ruleset set --ruleset-id=snmp --enabled true
/etc/init.d/snmpd start
```