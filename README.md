# Arcadia non-docker setup

This guide was completed on an Archlinux server as **root**. But this should works on every other distro (tell me if its not)

## Prerequisites

As the official guide, you must have these prerequisites:

- PostgreSQL
- Redis
- Rust & Cargo
- Node.js & npm
- Git
- base-devel (to have all dependencies for Rust compilation)

On Arch, these can be installed with this command:

```bash
pacman -Syu postgresql redis git nodejs npm base-devel
```

`-Syu` to make sure the all other dependencies are up-to-date too.

For Rust and Cargo, i recommend you to install it with Rustup (https://rust-lang.org/learn/get-started/)

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Here, reboot the system to correctly apply everything. And after the reboot, you can already install `sqlx`, we will use it later:

```bash
cargo install sqlx-cli --no-default-features --features native-tls,postgres
```

### Redis setup

The Redis setup is as simple as just enabling (--now) the Redis' systemd unit:

```bash
systemctl enable --now redis
```

If this doesn't work (if you get a error like "service not found" or whatever), run this instead:

```bash
systemctl enable --now valkey.service
```

### Postgres setup

First, enable it (--now):

```bash
systemctl enable --now postgresql.service
```

If the service fails to start (if you see "**"/var/lib/postgres/data" is missing or empty**" in the service logs), run this and then start the service:

```bash
su -l postgres -c "initdb --locale=C.UTF-8 --encoding=UTF8 -D '/var/lib/postgres/data'"
systemctl start postgresql.service
```

Then, create the user, the DB, and apply privileges (ofc, replace `DB_ARCADIA_USER_PASSWD`. I recommend you to keep alphanumeric password (a-z A-Z 0-9) too to avoid possible issues):

```bash
sudo -u postgres psql -c "CREATE USER arcadia WITH PASSWORD 'DB_ARCADIA_USER_PASSWD';"
# if you don't want to run this command in you terminal (with a clear password added to the history), create a file named 'passwd', write the password you want in it, and then run this instead:
sudo -u postgres psql -c "CREATE USER arcadia WITH PASSWORD '$(cat passwd)';"

sudo -u postgres psql -c "CREATE DATABASE arcadia OWNER arcadia;"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE arcadia TO arcadia;"
```

## Local repo setup before running

1. First clone the OG repo and go in it:

    ```bash
    git clone https://github.com/Arcadia-Solutions/arcadia.git arcadia-source && cd arcadia-source
    ```

2. Here we will copy some .env.example to made our own ones:

    ```bash
    cp backend/api/.env.example backend/api/.env
    cp backend/storage/.env.example backend/storage/.env
    cp backend/periodic-tasks/.env.example backend/periodic-tasks/.env
    cp frontend/.env.example frontend/.env
    cp shared/.env.example shared/.env
    cp tracker/arcadia_tracker/.env.example tracker/arcadia_tracker/.env
    ```

    1. For `backend/api/.env` set these values:

       - `ACTIX_HOST` to `0.0.0.0` (if you running a reverse proxy on an another host)
       - `JWT_SECRET` to a random string of 32 alphanumeric characters (a-z A-Z 0-9)
       - `DATABASE_URL` to `postgresql://arcadia:DB_ARCADIA_USER_PASSWD@127.0.0.1:5432/arcadia` with `DB_ARCADIA_USER_PASSWD` being the password you set on the Postgres setup
       - `ARCADIA_TRACKER_NAME` to the name you want
       - `ARCADIA_FRONTEND_URL` to a full HTTPS address of the tracker website. For ex: `https://your-tracker-name.com` (we will setup a reverse proxy to make this works)
       - `ARCADIA_TRACKER_URL` to the full HTTPS address of the **tracker itself**. For ex: `https://tracker.your-tracker-name.com` (we will setup a reverse proxy to make this works)
       - If you have a domain name that can send email too, i recommend you to config the SMTP Configuration part. So your instance will be able to send emails
       - `ARCADIA_TRACKER_API_KEY` to a random string of 32 alphanumeric characters (a-z A-Z 0-9)

    2. For `backend/storage/.env` set `DATABASE_URL` to `postgresql://arcadia:DB_ARCADIA_USER_PASSWD@127.0.0.1:5432/arcadia` with `DB_ARCADIA_USER_PASSWD` being the password you set on the Postgres setup

    3. For `backend/periodic-tasks/.env` set `DATABASE_URL` to `postgresql://arcadia:DB_ARCADIA_USER_PASSWD@127.0.0.1:5432/arcadia` with `DB_ARCADIA_USER_PASSWD` being the password you set on the Postgres setup

    4. For `frontend/.env` set these values:

        - `VITE_API_BASE_URL` to the full HTTPS address of the **API**. For ex: `https://api.your-tracker-name.com` (we will setup a reverse proxy to make this works)
        - `VITE_SITE_NAME` to the name you want

    5. For `shared/.env` set `DATABASE_URL` to `postgresql://arcadia:DB_ARCADIA_USER_PASSWD@127.0.0.1:5432/arcadia` with `DB_ARCADIA_USER_PASSWD` being the password you set on the Postgres setup

    6. For `tracker/arcadia_tracker/.env` set these values:

       - `API_KEY` with the same key you set in `backend/api/.env`
       - Uncomment the `REVERSE_PROXY_CLIENT_IP_HEADER_NAME` variable
       - `DATABASE_URL` to `postgresql://arcadia:DB_ARCADIA_USER_PASSWD@127.0.0.1:5432/arcadia` with `DB_ARCADIA_USER_PASSWD` being the password you set on the Postgres setup

3. Alow FQDN in Vite:

    Inside `frontend/vite.config.js` insert this the `defineConfig({` block:

    ```vue
    server: {
        allowedHosts: true
    },
    ```

4. Then run the first DB migration:

```bash
cd backend/storage
sqlx migrate run --database-url postgresql://arcadia:DB_ARCADIA_USER_PASSWD@127.0.0.1:5432/arcadia
# with `DB_ARCADIA_USER_PASSWD` being the password you set on the Postgres setup again
# if you don't want to run this command in you terminal again (with a clear password added to the history), create a file named 'passwd', write the password you want in it, and then run this instead:
sqlx migrate run --database-url postgresql://arcadia:$(cat passwd)@127.0.0.1:5432/arcadia
```

### Assets

#### Site logo

For me, my original logo was a .png file, but you can rename it to .svg and it will works too.

Add your logo file in `frontend/src/assets/` as `logo.svg` (so `frontend/src/assets/logo.svg`)

#### Favicon

Same here, originally a .png file, renaming it to .ico will works too.

Add your favicon file in `frontend/public/` as `favicon.ico` (so `frontend/public/favicon.ico`)

#### Add logo on the auth page

> [!NOTE]  
> This is optional, i just wanted to add the tracker logo on the auth page to look more like a Gazelle site. Like RED for ex. ^^

In `frontend/src/views/AuthView.vue`, instead of:

```vue
<div class="auth-view">
  <div class="title">{{ siteName }}</div>
```

change it to this:

```vue
<div class="auth-view">
<div class="wrapper-center logo-wrapper" style="display: flex; flex-direction: column">
  <img src="@/assets/logo.svg" alt="tracker-logo" class="logo" />
</div>
```

## Start Arcadia

For this, i run a tmux sessions with multiple tabs, each tab run something.

1. Start the tracker

    ```bash
    cd tracker/arcadia_tracker
    cargo run --release
    ```

2. Start the API

    ```bash
    cd backend/api
    cargo run --release
    ```

3. Start the frontend

    ```bash
    cd frontend
    npm install # install dependencies
    npm run build && npm run dev -- --host
    ```

These ports are now listening:

- `8080` : API
- `5173` : Frontend
- `8081` : Tracker

## Reverse proxy

What you have to do now is to redirect the 3 HTTPS address to your host. For this, i used nginx-ui (a frontend for nginx). You can use these configs example below.

For `https://your-tracker-name.com`:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name your-tracker-name.com;
    return 308 https://$host$request_uri;
}
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name your-tracker-name.com;
    if ($host != $server_name) {
        return 404;
    }
    ssl_certificate /path/to/your/cert
    ssl_certificate_key /path/to/your/cert/key
    location / {
        proxy_http_version 1.1;
        client_max_body_size 10m;
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Forwarded $proxy_add_forwarded;
        proxy_pass http://ARCADIA_HOST:5173;
    }
}
```

For `https://api.your-tracker-name.com`:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name api.your-tracker-name.com;
    return 308 https://$host$request_uri;
}
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name api.your-tracker-name.com;
    ssl_certificate /path/to/your/cert
    ssl_certificate_key /path/to/your/cert/key
    location /api {
        proxy_http_version 1.1;
        client_max_body_size 10m;
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Forwarded $proxy_add_forwarded;
        proxy_pass http://ARCADIA_HOST:8080/api;
    }
}
```

For `https://tracker.your-tracker-name.com`:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name tracker.your-tracker-name.com;
    return 308 https://$host$request_uri;
}
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name tracker.your-tracker-name.com;
    ssl_certificate /path/to/your/cert
    ssl_certificate_key /path/to/your/cert/key
    location / {
        proxy_http_version 1.1;
        client_max_body_size 10m;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Forwarded $proxy_add_forwarded;
        proxy_pass http://ARCADIA_HOST:8081/;
    }
}
```

After this set, you can now open `https://your-tracker-name.com` and it will show up the login page.

## Create a new user and update its permissions

You can go to `https://your-tracker-name.com/register` and simply register a new user.

After we registered our new user, this one has no permission yet. To fix this, you have to manually input the permissions into the DB for that user.

Me i use DBeaver as my DB manager, its pretty cool since it has a GUI. For security reason, you can connect from your DBeaver to the Arcadia's Postgres via a SSH tunnel.

1. Create a new connection using the blue icon on the top left:
![connection_button](assets/1_dbeaver_create_connection_button.png)
2. Choose PostgresSQL for the database connection:
![connection_button](assets/2_dbeaver_database_connection.png)
3. On the connection settings page, first set Host input to `127.0.0.1`, the database name to `arcadia`, and then click `+ SSH, SSL, ...` button and choose `SSH:
![connection_button](assets/3_dbeaver_connection_settings.png)
4. In the SSH settings tab, enter the Arcadia host IP and the user you want to connect as:
![connection_button](assets/3-2_dbeaver_ssh_tunnel_settings.png)
For the `Authentication Method` option, it's up to you to choose what you want. For me, i have my ssh-agent running that can deliver ssh keys to programs. Once you did that, click `OK`
5. Connect to the DB by double clicking on its entry in the Database Navigator column on the left:
![connection_button](assets/4_dbeaver_connect_to_db.png)
6. DBeaver will show a warning pop-up saying that the authenticity of the remote host can't be establish. And it will ask you if you want to continue, simple click `Yes`.
7. DBeaver will then ask you to download the required files for the Postgres driver, simply click `Download`:
![connection_button](assets/5_dbeaver_download_driver.png)
8. When download finished, go open the `users` table:
![connection_button](assets/6_dbeaver_open_users_table.png)
9. And input this below into the `permissions` column for the new user created (for all permissions):

    ```plain
    {download_torrent,create_torrent_request,immune_activity_pruning,create_forum_thread,create_forum_post,send_pm,view_torrent_peers,edit_collage,edit_torrent,set_user_custom_title,link_similar_wiki_articles,edit_wiki_article,create_wiki_article,delete_forum_post,edit_forum_post,edit_forum_thread,edit_title_group,edit_title_group_comment,edit_edition_group,edit_artist,edit_series,edit_torrent_request,edit_title_group_tag,move_torrent_to_other_edition_group,delete_edition_group,edit_torrent_up_down_factors,view_stats_details,merge_title_group,delete_collage_entry,search_user_edit_change_logs,delete_torrent,delete_title_group_tag,delete_series,remove_title_group_from_series,delete_artist,delete_title_group,read_staff_pm,reply_staff_pm,unresolve_staff_pm,resolve_staff_pm,set_torrent_staff_checked,upload_torrent,pin_forum_thread,delete_forum_category,delete_forum_sub_category,delete_forum_thread,lock_forum_thread,edit_forum_sub_category,edit_forum_category,create_forum_category,create_forum_sub_category,delete_collage,create_css_sheet,edit_css_sheet,warn_user,ban_user,edit_user,edit_user_permissions,lock_user_class,change_user_class,search_unauthorized_access,see_foreign_torrent_clients,search_donation,edit_donation,edit_user_class,create_user_class,get_user_application,update_user_application,delete_user_class,edit_arcadia_settings,create_donation,delete_donation,delete_user_edit_change_log,delete_torrent_report}
    ```

    After you input this, you can save it with <kbd>CTRL</kbd> + <kbd>s</kbd>

Now you have a new account with full permission that you can use to manage the instance.
