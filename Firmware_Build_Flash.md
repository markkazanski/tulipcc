# AMYboard Firmware Build and Flash

The next step is to build the AMYboard firmware from your fork and flash it over USB. Your Windows machine has no ESP-IDF, so the easiest route is to let your fork's GitHub Actions build it. Your fork already contains the official `AMYboard release` workflow.

### 1. Commit and push to a branch, not `main`
The changes are still uncommitted in `C:\PROJECTS\AmyBoard\FORK\tulipcc`.

```powershell
cd C:\PROJECTS\AmyBoard\FORK\tulipcc
git checkout -b juno-soft-clip
git add tulip/shared/amy_connector.c tulip/shared/user_c_dsp.c tulip/shared/user_c_dsp_web.js docs/user_c_dsp.md
git commit -m "Add output soft clip on bus 0 after FX chain"
git push -u origin juno-soft-clip
```

Use a branch because the workflow only publishes a release when it runs on `main`. Run from a branch, it's a dry run that just builds and uploads the files for you to download.

### 2. Build it on GitHub
1. On github.com/markkazanski/tulipcc, open **Actions**. Forks have Actions turned off by default, so enable them if asked.
2. Pick **AMYboard release**, click **Run workflow**, and choose `juno-soft-clip`.
3. When the `firmware` job finishes, download the **amyboard-firmware** artifact. It's only kept for 2 days. It contains `amyboard-firmware-AMYBOARD.bin`, `amyboard-full-AMYBOARD.bin` and `amyboard-sys.bin`.

The `web` job will fail because your fork has no `VERCEL_TOKEN`, and the VCV Rack jobs may fail too. Ignore both; the `firmware` job doesn't depend on them.

### 3. Flash it
The amyboard.com flasher always downloads shorepine's official release, so it can't flash your build. Use `esptool` instead:

```powershell
pip install esptool
```

Put the board in download mode: hold the BOOT button while plugging in USB. Then find its COM port in Device Manager and pick one of these options.

**Keep your sketches and files.** This writes only the app firmware:

```powershell
python -m esptool --chip esp32s3 -p COM5 erase_region 0xd000 0x2000
python -m esptool --chip esp32s3 -p COM5 write_flash 0x10000 amyboard-firmware-AMYBOARD.bin
```

The board has two firmware slots, `ota_0` at `0x10000` and `ota_1` at `0x3a0000` (see `tulip/amyboard/boards/AMYBOARD/tulip-partitions-16MB.csv`). Erasing the boot-selection data at `0xd000` makes it boot from `ota_0`, where you just wrote the new firmware. Otherwise a board that was upgraded over the air could keep booting the old firmware in `ota_1`.

**Clean full install.** This writes the whole image, including the user file area, so it erases your sketches and files:

```powershell
python -m esptool --chip esp32s3 -p COM5 write_flash 0x0 amyboard-full-AMYBOARD.bin
```

Unplug and replug the board (or press reset) after flashing.

### 4. Listen
Load a Juno patch with chorus II, hold a low six-note chord at full velocity, and check that it rounds off instead of crackling. A quiet single note should be exactly as loud as before.

### Alternative: build locally
Install ESP-IDF v5.4.1 (WSL is the easiest way on Windows). Run `tulip/shared/grab_submodules.sh`, then build in `tulip/amyboard`:

```
idf.py -DMICROPY_BOARD=AMYBOARD build
```

Then run `python fs_create.py amyboard` from `tulip/` to assemble the images. That's the same build the workflow runs, just on your machine.