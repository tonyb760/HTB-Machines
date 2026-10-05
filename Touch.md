HTB Tuch — spoiler-free guide
A progressive hint sheet for Tuch. It avoids credentials, exploit payloads, and flags. Stop reading when you have enough direction.
Starting clue
The Layover machine gave you a name and a booking code: Jenny Crawford / KS7X2M. Keep them in mind as you enumerate Tuch; determine whether the new machine has a service that can use device or customer context.
Progressive hints
<details>
<summary>Hint 1 — where to focus</summary>
Prioritize the web service on TCP/8443. Try the scheme that actually responds; a TLS failure does not mean the port has no useful web application.
</details>
<details>
<summary>Hint 2 — before authentication</summary>
The service is Nexion DeviceHub. Check its API routes before assuming every page requires a login. Device status and inventory details are useful clues.
</details>
<details>
<summary>Hint 3 — reaching the dashboard</summary>
Compare the information disclosed by the API with the login behavior. A device identifier may also be accepted as a weak password.
</details>
<details>
<summary>Hint 4 — turning the dashboard into access</summary>
Once authenticated, inspect the JavaScript the browser receives. Look for client-side configuration and credentials instead of looking for another server-side exploit.
</details>
<details>
<summary>Hint 5 — the user foothold</summary>
The disclosed account is for RDP on TCP/3389. The session is a restricted Windows kiosk; the airport check-in UI is part of the intended path.
</details>
<details>
<summary>Hint 6 — reaching the flag</summary>
The badge reader is broken. Follow the support link shown by its error, then inspect the browser and local user context already available in that kiosk session. The user flag is on the logged-in account’s desktop.
</details>
Keeping the route focused
Treat 8443 → DeviceHub → browser-delivered credentials → RDP as the main chain.
RDP is the foothold after web enumeration, so repeated password guessing there is unlikely to help.
Once the dashboard is open, inspect what the browser has already been given before pursuing unrelated services.
Root-stage hints
<details>
<summary>Hint 7 — after the user foothold</summary>
Get an interactive command prompt as the kiosk account, then inspect the local MySQL service and its configuration. It runs with a much stronger Windows service identity than the logged-in user.
</details>
<details>
<summary>Hint 8 — finding database access</summary>
Look through the Airways application’s ProgramData configuration and readable refresh/sync files. One of those files contains database credentials. The service is reachable locally even though its port is not an exposed network service.
</details>
<details>
<summary>Hint 9 — the escalation mechanism</summary>
MySQL’s file import/export setting blocks the obvious direct file read. Check the MySQL plugin directory’s permissions. A writable plugin path plus MySQL running as LocalSystem points to a user-defined function for command execution.
</details>
<details>
<summary>Hint 10 — confirming root</summary>
Load the 64-bit MySQL system UDF, confirm the command identity, then read the Administrator desktop flag through the SQL function. A hex-encoded command avoids Windows backslash escaping surprises.
</details>
> **Instance note:** the user flag was refreshed on `10.129.100.245`; the recorded root escalation was completed on the earlier `10.129.100.153` instance. A reset may change flags.
