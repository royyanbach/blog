---
title: 'Debugging WSL2, Docker, and PostgreSQL Connection Issues'
description: 'Lesson learned after wasting hours of debugging connection issues from NestJS project to containerized PostgreSQL in WSL2'
pubDate: 2024-12-06
category: 'tech'
---

If you're running a setup with your backend application in WSL2 and your PostgreSQL database in Docker Desktop on Windows, you might eventually hit a wall trying to connect the two. I recently spent way too long figuring out why my NestJS app in WSL couldn't talk to its Dockerized Postgres, even though DBeaver on Windows connected just fine. Here's a breakdown of the troubleshooting steps, the red herrings, and the final solution.

## The Setup & Initial Problem

*   **Application:** NestJS running via `yarn start:dev` directly in WSL2.
*   **Database:** PostgreSQL 16 running in a Docker container, managed by Docker Desktop for Windows (using the WSL2 backend).
*   **Initial `.env**:**
    ```dotenv
    DB_HOST=host.docker.internal # Standard DNS for host from container/WSL
    DB_PORT=5432
    DB_USER=postgres
    DB_PASSWORD=postgres
    DB_NAME=insight
    ```
*   **Symptom:** NestJS app throws `ETIMEDOUT` errors when trying to establish the initial database connection. DBeaver on Windows connects without issue using the same credentials and `host.docker.internal`.

## Step 1: Basic Config Validation

First things first: are the variables even being read correctly?

A quick check in `app.module.ts` revealed a slight drift between my `.env` and the `ConfigService` usage:

```typescript
// ConfigService was using .get('DB_USERNAME') and .get('DB_DATABASE')
// My .env had DB_USER and DB_NAME

// Fixed code:
username: configService.get<string>('DB_USER', 'postgres'),
database: configService.get<string>('DB_NAME', 'insight'),
```

Fixed that, restarted. Still `ETIMEDOUT`. Okay, not just a typo.

## Step 2: Checking PostgreSQL Authentication (`pg_hba.conf`)

Maybe the container itself wasn't allowing connections from the WSL network bridge?

```bash
docker exec -it insight-postgres cat /var/lib/postgresql/data/pg_hba.conf
```

Initially, it only had `trust` rules for `local`, `127.0.0.1`, and `::1`. Connections from Docker network IPs weren't explicitly allowed. Let's add a rule to allow any IPv4 host using password (`md5`) auth:

```bash
docker exec -i insight-postgres sh -c "echo 'host    all             all             0.0.0.0/0               md5' >> /var/lib/postgresql/data/pg_hba.conf"
docker exec -u postgres -i insight-postgres pg_ctl reload
```

This changed the game, but not in the way I expected. Now, trying to connect from WSL using `DB_HOST=localhost` (or `127.0.0.1`) resulted in:

```
error: password authentication failed for user "postgres"
```

This was weird. The `pg_hba.conf` had `trust` rules for loopback IPs, which *should* mean no password check. Why was it failing auth?

## Step 3: The `localhost` Red Herring (and the Native Postgres)

This led down a rabbit hole of checking connection logs (`log_connections = on` in `postgresql.conf`), which bizarrely showed *no logs* for these failed localhost attempts. The connection was clearly hitting the server process (to get the FATAL error) but wasn't being logged correctly pre-authentication.

The breakthrough came from remembering: **I also had PostgreSQL installed and running natively inside my WSL distribution.**

Of course! Any connection from WSL to `localhost:5432` was hitting the *native WSL Postgres*, not the Docker container. The password failure was simply because the `.env` password didn't match the native instance's password.

## Step 4: Targeting the Host - Firewall and Docker Bindings

Okay, so the WSL app *must* target the Windows host IP where Docker maps the port.

*   Tried `DB_HOST=host.docker.internal` (resolved via `nslookup` to `192.168.0.64`) -> `ETIMEDOUT`. Firewall maybe?
*   Tried `DB_HOST=10.255.255.254` (Host IP from `cat /etc/resolv.conf`) -> `ECONNREFUSED`. Nothing listening?

Let's address both:

1.  **Windows Firewall Rule:** Added a rule via PowerShell (Admin) to allow TCP 5432 from the WSL subnet (`172.19.32.0/20` in my case, check yours with `ip addr show` in WSL).
    ```powershell
    New-NetFirewallRule -DisplayName "WSL PostgreSQL Docker Inbound" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 5432 -RemoteAddress 172.19.32.0/20
    ```
2.  **Explicit Docker Port Binding:** Changed `docker-compose.yml` to ensure Docker binds to all host interfaces, not potentially just loopback.
    ```yaml
    services:
      postgres:
        ports:
          - "0.0.0.0:5432:5432" # Explicitly bind host port to 0.0.0.0
    ```

Recreated the container (`docker-compose down && docker-compose up -d --force-recreate postgres`), restarted the app... still the same `ETIMEDOUT`/`ECONNREFUSED` errors.

## Step 5: The `netstat` Conflict Reveal

What is *actually* listening on port 5432 on the Windows host? Time for `netstat` in Windows CMD/PowerShell:

```cmd
netstat -ano | findstr ":5432"
```

Output:

```txt {2,4}
TCP    0.0.0.0:5432           0.0.0.0:0              LISTENING       27540 # com.docker.backend.exe
TCP    127.0.0.1:5432         0.0.0.0:0              LISTENING       32648 # wslrelay.exe !!
TCP    [::]:5432              [::]:0                 LISTENING       27540 # com.docker.backend.exe
TCP    [::1]:5432             [::]:0                 LISTENING       32648 # wslrelay.exe !!
```

There it is! Two different processes. Docker (`com.docker.backend.exe`) was correctly listening on all interfaces (`0.0.0.0`, `[::]`). But `wslrelay.exe`, a WSL component, was *also* listening, but *only* on the loopback interfaces (`127.0.0.1`, `[::1]`). This conflict, likely triggered by the now-stopped native WSL Postgres service, was preventing Docker from correctly handling loopback traffic routed from WSL.

> You might have postgresql running in WSL2, which will conflict with the dockerized postgres if you don't stop it. This wasted a lot of time for me. Had i known this, i would have stopped the native postgres service in WSL2 in the first place.

## Step 6: The Final Fix - Bypassing the IPv6 Conflict

Even after stopping the native WSL Postgres service and restarting the entire machine, `netstat` showed `wslrelay.exe` *still* listening, but only on the IPv6 loopback `[::1]:5432`:

```txt {3}
TCP    0.0.0.0:5432           0.0.0.0:0              LISTENING       31128 # Docker
TCP    [::]:5432              [::]:0                 LISTENING       31128 # Docker
TCP    [::1]:5432             [::]:0                 LISTENING       30144 # wslrelay.exe !!
```

The IPv4 loopback (`127.0.0.1`) was finally clear! The solution was now obvious: force the connection over IPv4.

```dotenv
# .env
DB_HOST=127.0.0.1 # Force IPv4 loopback // [!code highlight]
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=postgres
DB_NAME=insight
DB_DEBUG=true
```

Restarted the NestJS app, and *finally*, the connection worked.

## Key Takeaways

*   Connections from WSL to `localhost` might not go where you expect if you have native services running in WSL on the same port.
*   Bridging WSL to Docker on the host often requires using `host.docker.internal` or the host IP from `/etc/resolv.conf`.
*   Windows Firewall can block connections from the WSL subnet to the host.
*   Docker port bindings might need explicit `0.0.0.0` to listen correctly for external connections.
*   `netstat -ano` on the Windows host is invaluable for diagnosing port conflicts, especially looking for unexpected processes like `wslrelay.exe` listening where Docker should be.
*   When in doubt, forcing IPv4 (`127.0.0.1`) can sometimes bypass stubborn IPv6 loopback conflicts between WSL and Windows host components.

Hopefully, detailing this process saves someone else from a similar descent into networking madness!
