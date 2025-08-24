# XCMD in inetutils/telnet

**Privates Telnet-Subnegotiation für lokale Kommandos (Client-seitig).**

> ⚠️ Sicherheit: XCMD führt Befehle **lokal auf dem Telnet-Client** aus. Nur in geschlossenen, vertrauenswürdigen Umgebungen nutzen!

---

## Was ist XCMD?
- Neuer privater Telnet-Option-Code: `TELOPT_XCMD = 200`.
- Wenn der Server sendet
  ```
  IAC SB 200 <ASCII-Command> IAC SE
  ```
  führt der Client `<ASCII-Command>` via `/bin/sh -lc` **lokal** aus.
- **Guard:** standardmäßig **aus**. Aktivierung nur per Umgebungsvariable:
  ```bash
  export TELNET_XCMD=1
  ```

## Dateien / Änderungen (Kurz)
- `include/arpa/telnet.h` → `#define TELOPT_XCMD 200`
- `telnet/kernel.c`, `telnet/kernel.h` → Parser & Handler für `SB 200 … SE`

## Build (Beispiel Debian/Ubuntu)
```bash
sudo apt update
sudo apt install -y build-essential autoconf automake libtool pkg-config \
  libreadline-dev libncurses-dev help2man

# aus Git-Checkout
./bootstrap 2>/dev/null || true   # falls vorhanden
autoreconf -fi 2>/dev/null || true
./configure
make -j
# optional: sudo make install
```

## Nutzung
1. Guard aktivieren
   ```bash
   export TELNET_XCMD=1
   ```
2. Verbinden
   ```bash
   ./telnet <host> <port>
   ```
3. Server sendet `SB 200 … SE` → Befehl läuft lokal auf dem Client.

---

## Minimal-Test mit **netcat** (kein Python)
Mit `nc` simulieren wir einen „Server“, der beim Connect sofort ein XCMD-Paket schickt.

### Variante A – Logfile anlegen
**Terminal A (Server-Ersatz):**
```bash
printf '\xFF\xFA\xC8echo XCMD_NETCAT_OK >> /tmp/xcmd.log\xFF\xF0' \
  | nc -l -p 2323 -N -v
```

**Terminal B (Client):**
```bash
export TELNET_XCMD=1
./telnet 127.0.0.1 2323
# irgendeine Taste / ENTER, dann Verbindung schließen
```

**Prüfen (Client):**
```bash
tail -n1 /tmp/xcmd.log
# Erwartet: XCMD_NETCAT_OK
```

### Variante B – Sprachausgabe (espeak)
**Terminal A:**
```bash
printf '\xFF\xFA\xC8espeak -v en+m3 "mighty pirate"\xFF\xF0' \
  | nc -l -p 2323 -N -v
```
**Terminal B:**
```bash
export TELNET_XCMD=1
./telnet 127.0.0.1 2323
```

### Variante C – Ton (oabeep)
**Terminal A:**
```bash
printf '\xFF\xFA\xC8oabeep -g 0.28 440:120 r:60 660:120 r:60 880:240 &\xFF\xF0' \
  | nc -l -p 2323 -N -v
```
**Hinweis:** Das `&` im Payload startet `oabeep` auf dem **Client** im Hintergrund.

> **Warum `-N`?** OpenBSD-`nc` schließt die Verbindung, sobald STDIN (hier: `printf`) fertig ist. So wird das Paket sauber gesendet und die Session beendet.

---

## Praxis-Tipps
- **Quoting:** Bytes `IAC SB 200 … IAC SE` müssen exakt übertragen werden. Mit `printf` nutzen wir `\xFF` (IAC), `\xFA` (SB), `\xC8` (200), `\xF0` (SE).
- **Shell-Fähigkeiten:** Payload läuft unter `/bin/sh -lc`. Pipes/Redirects sind möglich.
- **Rückkanal:** STDOUT/STDERR des Kommandos werden **nicht** an den Server gestreamt.

## Sicherheit
- **RCE-Charakter:** Niemals im öffentlichen Netz, nur Lab/On-Prem.
- **Guard zwingend:** Ohne `TELNET_XCMD=1` ignoriert der Client XCMD-Pakete.
- **Rechte:** Befehle laufen mit den Rechten des Telnet-Users.

## Troubleshooting
- Nichts passiert? → `echo $TELNET_XCMD` prüfen (muss `1` sein).
- `espeak`/`oabeep` unbekannt? → Pakete/Paths prüfen.
- Netcat-Variante: je nach Distribution heißt das Paket `netcat-openbsd`.

## Lizenz & Upstream
- Basierend auf GNU inetutils (GPL). Lizenzhinweise beibehalten.
- Für ein Upstream-Proposal: Feature per Runtime-Guard (vorhanden) + optional Build-Flag `--enable-xcmd`, sowie Manpage-Hinweis.

