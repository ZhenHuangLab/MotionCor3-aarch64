# MotionCor3 — Linux aarch64 + CUDA 13 + GB10 (sm_121)

Upstream: https://github.com/czimaginginstitute/MotionCor3

## Result: 🟢 PASS — 4 patches required

Same recipe as the AreTomo2/AreTomo3 ports, plus one CUDA-13-specific header fix.

## Patch 1 — drop the x86-only `-m64` flag (3 makefiles)

| file | before | after |
|---|---|---|
| `LibSrc/Util/makefile` | `CFLAG = -c -g -pthread -m64` | `CFLAG = -c -g -pthread` |
| `LibSrc/Mrcfile/makefile` | `CFLAGS = -m64 -c -g -pthread ...` | `CFLAGS = -c -g -pthread ...` |
| `makefile11` | `CFLAG = -c -g -pthread -m64` | `CFLAG = -c -g -pthread` |

The prebuilt `LibSrc/Lib/libmrcfile.a` and `libutil.a` shipped in the repo are
x86-64 archives and **must be rebuilt**:

```bash
rm -f LibSrc/Lib/*.a
(cd LibSrc/Util    && make clean && make)
(cd LibSrc/Mrcfile && make clean && make)
```

## Patch 2 — `makefile11` paths and CUDA architecture

```makefile
CONDA    = /usr                    # was $(HOME)/miniconda3
CUDAHOME = /usr/local/cuda         # was $(HOME)/nvidia/cuda-12.1

CUFLAG = -Xptxas -dlcm=ca -O2 \
	-gencode arch=compute_121,code=sm_121 \
	-gencode arch=compute_121,code=compute_121
```

(The upstream list of SM 70-90 is replaced outright; CUDA 13 no longer accepts
the older architectures anyway.)

## Patch 3 — `makefile11` link rule

```makefile
exe: $(OBJS)
	@g++ -g -pthread $(OBJS) \
	$(PRJLIB)/libmrcfile.a $(PRJLIB)/libutil.a \
	-L$(CUDALIB) \
	-L$(CONDA)/lib/aarch64-linux-gnu \
	-lcufft -lcudart -lcuda -ltiff -lc -lm -lpthread \
	-o MotionCor3
```

Changes: link with `g++` instead of `nvcc -g -G` (device debug builds are
needlessly slow), drop `-m64` and `-L/usr/lib64`, point at the Debian multiarch
lib dir, and **remove `-lnvToolsExt`** — CUDA 13 no longer ships
`libnvToolsExt.so` (NVTX3 is header-only).

## Patch 4 — NVTX3 include path (23 files)

CUDA 13 removed the top-level `nvToolsExt.h`; the header now lives under
`nvtx3/`.

```bash
grep -rl '#include <nvToolsExt.h>' --include=*.cu --include=*.cpp --include=*.h . \
  | xargs sed -i 's|#include <nvToolsExt.h>|#include <nvtx3/nvToolsExt.h>|'
```

## Build

```bash
make -f makefile11 cuda        # .cu -> .cpp via nvcc
make -f makefile11 compile -j"$(nproc)"
make -f makefile11 exe
```

Result: `MotionCor3`, `ELF 64-bit LSB pie executable, ARM aarch64`.
`cuobjdump --list-elf MotionCor3` shows sm_121 cubins throughout; `ldd` resolves
all libraries.

## Smoke test — motion correction accuracy

1024x1024 x 12-frame synthetic movie: a sharp random texture displaced by a
smooth, exponentially decaying beam-induced-motion-like drift
(x span 5.8 px, y span 2.9 px).

```bash
MotionCor3 -InMrc movie.mrc -OutMrc sum.mrc -Patch 0 0 \
           -Iter 20 -Tol 0.05 -Gpu 0 -PixSize 1.0 -OutAln . -FmDose 0
```

| metric | value |
|---|---|
| wall time | 0.73 s |
| **RMS shift error** | **0.113 px** |
| max per-frame error | 0.206 px |

Patch-based alignment (`-Patch 3 3`) also converges (per-patch iteration error
0.065) and the built-in CTF estimator runs.

## Caveat when writing your own test

MotionCor fits a *smooth* motion model to the global trajectory. Validating it
against a pure random-walk drift gives an apparent ~1.5 px RMS error that is the
model's smoothing, not a numerical fault. Use a smooth drift for verification.
