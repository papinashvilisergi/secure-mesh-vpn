# Secure Mesh VPN Infrastructure (Headscale + Docker)

This project provides a private, self-hosted VPN solution using Headscale and Docker. It allows you to create a secure, peer-to-peer mesh network to connect your remote devices without relying on third-party centralized servers.

---

## Architecture

```
[Alpine Client Container]
        |
        | Tailscale (userspace networking)
        |
[Headscale Server Container]  <-- private coordination server
        |
   [Docker Network]
```

- **Headscale** — an open-source, self-hosted alternative to the Tailscale control server
- **Alpine Client** — a lightweight Linux container running the Tailscale client, connected to the mesh network
- **Docker Compose** — orchestrates both services together

---

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and [Docker Compose](https://docs.docker.com/compose/install/) installed
- The server's IP address (referred to below as `host-server-ip`)

---

## Quick Start Guide

### 1. Clone the Repository

```bash
git clone https://github.com/papinashvilisergi/secure-mesh-vpn.git
```

### 2. Create Required Directories

These directories are needed for Headscale configuration and data persistence:

```bash
mkdir config data
```

- `config/` — Headscale configuration files (e.g. `config.yaml`)
- `data/` — Headscale database and other persistent state

### 3. Start the Services

```bash
docker compose up -d
```

> Starts the Headscale server and the Alpine client container in the background.

---

## Headscale Configuration

### 4. Create a User

In Headscale, devices belong to **users** (namespaces). Create one:

```bash
docker exec headscale headscale users create <username>
```

> Replace `<username>` with your desired name, e.g. `myuser`.

### 5. Generate a Pre-auth Key

A pre-auth key allows a client to join the network automatically without manual authentication:

```bash
docker exec headscale headscale preauthkeys create -u <username> --reusable
```

- `--reusable` — this key can be used by multiple devices
- Save the generated key — you will need it in the next step

---

## Client Configuration

### 6. Enter the Alpine Client Container

```bash
docker exec -it alpine-client sh
```

### 7. Install Tailscale Inside the Container

```bash
apk add tailscale
```

> Alpine's package manager (`apk`) installs both the Tailscale client and the daemon.

### 8. Start the Tailscale Daemon

```bash
tailscaled --tun=userspace-networking --socks5-server=localhost:1055 &
```

- `--tun=userspace-networking` — uses userspace networking instead of a TUN/TAP interface; required for containers that lack `CAP_NET_ADMIN` privileges
- `--socks5-server=localhost:1055` — opens a SOCKS5 proxy on port 1055, which you can use to route traffic out through the VPN from within the container
- `&` — runs the daemon in the background

### 9. Connect to the Network

```bash
tailscale up --login-server http://host-server-ip:8080 --authkey <YOUR_AUTH_KEY>
```

- `--login-server` — points to your self-hosted Headscale server instead of Tailscale's cloud
- `--authkey` — the pre-auth key generated in Step 5

---

## Checking Status

### List Registered Nodes

```bash
docker exec headscale headscale nodes list
```

> Shows all devices currently registered with this Headscale server.

### List Users

```bash
docker exec headscale headscale users list
```

> Prints all created users (namespaces).

### Find the Client's VPN IP Address

From inside the Alpine client container:

```bash
tailscale ip -4
```

> Returns the IPv4 address assigned by the Tailscale network (typically in the `100.64.x.x` range).

---

## Connection Test

### Ping the Headscale Server

```bash
ping 100.64.0.1
```

**How do we know this IP?**

`100.64.0.1` is the default Tailscale IP of the **Headscale server**. Headscale assigns IP addresses from the `100.64.0.0/10` CGNAT range ([RFC 6598](https://www.rfc-editor.org/rfc/rfc6598)). The server itself receives the first address in that range — `100.64.0.1`. You can verify this by running:

```bash
docker exec headscale headscale nodes list
```

The `IP ADDRESSES` column in the output will show the exact addresses of all registered nodes.

---

## Useful Commands

| Command | Description |
|---------|-------------|
| `docker compose up -d` | Start all services in the background |
| `docker compose down` | Stop all services |
| `docker compose logs -f headscale` | Stream Headscale logs in real time |
| `docker exec headscale headscale nodes list` | List all registered nodes |
| `docker exec headscale headscale users list` | List all users |
| `docker exec headscale headscale preauthkeys list -u <user>` | List pre-auth keys for a user |
| `tailscale status` | (inside client) Show VPN connection status |
| `tailscale ip -4` | (inside client) Show the node's Tailscale IPv4 address |

---

## Troubleshooting

**Client cannot reach the server**
- Make sure `host-server-ip:8080` is reachable from the client container
- Check Headscale logs: `docker compose logs headscale`

**`tailscaled` starts but `tailscale up` cannot find the server**
- Verify the `--login-server` URL is correct and uses HTTP (not HTTPS) unless you have TLS configured

**IP conflict**
- Headscale automatically manages the `100.64.0.0/10` range — no manual intervention is needed

---

## License

MIT
