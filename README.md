# ghidra-ExportDwarfELFSymbols
A format agnostic script to export an ELF file with DWARF symbols from a Ghidra program

This script was heavilly inspired by CeSeNA's [ghidra2dwarf](https://github.com/cesena/ghidra2dwarf) script, but the main difference is their script append informations to an existing ELF while this script is to generate a new one from scratch and figure out the proper format.

## Motivations
This script was made because there currently isn't a good way to work with relatively esotheric debugging targets for ghidra (like console emulators) while being able to have symbols, but also have access to debugging features such as `nexti` (step over). The `nexti` command has trouble to works on these targets (it would act as a `stepi`) because GDB doesn't have enough informations to unwind the stack and after research, it was found that giving minimal symbols to GDB (mainly where functions are) is enough for it to figure out how to decide that a new stack frame was entered

## Features
This script was made to generate the ELF from scratch: it doesn't care how the program is formatted, it simply spit DWARF informations from what Ghidra knows.

Here is how this script adapts the ELF depending on the target:
- 32 bit or 64 bit are supported (will take the proper ELF format variant and assign the header's machine correctly).
- Either endianness (little or big) are supported and will be assigned correctly to the ident of the ELF header.
- The following CPU languages are assigned to the machine field of the ELF header: x86, PowerPC, ARM and MIPS (default to x86 if not from this list).
- The entry point is taken from the first entry point refference in the program.

Currently, only the names, entry points and the ends address of functions are exported. 

## Installation instructions
1. Download the latest zip file from the release page.
2. Open Ghidra. On the main window, go to File → Install Extensions..., click the "+" button, and select the downloaded zip. Restart Ghidra when prompted.
3. Open the Script Manager window and click the button on the top right that says Manage Script Directories. The script should now appear under the DWARF category.

## Usage instructions
1. Launch the script, you will be prompted for an output file. Select a suitable location and click OK.
2. The scripting console will log all interesting events that the script is doing and if everything went accordingly, it should say that it successfully exported the file at the end.
3. You may now load your newly created ELF file to GDB after connecting to your target via the `symbol-file` command.

## Building instructions
> Note, if you just want to use the script, you do not need to do this, this section is for developers

### Prerequisites

- Ghidra installation (path passed via `-DGHIDRA_INSTALL_DIR` — no separate Gradle install needed, the build uses Ghidra's bundled `gradlew`)
- CMake 3.10+ and a C/C++ toolchain

### 1. Fetch the libdwarf submodule

```bash
git submodule update --init
```

### 2. Configure

Pass your Ghidra installation path via `-DGHIDRA_INSTALL_DIR`:

```bash
cmake -S . -B build/cmake-build \
    -DBUILD_SHARED=ON -DBUILD_NON_SHARED=OFF \
    -DGHIDRA_INSTALL_DIR=/path/to/ghidra
```

The value is stored in the CMake cache, so you only need to pass it once.

### 3. Build and package

```bash
cmake --build build/cmake-build
```

This builds the native libdwarf shared library, copies it into `os/<platform>/`, and runs
`gradle distributeExtension` to produce a zip in `dist/`.

### 4. Install into Ghidra

```bash
cmake --install build/cmake-build
```

This copies the zip from `dist/` into `$GHIDRA_INSTALL_DIR/Extensions/Ghidra/`. Restart Ghidra
and the extension will be available.

## License
This script is licensed under the MIT license which grants you the rights to share, modify and distribute this script as long as you mention the original author. For more details, please consult the LICENSE file.

