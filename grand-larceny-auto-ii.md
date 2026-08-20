# Grand Larceny Auto II — TryHackMe Write-up

## The idea

The sequel flips the whole thing. The flag isn't in the game this time — it's on a server (a Kestrel API called `GLAII_Server`). The game client sends "proof of play" checkpoints, and only if you complete the heist does the server hand over the flag. The catch: every request is signed, and the signing key lives in the client. So the job is: pull the game apart, figure out how it talks to the server, and talk to the server yourself.

## Recon

```
$ nmap -Pn -T4 -p- 10.129.182.9
22/tcp open  ssh
80/tcp open  http    (Server: Kestrel)
```

Fuzzing gave a small API surface:

```
POST /session     -> {"session_id":"...","stash_order":[0,2,1],"token":"..."}
POST /checkpoint  -> {"error":"bad_sig"}
POST /claim       -> {"error":"incomplete","at":0,"need":5}
```

So: create a session, report progress, and once you've proven enough, claim the flag. `bad_sig` means there's a signature to figure out. `need:5` means five checkpoints total.

## Reading the client

Same approach as room 1 — `monodis` on `GrandLarcenyAuto.dll`. The `PoPClient` class contained the whole protocol:

The signing key is just a hardcoded string in the static constructor:

```csharp
SignKey = UTF8.GetBytes("gla2_crew_sign_v1_2f9b6c8ad14e")
```

`Sign(msg)`:

```csharp
sig = HMACSHA256(SignKey, UTF8(msg)).ToHex()   // lowercase hex
```

`ReportCheckpoint(step)` builds the signed message as:

```csharp
msg  = sessionId + "|" + step + "|" + token
body = {"session_id": sid, "step": step, "token": token, "sig": sig}
```

And there was a `DeriveStaffRole()`:

```csharp
role = SHA1("heat5_stash{order[0]}_stash{order[1]}_stash{order[2]}_vault").ToHex()
```

The `stash_order` comes from the session and changes every time, so the order of `stash` steps matters. The five steps are:

`heat5` -> `stash{order[0]}` -> `stash{order[1]}` -> `stash{order[2]}` -> `vault`

One more detail that bit me: each successful checkpoint returns a **new** token, and the next request has to use it. And the server rate-limits — if you fire them too fast it replies `too_fast` and wants ~6 seconds between calls.

## The exploit

Reimplementing it in Python:

```python
import hmac, hashlib, json, urllib.request, time

KEY  = b"gla2_crew_sign_v1_2f9b6c8ad14e"
BASE = "http://10.129.182.9"

def sign(msg):
    return hmac.new(KEY, msg.encode(), hashlib.sha256).hexdigest()

# 1) new session
sess  = post("/session", {})
sid   = sess["session_id"]
order = sess["stash_order"]

# 2) walk the checkpoints; each one rotates the token
steps = ["heat5"] + [f"stash{i}" for i in order] + ["vault"]
for step in steps:
    sig  = sign(f"{sid}|{step}|{tok}")
    resp = post("/checkpoint", {"session_id": sid, "step": step, "token": tok, "sig": sig})
    tok  = resp["token"]          # rotation
    time.sleep(6.5)               # server wants ~6s between calls

# 3) claim with the staff role
role = hashlib.sha1(f"heat5_stash{order[0]}_stash{order[1]}_stash{order[2]}_vault".encode()).hexdigest()
sig  = sign(f"{sid}|claim|{tok}")
resp = post("/claim", {"session_id": sid, "role": role, "token": tok, "sig": sig})
```

## The two answers

Claiming as a normal player (or with any junk role) gives a decoy:

```json
{"flag":"THM{n1c3_dr1v1ng_but_th4ts_th3_wr0ng_v4ult}",
 "tier":"player",
 "note":"civilian access — the real vault is staff-only"}
```

Claiming with the derived staff role gives the real one — and notably it comes back with no `note`, which is exactly what the client treats as "real flag":

```json
{"flag":"THM{Th4ts_th3_wr0ng_g4m3_t0mmy}"}
```

The game client's own logic confirms it: `ShowServerWin` only shows "Staff access granted. That's the real score." when the claim response has no note.

## Takeaways

- `role=player` is a trap; the server picks which flag you get based on the role, and only the staff role matches what it computes.
- Session-scoped tokens that rotate on every step are easy to trip over — grab the new token from each response.
- The rate limit (6s per checkpoint) is just a patience test.
- This room is basically "the flag moved server-side, and the client hands you the signing key."