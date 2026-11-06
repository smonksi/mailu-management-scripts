# Mailu – Let's Encrypt Certificate Renewal Runbook

## Purpose

Renew Let's Encrypt certificates on a Mailu VM when its normal network arrangement prevents the Mailu frontend from reaching its own public hostname.

This procedure was tested successfully on the private Mailu VM.

It can be repeated on the public Mailu VM, substituting the appropriate VM IP addresses and adjusting the router NAT rules.

## 1. Understand the network arrangement

The normal arrangement is:

- Public DNS points the Mailu hostnames to the public IPv4 address.
- Incoming HTTPS traffic is normally forwarded to the Nginx reverse proxy.
- The reverse proxy forwards HTTPS traffic to the public Mailu VM.
- The private Mailu VM is accessed internally and has its own DNS resolver (Unbound).
- Mailu's frontend needs to resolve its own public hostname and reach the HTTP challenge endpoint during certificate validation.

The problem occurs when the frontend's HTTP request reaches the reverse proxy instead of the Mailu VM. The proxy returns HTTP 502 because it is not configured to forward that HTTP request to the relevant Mailu frontend.

## 2. Check Unbound and DNSSEC

Run these commands from the Mailu Docker Compose directory.

### Check the resolver container

```bash
sudo docker compose ps resolver
```

Confirm that the resolver is running and healthy.

### Test DNSSEC validation

```bash
dig @192.168.203.254 cloudflare.com
```

Expected result:

- `status: NOERROR`
- The flags should include `ad`.

Now test a deliberately DNSSEC-invalid domain:

```bash
dig @192.168.203.254 dnssec-failed.org
```

Expected result:

- `status: SERVFAIL`

This confirms that Unbound is validating DNSSEC rather than simply resolving everything.

### Test Unbound hostname resolution

```bash
sudo docker exec mailu-resolver-1 unbound-host cloudflare.com
```

Then test the Mailu hostname:

```bash
sudo docker exec mailu-resolver-1 unbound-host mail.example.com
```

The hostname should resolve to the expected public IPv4 address.

## 3. Check the Mailu frontend DNS configuration

Check the frontend container's resolver configuration:

```bash
sudo docker exec mailu-front-1 cat /etc/resolv.conf
```

Expected:

```text
nameserver 127.0.0.11
options edns0 trust-ad ndots:0
```

This is normal: `127.0.0.11` is Docker's embedded DNS resolver.

Check that the frontend is configured to use Mailu's Unbound resolver:

```bash
sudo docker inspect mailu-front-1 --format '{{json .HostConfig.Dns}}'
```

Expected:

```text
["192.168.203.254"]
```

### Test hostname resolution from the frontend

```bash
sudo docker exec mailu-front-1 python -c "import socket; print(socket.gethostbyname('mail.example.com'))"
```

The result should be the expected public IPv4 address.

## 4. Temporarily redirect router NAT

You do not need to stop the other Mailu VM.

1. Open the router's port-forwarding configuration.
2. Temporarily change the incoming TCP port 80 forwarding rule to point directly to the Mailu VM being updated.
3. Temporarily change the incoming TCP port 443 forwarding rule to the same VM.
4. Check that the public DNS A records for the Mailu hostnames point to your public IPv4 address.

**Important:** Let's Encrypt HTTP-01 validation requires inbound TCP port 80. Port 443 is not required for HTTP-01 itself, but forwarding both ports allows the Mailu instance to operate normally during the maintenance window.

## 5. Allow Mailu to renew its certificates

With the temporary NAT rules in place:

1. Ensure the Mailu VM is running.
2. Allow the Mailu frontend to perform its normal Let's Encrypt certificate checks and renewal.
3. Check the frontend logs:

```bash
sudo docker compose logs --tail=100 front
```

Look for successful certificate issuance or renewal, and check for any remaining ACME errors.

If necessary, follow the logs:

```bash
sudo docker compose logs -f front
```

Exit the log display with `Ctrl+C`.

## 6. Restore the original NAT configuration

Once certificate renewal has succeeded:

1. Restore the original TCP port 80 forwarding rule.
2. Restore the original TCP port 443 forwarding rule.
3. Confirm that the ports once again point to the normal Nginx proxy / public Mailu arrangement.
4. Check that the public Mailu service is operating normally.

## 7. Verify the renewed certificate

From a machine with access to the Mailu hostname, run:

```bash
openssl s_client -connect mail.example.com:443 -servername mail.example.com </dev/null 2>/dev/null | openssl x509 -noout -dates
```

Check the `notBefore` and `notAfter` dates to confirm the renewed certificate is being served.

Repeat for other hostnames if necessary.

---

## Important notes

- Do not change the Unbound configuration if the DNSSEC tests and hostname resolution checks are already successful.
- The Docker resolver address for this Mailu installation is `192.168.203.254`.
- The frontend container name used in these examples is `mailu-front-1`.
- The resolver container name is `mailu-resolver-1`.
- Container names may differ on the other Mailu installation.
- The public VM can remain running while the private VM is being updated, provided the NAT rules are temporarily directed to the correct VM.
- For the public Mailu VM, substitute its own LAN IP address in the router NAT rules.
- Restore the normal NAT configuration immediately after successful renewal.

## Quick renewal checklist

- [x] Resolver container healthy.
- [x] DNSSEC validation successful (`ad` flag).
- [x] Invalid DNSSEC domain returns `SERVFAIL`.
- [x] Mailu hostname resolves correctly from Unbound.
- [x] Mailu hostname resolves correctly from the frontend.
- [ ] Router TCP 80 and 443 temporarily forwarded to the VM being updated.
- [ ] Let's Encrypt renewal successful.
- [ ] Certificate expiry dates verified.
- [ ] Original NAT rules restored.
- [ ] Normal Mailu and proxy operation confirmed.