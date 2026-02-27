---
id: custom-install
title: Custom Installations
sidebar_label: Custom Install
---

# Custom Installations

You will not always find the software you need in the ACE module system. Administrators install packages that are widely used across many users. Specialist research tools — maintained by small academic groups, tied to specific workflows, or simply too new to have been packaged — need to be compiled and installed by you, in your own home directory, without root access.

This guide demonstrates the full process using **HipSTR** (Haplotype inference and phasing for Short Tandem Repeats) as a concrete example. HipSTR is a C++ application with a non-trivial dependency (HTSlib) that itself must be compiled from source. Working through this example teaches every technique you will need to install nearly any open-source scientific software on the cluster:

- Setting up a personal install prefix
- Managing `PATH` and `LD_LIBRARY_PATH`
- Loading a compiler module
- Resolving and building missing dependencies
- Compiling and installing the target software

---

## Prerequisites and Assumptions

- You do not have root access — this guide does not use `sudo` at any point
- The cluster has a module system (`ml` or `module load`)
- `git` and `wget` are available on the login node
- You have basic familiarity with the Linux terminal (see [Introduction to Linux](../tutorials/linux-basics/linux-intro) if not)

---

## Why Install Software Yourself?

On well-managed HPC systems you cannot use `sudo`. Administrators install only widely-used packages. You may need a newer version, a personal tool, or something that is not available on the cluster at all.

The good news is that most open-source scientific software supports user-level installation. HipSTR is no exception — you can compile and run a fully functional installation entirely within your home directory.

---

## Choose a Personal Install Location

A standard, safe convention is to use `$HOME/.local/` as your personal prefix. Create the directories that will hold your compiled binaries, libraries, headers, and source trees:

```bash
mkdir -p ~/.local/src
mkdir -p ~/.local/bin
mkdir -p ~/.local/lib
mkdir -p ~/.local/include
```

Make these directories visible to your shell by adding them to the relevant environment variables. Open `~/.bashrc` with `nano`:

```bash
nano ~/.bashrc
```

Add these two lines at the bottom of the file:

```bash
export PATH="$HOME/.local/bin:$PATH"
export LD_LIBRARY_PATH="$HOME/.local/lib:$LD_LIBRARY_PATH"
```

Save and exit (`Ctrl+O`, then `Ctrl+X`), then reload the shell:

```bash
source ~/.bashrc
```

Any binary you install to `~/.local/bin` will now be found automatically when you type its name.

---

## Load the Compiler Toolchain

HipSTR is written in C++11 and requires `g++`. Load the GCC module:

```bash
ml gnu14
```

Verify the compiler is available:

```bash
g++ --version
```

You should see output like `g++ (GCC) 14.x.x`.

---

## Download the HipSTR Source

Navigate to your personal source directory and clone the HipSTR repository:

```bash
cd ~/.local/src
git clone https://github.com/HipSTR-Tool/HipSTR.git
cd HipSTR
```

HipSTR's build system (its `Makefile`) expects HTSlib to be present as a subdirectory named `htslib` inside the HipSTR source tree. Do not move or rename this directory once it is created.

---

## Build HTSlib

HTSlib is the only build dependency that must be compiled from source. It provides the library functions HipSTR uses to read and write BAM, SAM, and CRAM files.

Download the HTSlib 1.21 release tarball directly into the HipSTR directory, then extract and rename it so the directory is called `htslib`, which is what the HipSTR Makefile looks for:

```bash
wget https://github.com/samtools/htslib/releases/download/1.21/htslib-1.21.tar.bz2
tar -xjf htslib-1.21.tar.bz2
mv htslib-1.21 htslib
cd htslib
```

Configure HTSlib to install its headers and shared library into your personal prefix:

```bash
./configure --prefix=$HOME/.local
```

If configure completes without errors, skip ahead to the build step. If you see an error, follow the relevant fix below before continuing.

### If configure fails: missing libbz2

```
configure: error: libbz2 development files not found
```

The bzip2 development library is not available as a system library. Install it locally:

```bash
cd ~/.local/src
wget https://sourceware.org/pub/bzip2/bzip2-1.0.8.tar.gz
tar -xzf bzip2-1.0.8.tar.gz
cd bzip2-1.0.8

make -f Makefile-libbz2_so
make install PREFIX=$HOME/.local
```

This installs `libbz2.so` into `~/.local/lib` and its headers into `~/.local/include`. Return to the HTSlib directory and re-run configure, this time passing the locations of your local headers and libraries:

```bash
cd ~/.local/src/HipSTR/htslib
./configure --prefix=$HOME/.local \
    CPPFLAGS="-I$HOME/.local/include" \
    LDFLAGS="-L$HOME/.local/lib"
```

### If configure fails: missing liblzma

```
configure: error: liblzma development files not found
```

Install the XZ utils library:

```bash
cd ~/.local/src
wget https://tukaani.org/xz/xz-5.4.5.tar.gz
tar -xzf xz-5.4.5.tar.gz
cd xz-5.4.5

./configure --prefix=$HOME/.local
make -j4
make install
```

Return to the HTSlib directory and re-run configure with the updated flags:

```bash
cd ~/.local/src/HipSTR/htslib
./configure --prefix=$HOME/.local \
    CPPFLAGS="-I$HOME/.local/include" \
    LDFLAGS="-L$HOME/.local/lib"
```

### Build and install HTSlib

Once configure finishes cleanly, compile and install:

```bash
make -j4
make install
```

This installs:
- `libhts.so` and `libhts.a` into `~/.local/lib`
- HTSlib headers (`hts.h`, `sam.h`, etc.) into `~/.local/include/htslib`

The shared library also stays in the `htslib/` build directory. HipSTR links against this copy directly, which is what the `-Wl,-rpath` flag in its Makefile is for.

---

## Build HipSTR

Return to the HipSTR source directory:

```bash
cd ~/.local/src/HipSTR
```

Build HipSTR, passing the absolute path to the htslib directory as the `HTSLIB` variable. Using the absolute path is important — it gets embedded into the binary as a runtime library search path (`rpath`), so the binary continues to find `libhts.so` after you copy it elsewhere:

```bash
make HTSLIB=$HOME/.local/src/HipSTR/htslib
```

The `-j4` flag is not used here because HipSTR's Makefile does not declare inter-file dependencies in a way that makes parallel compilation safe by default. The build is fast regardless.

### If make fails: `uint32_t` has not been declared

When building with GCC 14, you may see this error:

```
src/region.h:62:49: error: 'uint32_t' has not been declared
make: *** [Makefile:106: src/region.o] Error 1
```

This is a known compatibility issue. Newer GCC versions are stricter about requiring the `<cstdint>` header to be explicitly included. Patch the file with:

```bash
sed -i '1s/^/#include <cstdint>\n/' src/region.h
```

Then re-run `make`:

```bash
make HTSLIB=$HOME/.local/src/HipSTR/htslib
```

### If make fails: missing `-lcurl` or `-lcrypto`

If `make` fails with a linker error about `-lcurl` or `-lcrypto`, these are optional HTSlib dependencies for accessing remote files over HTTP. Since HPC workflows typically operate on local files, you can disable them. Re-build HTSlib with:

```bash
cd htslib
./configure --prefix=$HOME/.local --disable-libcurl
make -j4
make install
cd ..
```

Then re-build HipSTR with the curl and crypto libraries removed from the link flags:

```bash
make HTSLIB=$HOME/.local/src/HipSTR/htslib \
     LIBS="-L$HOME/.local/src/HipSTR/htslib \
           -Wl,-rpath,$HOME/.local/src/HipSTR/htslib \
           -lhts -lz -lbz2 -llzma -lpthread"
```

---

## Install the Binary

After a successful build, the `HipSTR` binary sits in the source directory. Copy it to your personal `bin` directory:

```bash
cp -r HipSTR ~/.local/bin/
```

Confirm it is on your PATH:

```bash
which HipSTR
```

---

## Test the Installation

```bash
HipSTR --help
```

You should see HipSTR's usage information printed to the terminal, beginning with something like:

```
Usage: HipSTR --bams <list_of_bams> --fasta <genome.fa> --regions <region_file.bed> --str-vcf <output.vcf.gz> [OPTIONS]
```

If instead you see:

```
HipSTR: error while loading shared libraries: libhts.so.3: cannot open shared object file
```

The runtime linker cannot find `libhts.so`. This should not happen if you used the absolute path for `HTSLIB` during the build, but if it does, add the htslib directory to your library path permanently:

```bash
echo 'export LD_LIBRARY_PATH="$HOME/.local/src/HipSTR/htslib:$LD_LIBRARY_PATH"' >> ~/.bashrc
source ~/.bashrc
```

---
