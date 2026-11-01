# rpcs3-silent-hill-3-hd-boot-fix

An RPCS3 game patch that stops **Silent Hill 3** in the **Silent Hill HD Collection** (US, `BLUS30810`) from
hanging right after it starts.

## The problem

Pick Silent Hill 3 in the HD Collection launcher and it never gets anywhere:

- the screen stays on the HD Collection artwork (sometimes with RPCS3's "Building SPU Cache…" overlay frozen on top)
- no crash, no error, no freeze message: the emulator keeps running at a few busy cores
- Silent Hill 2 from the same launcher boots fine

The log goes quiet right after the trophy step. The last things Silent Hill 3 does are:

```
sys_fs: 'sys_fs_stat' failed with 0x8001003a : CELL_ENOTMOUNTED, "/sh3/data/chr/pl/chhaa.mdl"
cellSysutil: cellMsgDialogClose(delay=100.000000)
SYS: 'cellMsgDialogClose' failed with 0x8002b302 : CELL_MSGDIALOG_ERROR_DIALOG_NOT_OPENED
```

Changing settings doesn't help. All of these hang the same way: SPU interpreter, accurate xfloat, DMA and RSX
reservations, no SPU cache, frame limit off, strict rendering, LLE `libsysutil_game`. It also hangs on older RPCS3
builds (0.0.42) as well as current master (0.0.43), on the retail disc and on a converted digital copy.

## Why

It's a race between two of the game's own threads. RPCS3 just happens to lose it.

Silent Hill 3 looks for each asset in three places, in this order:

1. the patch directory, `/dev_hdd0/game/BLUS30810/USRDIR/patch/sh3/data/…`
2. the HDD install, `/dev_hdd0/game/BLUS30810/USRDIR/sh3/data/…` (only once the install check has finished)
3. the disc, `/dev_bdvd/PS3_GAME/USRDIR/sh3/data/…`

The lookup (`sh3game` `0x1de00`) builds each path with `sprintf("%s/%s/data/%s", root, "sh3", file)` and calls
`cellFsStat`. **If the stat fails with anything other than `CELL_FS_ENOENT`, it calls the game's fatal "media
error" handler** (`0x2dff78`). After that the main thread does nothing but pump `cellSysutilCheckCallback`, waiting
for the user to quit. That's the "hang".

```
1df00:  lis   r4,-32767
1df04:  ori   r4,r4,6         ; 0x80010006 = CELL_FS_ENOENT
1df08:  cmpw  r3,r4
1df0c:  beq   1df1c           ; "not here" -> try the next root
1df10:  mr    r3,r31
1df14:  bl    2dff78          ; anything else -> fatal media error   <- patched to nop
1df18:  nop
1df1c:  li    r25,0           ; "not found"
```

The patch-directory root string is written by the game's "Initialization Thread", after `cellGameBootCheck`,
`cellGamePatchCheck` and `cellGameContentPermit`. RPCS3's HLE versions of those functions sleep on purpose to match
real hardware timing ([`cellGame.cpp`](https://github.com/RPCS3/rpcs3/blob/master/rpcs3/Emu/Cell/Modules/cellGame.cpp):
120 ms in `cellGameDataCheck`, 200 ms in `cellGameContentPermit`, "This function is incredibly slow"). Under the LLVM
recompiler, though, the game's main thread reaches its first asset lookups about 300 ms after starting, while that
string is still empty. So the first root is `""`, the path comes out as `/sh3/data/chr/pl/chhaa.mdl`, and the stat
returns **`CELL_ENOTMOUNTED`** (there's no `/sh3` device) instead of `ENOENT`. That goes straight to the fatal
handler.

How this was confirmed:

- With the **PPU interpreter** the main thread is much slower, the root is set in time, and Silent Hill 3 boots to
  its opening movie. The first lookups then read `/dev_hdd0/game/BLUS30810/USRDIR/patch/sh3/data/…`. Under LLVM
  they read `/sh3/data/…`.
- RPCS3's GDB stub showed the main thread parked in the `cellSysutilCheckCallback` loop, with the lookup's
  registers still holding the empty root string.
- None of the error lines in the log are the actual cause: the `cellMsgDialogClose … DIALOG_NOT_OPENED`, the
  lwmutex `ESRCH` and the `/sh3/…` open failures are all side effects.

## The fix

The patch NOPs the two calls to the fatal handler for the **patch-directory** and **HDD-install** roots (`0x1DF14`,
`0x1DFE8`). An error there now means "not here, try the next root", exactly like `ENOENT`, which is what those
lookups expect anyway. The **disc** check (`0x1E198`) is left alone, so a genuinely unreadable disc file still goes
to the game's error path. Once the root strings are set, a few hundred ms in, nothing changes.

(Pre-filling the root string from a patch doesn't work: the game clears it during its own init after the patch has
been applied.)

## Install

Requirements: **Silent Hill HD Collection, US (`BLUS30810`), with the v01.02 update installed.** The patch keys on
the PPU hash of that update's `sh3game.ppu.self`: `PPU-5dd8d3683fb219af941b921ed2280bfaaa49e30a`.

**Easiest:** in RPCS3 open **Manage → Game Patches**, drag
[`patches/Silent_Hill_HD_Collection_SH3_boot_fix.yml`](patches/Silent_Hill_HD_Collection_SH3_boot_fix.yml) onto the
window to import it, then tick **"Fix Silent Hill 3 hanging after the trophy check (startup race)"** under Silent Hill
HD Collection.

**Manual:**

1. Copy the contents of `patches/Silent_Hill_HD_Collection_SH3_boot_fix.yml` into `patches/imported_patch.yml`. If
   that file already exists, add only the `PPU-5dd8…:` block and don't repeat the `Version:` line. On Linux that's
   `~/.config/rpcs3/patches/`; on Windows it's the `patches` folder inside the RPCS3 folder.
2. Enable it: tick it in Manage → Game Patches, or merge `patches/patch_config.example.yml` into
   `patch_config.yml` (`~/.config/rpcs3/` on Linux; the `config` folder inside the RPCS3 folder on Windows).

When it's working, the log shows this when SH3 starts:

```
PAT: Applied patch (hash='PPU-5dd8d3683fb219af941b921ed2280bfaaa49e30a', description='Fix Silent Hill 3 hanging after the trophy check (startup race)', ...)
```

## Status

- Tested on RPCS3 0.0.43-20550 (Linux AppImage, LLVM): Silent Hill 3 gets past the hang, starts its sound
  loader and starts the opening movie (opens `movie/op.bik` and spins up its Bink threads) about 12 s after launch.
  It used to hang at about 4 s. Also confirmed in normal play: launcher → Silent Hill 3 → the game runs.
- Silent Hill 2 doesn't need it; the patch only touches `sh3game`.
- **Not covered:** the disc's own v01.00 `sh3game` (a different build, so a different hash and possibly different
  addresses) and the EU release (`BLES01504`). Both very likely have the same race. To port the patch, find the same
  pattern in that build's asset lookup: `lis rX,-32767 / ori rX,rX,6 / cmpw / beq / mr r3,r31 / bl <fatal>` after a
  `cellFsStat` on a `"%s/%s/data/%s"` path. Then NOP the first two `bl`s. PRs welcome.

## License

MIT
