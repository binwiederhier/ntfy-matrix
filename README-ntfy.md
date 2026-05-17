# ntfy Discord-Matrix Bridge

This is a fork of [matrix-docker-ansible-deploy](https://github.com/spantaleev/matrix-docker-ansible-deploy)
used to deploy a Discord-Matrix bridge for the ntfy project. The main branch in this fork is `ntfy`.

It runs a Synapse homeserver at `matrix.ntfy.sh` with [mautrix-discord](https://github.com/mautrix/discord)
to bridge Discord channels to existing Matrix rooms on matrix.org.

Please also refer to the [upstream README](README.md).

## Secrets

Secrets are stored in `secrets/secrets.yml` (gitignored). Create it with:

```yaml
vault_matrix_homeserver_generic_secret_key: '<value>'
vault_postgres_connection_password: '<value>'
vault_bridge_initial_password: '<value>'
```

## Deploy

### 1. Prepare the server

SSH into the server and update:

```bash
apt update && apt upgrade -y  # keep existing config for openssh
reboot
```

### 2. Install galaxy roles

```bash
just roles
```

### 3. Run the playbook (install or update)

Optionally, update the playbook (this updates the software too):

```bash
git pull upstream master
```

Then run the playbook:
```bash
ansible-playbook --limit matrix.ntfy.sh -i inventory/hosts setup.yml \
  --extra-vars @secrets/secrets.yml \
  --tags=install-all,ensure-matrix-users-created,start
```

If the SSH connection dies during image pulls, set `ServerAliveInterval 60` and
`ServerAliveCountMax 20` in your `~/.ssh/config` and try again.

### 4. Verify

```bash
curl https://matrix.ntfy.sh/.well-known/matrix/server
# Should return: {"m.server":"matrix.ntfy.sh:8448"}
```

## Set up the Discord bot

1. Go to https://discord.com/developers/applications and create a new application
2. Installation > uncheck "User Install", set Install Link to "None", Save
3. Bot > toggle on "Server Members Intent" and "Message Content Intent", Save
4. Reset the bot token and save it somewhere safe

## Connect the bridge to Discord

1. Go to https://app.element.io and sign in
2. Edit homeserver to `matrix.ntfy.sh`
3. Log in as `bridge` (password is `vault_bridge_initial_password` from secrets)
4. Start a chat with `@discordbot:matrix.ntfy.sh`
5. Send `help` to verify the bot is listening
6. Send `login-token bot <bot token from above>` (current token see `secrets.yml`)

## Invite bot to Discord guild

1. On the Discord application page, go to OAuth2
2. Check "bot"
3. Check permissions: "Manage Webhooks", "Send Messages", "Create Public Threads",
   "Send Messages in Threads", "Read Message History", "Add Reactions"
4. Set "Integration Type" to "Guild Install" and copy the generated URL
5. Open the URL in your browser and add the bot to the ntfy guild
6. Open each channel you want to bridge and note the channel ID from the URL:
   `discord.com/channels/<guild id>/<channel id>`

## Bridge rooms

### Invite the bot to Matrix rooms

1. Log into your `@binwiederhier:matrix.org` account
2. Invite `@discordbot:matrix.ntfy.sh` to each Matrix room
3. Set `@discordbot:matrix.ntfy.sh` power level to 50 (Moderator)

### Bridge each channel

In Element Web, logged in as `@bridge:matrix.ntfy.sh`:

1. Join the Matrix room (e.g. `#ntfy:matrix.org`)
2. Send `!discord bridge <channel id>` — this bridges Discord messages to Matrix
3. Get the Matrix room's internal ID (Room Settings > Advanced > Internal room ID)
4. Go back to the DM with `@discordbot:matrix.ntfy.sh`
5. Send `set-relay <room id> --create` — this bridges Matrix messages to Discord

## Updating

To redeploy after config changes:

```bash
ansible-playbook --limit matrix.ntfy.sh -i inventory/hosts setup.yml \
  --extra-vars @secrets/secrets.yml \
  --tags=install-all,start
```

For bridge-only changes:

```bash
ansible-playbook --limit matrix.ntfy.sh -i inventory/hosts setup.yml \
  --extra-vars @secrets/secrets.yml \
  --tags=install-mautrix-discord,start
```

## Logs

```bash
ssh root@matrix.ntfy.sh journalctl -u matrix-mautrix-discord -f   # bridge
ssh root@matrix.ntfy.sh journalctl -u matrix-synapse -f            # homeserver
ssh root@matrix.ntfy.sh journalctl -u matrix-traefik -f            # reverse proxy
```
