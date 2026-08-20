# Grand Larceny Auto — TryHackMe Write-up

## The idea

This room gives you a small Godot game that plays like a GTA parody. Somewhere in it there's a "Safehouse Vault" that's supposed to never open. The trick is that the actual game logic isn't in the `.pck` file — it's in the .NET assembly.

## First look

I unzipped the Windows build and looked around:

```
$ file GrandLarcenyAuto.exe
PE32+ executable (GUI) x86-64, for MS Windows, 12 sections

$ du -h GrandLarcenyAuto.pck
4,0K    GrandLarcenyAuto.pck
```

A 4KB `.pck` for a whole game is basically nothing. That told me the real code was elsewhere. Sure enough, in `data_GrandLarcenyAuto_windows_x86_64/` there was a `GrandLarcenyAuto.dll` (about 92KB) next to `GodotSharp.dll`. It's a .NET 8 self-contained build.

## Poking at the DLL

A quick `strings` pass over the assembly pointed straight at the interesting stuff:

```
SealedBlob
SafehouseVault
UnlockStars
MaxStars
get_WantedStars
DeriveKey
```

So there's a vault, a sealed blob, and a key derived from something. Time to disassemble properly.

`monodis` dumped the IL nicely:

```
$ monodis data_GrandLarcenyAuto_windows_x86_64/GrandLarcenyAuto.dll > gla.il
```

Two classes mattered.

`CryptoUtil` had:

```csharp
Salt = "GLA::vault::key::v1::stars="
DeriveKey(stars) = SHA256(UTF8(Salt + stars))   // used as the XOR key
```

The methods were wrapped in control-flow-flattening obfuscation (the classic giant `switch` loops with junk arithmetic), but once you trace through it, the logic is plain.

`SafehouseVault.TryOpen()` boils down to:

```csharp
if (player.WantedStars < 6)
    return "The vault stays shut...";          // lock
key   = CryptoUtil.DeriveKey(player.WantedStars);
plain = CryptoUtil.Xor(SealedBlob, key);
return UTF8.GetString(plain);                  // VAULT UNSEALED + message
```

Here's the fun part: the check accepts **6 stars or more**, but the key is built from the **exact** star count. The blob was encrypted with the key for exactly 6 stars. So getting 7+ stars passes the check but produces garbage when it decrypts. You need precisely 6.

## Getting the flag

I didn't want to play the game. Easier: load the DLL in a small .NET console app and call the vault directly via reflection.

```csharp
using GrandLarcenyAuto;

foreach (var stars in new[] { 0, 5, 6, 7, 10 })
{
    var player = new PlayerState { WantedStars = stars };
    var vault = new SafehouseVault(player);
    Console.WriteLine($"stars={stars} =>");
    Console.WriteLine(vault.TryOpen());
}
```

Run it:

```
stars=5 => The vault stays shut...
stars=6 => VAULT UNSEALED
THM{h0tf1x3d_my_0wn_w4nt3d_l3v3l}
stars=7 => VAULT UNSEALED
��|�]��Ʒ...          (garbage)
```

Exactly as expected: 6 stars, real flag.

## Takeaways

- When a Godot `.pck` is tiny, the meat is probably in the .NET DLL next to it.
- Control-flow flattening looks scary in IL but the switches are just a mess of jumps; the real path is usually short.
- You don't need to run the game at all — .NET assemblies are happy to let you instantiate their classes and call methods directly. That's the whole solve here.