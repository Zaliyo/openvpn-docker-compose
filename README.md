# OpenVPN Docker Compose Stack

This is a standalone docker-compose project that deploys OpenVPN from a pre-built Docker Hub image.

## Quick Start

### 1. Configuration

Copy the example environment file:

```bash
cp .env.example .env
```

Edit `.env`:

```env
# Your Docker Hub image
DOCKER_IMAGE=zaliyo/openvpn:2.7.7

# This server's public IP or hostname - clients need this to reach it.
# Every .ovpn file create-clients generates uses this as its "remote".
VPN_IP=x.x.x.x
```

`VPN_IP` has to be an address your clients can actually reach (this
server's public IP or a DNS name pointing at it) - not an internal or
LAN address. If you skip this, `create-clients` still works, but each
generated `.ovpn` file gets a placeholder `remote` line you'll have to
edit by hand.

### 2. Create Directories

```bash
mkdir -p data/conf
mkdir -p data/logs
mkdir -p data/backups
mkdir -p users
```

`data/conf` is mounted to the container's `/etc/openvpn`, so once the
container starts it will contain `openvpn.conf` and a `pki/`
subdirectory - both created automatically on first boot (see below).
Do not mount a host directory onto `/etc/openvpn/pki` directly -
`init-pki` needs to remove and recreate that directory, which is
impossible if it's itself a bind-mount point.

### 3. Start the Container

```bash
docker-compose up -d
```

That's it - **no manual initialization step is required.** On first
boot the container automatically:

1. Installs a default `openvpn.conf` into `data/conf/` if one isn't
   already there.
2. Initializes the PKI (CA, server certificate, Diffie-Hellman
   parameters, TLS-crypt key) if `data/conf/pki/ca.crt` doesn't exist
   yet.
3. Starts the OpenVPN server.

All of this happens inside the same startup - watch it happen with:

```bash
docker-compose logs -f openvpn
```

First boot takes a minute or two (mostly DH parameter generation).
Every subsequent restart is instant, since both checks above just
no-op once their files exist - your PKI and config are never
regenerated or overwritten by a later start.

If initialization ever fails partway (check the logs for why), fix
the underlying issue and either restart the container to retry
automatically, or re-run it directly:

```bash
docker-compose exec openvpn init-pki
```

### 4. Create a Client

```bash
docker-compose exec openvpn create-clients alice
```

This creates the client's certificate/key and writes a ready-to-import
`.ovpn` file to `users/alice.ovpn` (its `remote` line comes from
`VPN_IP` in `.env` - see step 1).

## Management Scripts

All commands use `docker-compose exec openvpn <command>`:

- `init-pki` - (Re-)run PKI initialization manually. Not needed on a
  normal first boot - the container does this automatically - but
  useful to retry after fixing a failed initialization.
- `create-clients <name>` - Create new VPN clients (writes `.ovpn` to `users/`)
- `revoke-clients <name>` - Revoke client certificates
- `list-clients` - List all certificates with status and expiry
- `status` - System health check and version info
- `renew-clients <name>` - Auto-renew expiring certificates
- `backup-pki` - Backup/restore PKI with encryption

### Examples

```bash
# Create clients
docker-compose exec openvpn create-clients alice
docker-compose exec openvpn create-clients bob charlie

# List clients with expiry dates
docker-compose exec openvpn list-clients

# Check system status
docker-compose exec openvpn status

# Renew expiring certificates
docker-compose exec openvpn renew-clients alice

# Backup PKI with encryption
docker-compose exec openvpn backup-pki

# Revoke a client
docker-compose exec openvpn revoke-clients baduser
```

## Logs

View OpenVPN logs:

```bash
docker-compose logs openvpn
docker-compose logs -f openvpn  # Follow logs
```

Or directly:

```bash
cat data/logs/openvpn.log
```

## Troubleshooting

**Container fails to start:**
```bash
docker-compose logs openvpn
docker-compose ps  # Check status
```

**PKI initialization failed on first boot:** check the logs for the
actual error, fix it, then either `docker-compose restart openvpn`
(retries automatically) or run `docker-compose exec openvpn init-pki`
directly once the container is up.

**Health check failing:**
```bash
docker-compose exec openvpn nc -u -z localhost 1194
```

**Certificate permissions issue:**
```bash
sudo chown -R 65534:65534 data/conf
chmod 755 data/conf
```

**Client can't connect / `.ovpn` has the wrong address:** the `remote`
line in every generated `.ovpn` comes from `VPN_IP` in `.env` at the
time `create-clients` ran. If `VPN_IP` was unset or wrong, edit the
`.ovpn` file directly, or fix `.env` and re-run `create-clients` for
that client.

## Stop & Cleanup

```bash
# Stop container
docker-compose down

# Remove container and volumes
docker-compose down -v

# Remove image
docker rmi $(DOCKER_IMAGE)
```

## Network Configuration

Edit `docker-compose.yml` to customize:
- `VPN_PORT` - Change UDP port (default: 1194)
- `openvpn-network` - Change bridge network name
- Volume mounts - Change data directories

## Security

1. **Always back up PKI before changes:**
   ```bash
   docker-compose exec openvpn backup-pki
   ```

2. **Protect `.ovpn` client files:**
   ```bash
   chmod 600 users/*.ovpn
   ```

3. **PKI backup is encrypted with AES256 (recommended)**
   - Backups saved to `data/backups/` volume
   - Optional GPG encryption with passphrase

4. **Monitor logs for issues:**
   ```bash
   docker-compose logs -f openvpn
   ```

5. **Never commit `data/`** - it holds the live PKI, including the CA
   and server private keys. `.gitignore` excludes it, but double-check
   `git status` before any `git add .` on the deployment host.

## More Information

- [README.md](https://github.com/Zaliyo/openvpn/blob/main/README.md) - Build from source instructions
- [CHANGELOG.md](https://github.com/Zaliyo/openvpn/blob/main/CHANGELOG.md) - Version history and features
