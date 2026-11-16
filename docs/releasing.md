# Publishing a release

A release is what players install from: `Hookshot-Setup.exe` and the list it reads, `hookshot-setup.txt`. Both are made by the
**Windows** workflow (`.github/workflows/windows.yml`) and attached to the tag's release; the parts the list names go to one
release of their own, `builder-parts`. No game data is involved at any step.

## Each release

1. Set the version in `launcher/Hookshot.csproj` (`<Version>`), commit, and push to `main`. The workflow runs on every push and
   leaves its files as an artifact (`Hookshot-windows`): a failed run here is found before the tag.
2. Tag the commit `vX.Y.Z` and push the tag.
3. The workflow builds the package again for the tag, uploads the new parts to `builder-parts`, and creates the release with
   `Hookshot-Setup.exe` and `hookshot-setup.txt`. A tag with `-rc` is a prerelease and never "latest".

Players' launchers then offer the update: they compare the game folder's version with the latest release's tag.

The release also gets `ProjectHookshot-vX.Y.Z-Windows-x64.zip`: every part of `hookshot-setup.txt` in one zip, laid out as the
setup installs them, with `recomplauncher.json` beside `Hookshot.exe` (`scripts/release_parts.py zip`). RecompLauncher installs
from it, runs `nx2/make_game.py --game <your copy> --out <its folder>\game --progress-lines` as its setup step and starts
`game\bin\hookshot.exe`. Keep `recomplauncher.json` in step when a key of `hookshot_options.ini` changes; check it against
https://recomplauncher.com/schema/recomplauncher.json.

## The seed: the first release, new tools, a new play profile

The setup installs six parts. Three are made from this repository at every release: the launcher with the sources and the
play profile (`app`), .NET's runtime (`dotnet`) and `third_party`. llvm-mingw, the compiler, comes from its own official
release. The other two are tools that change rarely and are not rebuilt in CI: Python with the build's modules (`python`)
and the Vulkan headers with FFmpeg's libraries (`cpm`). Each release takes those two, and the play profile, from the
release before it.

SDL's two libraries (`SDL2.dll` and `SDL3.dll`, beside the game: the controllers that are not Xbox pads, the microphones)
are not in the repository either. `scripts/release_parts.py package` puts them in the `third_party` part: from the release
before when it had them, else from SDL's own releases (the versions `third_party/sdl2/VERSION.txt` names), checked by
size and SHA-256. Version 0.6.1 was made before that and has none.

A repository with no release yet has nothing to take them from, and a new play profile or a new Python has to come from
somewhere too. That is the seed, made on the maintainer's PC by `scripts/make_seed.py` from three folders:

- `--python`: python.org's embeddable package for Windows (3.12), unpacked, with `Lib\site-packages` and `import site` added
  to its `python312._pth`, and `pip install --target <folder>\Lib\site-packages lz4 numpy capstone "unicorn>=2.1,<2.2"
  zstandard` (delete the `bin` folder pip adds: its launchers hold the path of the Python that ran pip).
- `--cpm`: from a CPM cache, `vulkanheaders/<version>` and `spirv-headers/<version>` (their `include` and license files),
  `vulkanmemoryallocator/<version>` (`include`, license) and `ffmpeg/<version>` (`include`, and `avcodec`, `avutil`,
  `swresample` and `swscale` of `lib`; the build used so far is Takoroka/ffmpeg-av1's 9.0.2-av1-1 for Windows).
- `--work`: the play profile, `hot_funcs.txt`, `hookshot_pieces.prof` and `image/chunks.txt` of the build folder the profile
  was made in. Addresses and counts; no game code. `hookshot_vk.profdata`, the Vulkan library's own profile: it has to be made
  with llvm-mingw, the compiler players build with (a profile from another compiler's build names the C++ functions
  differently and fits nothing); without it the library is built with no profile and the heaviest views lose 3 to 4
  frames a second. And `pipeline_states.bin`, a copy of `hookshot_pipeline_states.bin` from a
  game folder that was played through many places: the pipelines the game asked for there (hashes, formats and draw state).
  A player's first start builds them before the title, from the player's own shaders; without it every place stutters on
  its first visit while the driver compiles. And `computes.bin`, a copy of `hookshot_computes.bin` from the same folder: the
  compute programs play used, 8 bytes each (a hash of the program's code, the same on every PC and in every build).
  `nx2/make_game.py` installs it beside the game as `hookshot_computes_shipped.bin`, and every lean start (the default) builds
  those compute pipelines before the title; without it a new install builds each at its first dispatch, and leaves that work out of its
  first frames while the driver compiles. `make_seed.py` says how many entries each list holds, and stops on a
  `computes.bin` whose size is not a multiple of 8.

It writes three zips and `hookshot-seed.txt`. Upload the four files to the `builder-parts` release (a prerelease, made by hand
the first time), then set the repository variable `SEED_LIST` (Settings, Secrets and variables, Actions, Variables) to
`hookshot-seed.txt`. While the variable is set, the workflow starts from the seed instead of the latest release's list.
Remove it once a release made from the seed is out: the next releases then chain from that one.

The repository must be public for players to download a release's files, and for the workflow to read the seed.

A new play profile file (a new `pipeline_states.bin` or `computes.bin`, a new `hookshot_vk.profdata`) only reaches players
through a seed: the releases' lists carry the profile on from the seed or release before, and nothing in CI reads it from
this repository. Put the file in the seed's `--work` folder and run `make_seed.py` again with the same `--python` and `--cpm`
folders. While those folders and the Python that runs the script are unchanged, their zips come out the same, under the
same names (`hookshot-seed.txt` shows it), so only the new `Hookshot-profile-<sha>.zip` and `hookshot-seed.txt` (which replaces
the old one) go to `builder-parts`. While `SEED_LIST` names `hookshot-seed.txt`, the next
tag starts from it. The workflow's "Play profile" step prints what the profile gives a first start
(`scripts/release_parts.py profile`: the number of pipelines and compute programs, or that a list is missing).

## Checking a release before it is the latest

`Hookshot-Setup.exe` reads two environment variables, for tests:

- `HOOKSHOT_SETUP_LIST=<url>`: another list than the latest release's.
- `HOOKSHOT_SETUP_BASE=<folder>`: install there instead of `%LOCALAPPDATA%\Hookshot`.

The launcher has its own: `HOOKSHOT_LATEST_TAG=<tag>` (or `none`) stands in for GitHub's answer, `HOOKSHOT_NO_SEARCH=1` skips the
search for `.nup` files, `HOOKSHOT_LANG=fr|en` chooses its language, and
`Hookshot.exe --snapshot out.png --view install|build|update|ready|problem|home|home-update|smooth|game|controls|playing`
writes a picture of one page without running anything.
