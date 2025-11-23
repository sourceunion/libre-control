# DNS Listener (Tunneling Gateway)

This directory contains the specialized listener for handling **DNS Tunneling** traffic.

## Operational Role

This service binds to UDP port 53 and masquerades as an Authoritative Name Server. It enables communication in highly restricted networks where only DNS resolution is allowed.
