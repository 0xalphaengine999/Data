# Running VS2019 .NET Core Projects on Offline RHEL 8.5 / 9.0

A complete offline kit: build on an **offline Windows PC** with Visual Studio 2019, then run on an **offline RHEL Linux PC**.

---

## 0. How it works

```
[Internet PC]  --(USB, ONCE)-->  [Offline Linux PC]   ← install .NET runtimes + OS libs one time
[Offline Windows PC] --(USB, every project)--> [Offline Linux PC]   ← copy published app, run it
```

- You **never** build on Linux and you **never** need the internet on Windows.
- Your NuGet packages (MySQL, Redis, etc.) are already on the Windows PC. `publish` copies them into the output folder.
- Linux only needs the **.NET runtime**, which you carry in once by USB.

---

## 1. Download list (do this ONCE on any PC with internet)

Put everything in one folder on a USB drive, for example `USB/dotnet-kit/`.

### 1.1 .NET runtimes for Linux x64 (required)

VS2019 builds `netcoreapp3.1` and `net5.0` projects. All of these versions are end-of-life, so these are the **final versions** and the kit never needs refreshing.

Each link opens Microsoft's official download page, and the `.tar.gz` download starts automatically. The **ASP.NET Core Runtime** file includes the base .NET runtime, so it runs both web apps and console apps.

| # | File you get | Download link | Needed for |
|---|---|---|---|
| 1 | `aspnetcore-runtime-6.0.36-linux-x64.tar.gz` | https://dotnet.microsoft.com/en-us/download/dotnet/thank-you/runtime-aspnetcore-6.0.36-linux-x64-binaries-targz | **Required on RHEL 9**. Runs 3.1/5.0 apps via roll-forward (see section 6) |
| 2 | `aspnetcore-runtime-5.0.17-linux-x64.tar.gz` | https://dotnet.microsoft.com/en-us/download/dotnet/thank-you/runtime-aspnetcore-5.0.17-linux-x64-binaries-targz | `net5.0` projects on RHEL 8 |
| 3 | `aspnetcore-runtime-3.1.32-linux-x64.tar.gz` | https://dotnet.microsoft.com/en-us/download/dotnet/thank-you/runtime-aspnetcore-3.1.32-linux-x64-binaries-targz | `netcoreapp3.1` projects on RHEL 8 |

If a link ever changes, use the version pages and pick **ASP.NET Core Runtime → Linux → Binaries → x64**:
- https://dotnet.microsoft.com/en-us/download/dotnet/6.0
- https://dotnet.microsoft.com/en-us/download/dotnet/5.0
- https://dotnet.microsoft.com/en-us/download/dotnet/3.1

**Tip:** download all three and install them side by side. Then any VS2019 project works.

### 1.2 RHEL installation ISO (required)

You need the **same RHEL version as your Linux PC** (8.5 or 9.0). It provides the OS libraries .NET needs (`libicu`, `openssl-libs`, `krb5-libs`, `zlib`, `libstdc++`).

- If you still have the ISO or DVD you used to install RHEL, **use that. Nothing to download.**
- Otherwise download it from the Red Hat Customer Portal: https://access.redhat.com/downloads/content/rhel (free Red Hat Developer account: https://developers.redhat.com/products/rhel/download). Choose the exact version (8.5 or 9.0) → **Binary DVD (x86_64)**.

### 1.3 What you do NOT need

- ❌ NuGet packages. They're already on your Windows PC, and `publish` bundles them.
- ❌ The .NET SDK for Linux. You never build on Linux.
- ❌ Anything new on Windows. VS2019 already has everything needed to publish.

---

## 2. One-time Linux setup

Do this once per Linux PC.

### 2.1 Mount the RHEL ISO as a local package repository

```bash
sudo mkdir -p /mnt/rhel
# ISO file:
sudo mount -o loop /path/to/rhel-x86_64-dvd.iso /mnt/rhel
# or physical DVD:
# sudo mount /dev/sr0 /mnt/rhel
```

Create `/etc/yum.repos.d/local.repo`:

```ini
[local-baseos]
name=Local BaseOS
baseurl=file:///mnt/rhel/BaseOS
enabled=1
gpgcheck=0

[local-appstream]
name=Local AppStream
baseurl=file:///mnt/rhel/AppStream
enabled=1
gpgcheck=0
```

Disable online repos so `dnf` doesn't hang trying to reach the internet:

```bash
sudo dnf config-manager --disable '*' 2>/dev/null
sudo dnf config-manager --enable local-baseos local-appstream
```

### 2.2 Install the OS libraries .NET needs

```bash
sudo dnf install -y libicu openssl-libs krb5-libs zlib libstdc++ libgcc
```

### 2.3 Install the .NET runtimes (all side by side)

Plug in the USB, then:

```bash
cd /run/media/$USER/<usb-name>/dotnet-kit

sudo mkdir -p /usr/share/dotnet
for f in aspnetcore-runtime-*-linux-x64.tar.gz; do
  echo "Installing $f"
  sudo tar zxf "$f" -C /usr/share/dotnet
done
sudo ln -sf /usr/share/dotnet/dotnet /usr/bin/dotnet
```

Make `dotnet` known system-wide:

```bash
echo 'export DOTNET_ROOT=/usr/share/dotnet' | sudo tee /etc/profile.d/dotnet.sh
source /etc/profile.d/dotnet.sh
```

### 2.4 Verify

```bash
dotnet --list-runtimes
```

Expected output (one line per installed version):

```
Microsoft.AspNetCore.App 3.1.32 [/usr/share/dotnet/shared/Microsoft.AspNetCore.App]
Microsoft.AspNetCore.App 5.0.17 [/usr/share/dotnet/shared/Microsoft.AspNetCore.App]
Microsoft.AspNetCore.App 6.0.36 [/usr/share/dotnet/shared/Microsoft.AspNetCore.App]
Microsoft.NETCore.App 3.1.32 [/usr/share/dotnet/shared/Microsoft.NETCore.App]
Microsoft.NETCore.App 5.0.17 [/usr/share/dotnet/shared/Microsoft.NETCore.App]
Microsoft.NETCore.App 6.0.36 [/usr/share/dotnet/shared/Microsoft.NETCore.App]
```

✅ The one-time setup is done.

---

## 3. For every project: publish on Windows (offline)

### 3.1 Check the project first

Open the `.csproj` and confirm:

```xml
<Project Sdk="Microsoft.NET.Sdk">          <!-- or Microsoft.NET.Sdk.Web -->
  <PropertyGroup>
    <TargetFramework>netcoreapp3.1</TargetFramework>   <!-- or net5.0 -->
```

If you see `<TargetFrameworkVersion>v4.x</TargetFrameworkVersion>` instead, that's **.NET Framework**, which can't run on Linux. It must be ported to .NET Core first.

### 3.2 Publish

**Option A: Visual Studio 2019**
1. Right-click the project → **Publish**.
2. Choose **Folder** and set a path such as `C:\publish\MyApp`.
3. Click **Show all settings** (or **Edit**) and set:
   - **Configuration:** Release
   - **Deployment mode:** Framework-dependent
   - **Target runtime:** Portable
4. Click **Publish**.

**Option B: Command line** (Developer PowerShell or CMD in the project folder)

```bash
dotnet publish -c Release -o C:\publish\MyApp
```

Both options work fully offline, because no Linux-specific packages are needed.

### 3.3 Copy to USB

Copy the whole `C:\publish\MyApp` folder to the USB drive.

---

## 4. For every project: deploy and run on Linux

### 4.1 Copy and test

```bash
sudo mkdir -p /opt/MyApp
sudo cp -r /run/media/$USER/<usb-name>/MyApp/* /opt/MyApp/
sudo restorecon -Rv /opt/MyApp          # fix SELinux labels

cd /opt/MyApp
dotnet MyApp.dll
# On RHEL 9 with a 3.1 or 5.0 app:
DOTNET_ROLL_FORWARD=Major dotnet MyApp.dll
```

Press `Ctrl+C` to stop it.

### 4.2 Run as a service (auto-start on boot)

Create `/etc/systemd/system/myapp.service`:

```ini
[Unit]
Description=MyApp
After=network.target

[Service]
WorkingDirectory=/opt/MyApp
ExecStart=/usr/bin/dotnet /opt/MyApp/MyApp.dll
Restart=always
RestartSec=5
User=myappuser
Environment=DOTNET_ROOT=/usr/share/dotnet
Environment=DOTNET_ENVIRONMENT=Production
# Web apps only:
Environment=ASPNETCORE_URLS=http://0.0.0.0:5000
# RHEL 9 + netcoreapp3.1/net5.0 apps only:
Environment=DOTNET_ROLL_FORWARD=Major

[Install]
WantedBy=multi-user.target
```

Enable and start it:

```bash
sudo useradd -r -s /sbin/nologin myappuser 2>/dev/null
sudo chown -R myappuser:myappuser /opt/MyApp
sudo systemctl daemon-reload
sudo systemctl enable --now myapp
sudo systemctl status myapp
journalctl -u myapp -f          # live logs
```

### 4.3 Open the firewall (web apps only)

```bash
sudo firewall-cmd --add-port=5000/tcp --permanent
sudo firewall-cmd --reload
```

### 4.4 Updating an app later

```bash
sudo systemctl stop myapp
sudo rm -rf /opt/MyApp_prev && sudo mv /opt/MyApp /opt/MyApp_prev    # keep a backup
sudo mkdir -p /opt/MyApp
sudo cp -r /run/media/$USER/<usb-name>/MyApp/* /opt/MyApp/
sudo chown -R myappuser:myappuser /opt/MyApp
sudo restorecon -Rv /opt/MyApp
sudo systemctl start myapp
```

To roll back: stop the service, swap `/opt/MyApp_prev` back to `/opt/MyApp`, and start it again.

---

## 5. Which runtime runs which project

| Project `<TargetFramework>` | RHEL 8.x | RHEL 9.x |
|---|---|---|
| `netcoreapp3.1` | 3.1.32 (native) | 6.0.36 + `DOTNET_ROLL_FORWARD=Major` |
| `net5.0` | 5.0.17 (native) | 6.0.36 + `DOTNET_ROLL_FORWARD=Major` |
| `net6.0` | 6.0.36 (native) | 6.0.36 (native) |

---

## 6. RHEL 9 warning: OpenSSL 3

RHEL 9 ships **OpenSSL 3**. The **.NET Core 3.1 and .NET 5 runtimes don't support OpenSSL 3**. On RHEL 9 they fail with errors like:

- `No usable version of libssl was found`
- SSL/TLS errors when connecting to MySQL, or when using HTTPS or crypto

**Fix (no rebuild needed):** run the app on the .NET 6 runtime with roll-forward. Use either method:

- **Method A: environment variable.** Put `DOTNET_ROLL_FORWARD=Major` in the service file, or prefix it on the command line (see section 4).
- **Method B: edit `MyApp.runtimeconfig.json`** in the publish folder:

  ```json
  {
    "runtimeOptions": {
      "rollForward": "Major",
      "tfm": "net5.0",
      "framework": { "name": "Microsoft.AspNetCore.App", "version": "5.0.0" }
    }
  }
  ```

  Only add the `"rollForward": "Major"` line. Keep everything else exactly as it is.

- **Method C: in the `.csproj`** (applies to every future publish):

  ```xml
  <RollForward>Major</RollForward>
  ```

---

## 7. Code rules so every project runs on Linux

| Rule | Why |
|---|---|
| Use `Path.Combine()`, never `C:\...` or `"\\"` | Linux paths use `/` and have no drive letters |
| Match filename case exactly | `Config.json` ≠ `config.json` on Linux |
| Put paths and connection strings in `appsettings.json` | Change them per server without rebuilding |
| Use real IPs or hostnames for MySQL and Redis | `localhost` means the Linux PC itself |
| Avoid Registry, `System.Drawing`, `EventLog`, WMI, Windows Services | Windows-only; they crash on Linux |
| Resolve files with `AppContext.BaseDirectory` | The working directory may differ under systemd |
| Use IANA time zones (`Asia/Singapore`), not Windows IDs | Windows zone names fail on Linux |

Also make sure `appsettings.json` has **Copy to Output Directory = Copy if newer** (right-click → Properties).

---

## 8. Troubleshooting

| Error / symptom | Cause | Fix |
|---|---|---|
| `You must install or update .NET to run this application` | Required runtime version isn't installed | Run `dotnet --list-runtimes`; install the missing tarball, or use `DOTNET_ROLL_FORWARD=Major` |
| `No usable version of libssl was found` | RHEL 9 + .NET 3.1/5.0 | Section 6 |
| `Couldn't find a valid ICU package` | `libicu` missing | `sudo dnf install libicu` from the ISO, or add `<InvariantGlobalization>true</InvariantGlobalization>` to the `.csproj` and republish |
| `Could not load file or assembly 'XYZ'` | Incomplete copy | Copy the **entire** publish folder again |
| MySQL `Access denied for user 'x'@'192.168.x.x'` | MySQL user not allowed from the Linux PC's IP | On the MySQL server: `CREATE USER 'x'@'%' IDENTIFIED BY '...'; GRANT ALL ON db.* TO 'x'@'%';` |
| MySQL `caching_sha2_password` / `AllowPublicKeyRetrieval` | MySQL 8 authentication | Add `SslMode=None;AllowPublicKeyRetrieval=True;` to the connection string |
| MySQL/Redis timeout | Network or firewall | Test with `timeout 3 bash -c '</dev/tcp/<ip>/3306' && echo OPEN` (Redis: port 6379). Check `bind-address` on the server |
| Service won't start, SELinux denials | Files have wrong labels | `sudo restorecon -Rv /opt/MyApp`; check `sudo ausearch -m avc -ts recent` |
| `Permission denied` running files | Ownership | `sudo chown -R myappuser:myappuser /opt/MyApp` |

---

## 9. Quick reference

```bash
# ONE TIME (Linux)
sudo mount -o loop rhel.iso /mnt/rhel          # + create local.repo
sudo dnf install -y libicu openssl-libs krb5-libs zlib libstdc++ libgcc
sudo mkdir -p /usr/share/dotnet
for f in aspnetcore-runtime-*-linux-x64.tar.gz; do sudo tar zxf "$f" -C /usr/share/dotnet; done
sudo ln -sf /usr/share/dotnet/dotnet /usr/bin/dotnet
dotnet --list-runtimes

# EVERY PROJECT (Windows)
dotnet publish -c Release -o C:\publish\MyApp        # or VS: Folder / Framework-dependent / Portable

# EVERY PROJECT (Linux)
sudo mkdir -p /opt/MyApp && sudo cp -r <usb>/MyApp/* /opt/MyApp/
sudo restorecon -Rv /opt/MyApp
cd /opt/MyApp && dotnet MyApp.dll                     # RHEL 9 + 3.1/5.0: DOTNET_ROLL_FORWARD=Major dotnet MyApp.dll
```
