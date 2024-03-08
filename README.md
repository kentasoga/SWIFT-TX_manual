# はじめに
この文章の目的はSWIFT codeを触っているときに発生した問題や、コードの改変に関する記録を残すことです。
したがって、ユーザーマニュアルではありません。
実際、公式のドキュメントがかなり丁寧に書かれているので自前でマニュアルを書く必要は（少なくとともオリジナルコードを動かすだけなら）ないと思います。
ただし、これからコードを改変していくうちに詳細な解説が必要になればマニュアルの役割を果たす側面が出てくるかもしれません。
以下の英語の文章がオリジナルのReadme で、この文章はそれを上書きしています。
編集の都合上、今のところ一番上に表示していますが、真っ先に読むべきものはオリジナルのほうなので、ぜひそちらに目を通してみてください。


# コンパイル&ジョブの投入まで
## 階層
```
~/SWIFT$ tree -L 1
.
├── configure.ac
├── src
└── swift.c


~/SWIFT$ tree -L 2 -d
.
└── src
    ├── black_holes
    ├── chemistry
    ├── cooling
    ├── entropy_floor
    ├── equation_of_state
    ├── extra_io
    ├── feedback
    ├── forcing
    ├── gravity
    ├── hydro
    ├── lightcone
    ├── mhd
    ├── neutrino
    ├── potential
    ├── pressure_floor
    ├── riemann
    ├── rt
    ├── sink
    ├── star_formation
    ├── stars
    └── tracers

```


## 前提: サブグリッドモデルの指定方法
SWIFT codeは configure するときにオプションを指定することで、どの流体スキームやフィードバックモデルを使うかを切り替えています。
例えば、`./configure --with-subgrid=EAGLE`を叩くと、EAGLEモデルを使用するMakefile が作られます。
この場合は、星形成、BHなどを全てEAGLEモデルを使うことを指定しますが、どうやらサブグリッドごとに指定もできるようです。  
例：`./configure --with_subgrid_star_formation=EAGLE --with_subgrid_black_holes=SPIN_JET`  
ただし、一部のサブグリッドモデル間に依存関係があるようで、完全に自由な組み合わせで選べるかは現状、よくわかっていません。

## コンフィグコマンド
`./configure --with-metis=/home/soga/local --with-tbbmalloc --enable-stand-alone-fof --with-subgrid=TSUKUBA`


## ジョブスクリプト
```
#!/bin/bash
#SBATCH -J swift-tx                                                     # name of run
#SBATCH -p defq                                                         # partition name
#SBATCH --nodes=1             # number of nodes, set to SLURM_JOB_NUM_NODES
#SBATCH --ntasks=36            # number of total MPI processes, set to SLURM_NTASKS
#SBATCH -t 30-00:00:00                          # calculation time
#SBATCH -o stdout
#SBATCH -e stderr

PROG=./swift
LOG=log
PARAM_FILE=swift_tx.yml
export NUM_THREADS=${SLURM_NTASKS}
cd $SLURM_SUBMIT_DIR

${PROG} --tsukuba --cosmology --threads=${NUM_THREADS} ${PARAM_FILE} > ${LOG}
```

# コードの改変方法


## 改変用ディレクトリの作成
今後の拡張のため、EAGLEモデルのディレクトリをほぼコピーしてTSUKUBAディレクトリを作成しました。
これにより、TSUKUBAディレクトリ内のソースコードを編集することでオリジナルのモデルを入れられるはずです。
現状、TSUKUBAディレクトリを作ったのは、以下の６つです。
black_holes/ chemistry/ cooling/ feedback/ star_formation/ stasrs/

以下では、star_formation の場合の具体的な作業を解説します。
### EAGLE ディレクトリをコピー
`cp - r src/star_formation/EAGLE src/star_formation/TSUKUBA`

### TSUKUBAディレクトリ内のヘッダーファイルを適宜編集
変数名が”EAGLEなんとか”みたいなやつを、とりあえず”TSUKUBAなんとか”にしておく



### src/ ディレクトリのファイルの編集
star_formation.h, star_formation_io.h, 等のファイルにTSUKUBAディレクトリ内のヘッダーを include したり、マクロの調整を行ってください。  
例：
```star_formation.h
......
#elif defined(STAR_FORMATION_EAGLE)
#define swift_star_formation_model_creates_stars 1
#include "./star_formation/EAGLE/star_formation.h"
#elif defined(STAR_FORMATION_TSUKUBA)
#define swift_star_formation_model_creates_stars 1
#include "./star_formation/TSUKUBA/star_formation.h"
```

> [!CAUTION]
> いくつかのサブグリッドモデルでは、そのサブグリッドモデル名を含まないソースコードの編集が必要です。  
> 例えば、black_holes の場合、src/ 内の black_holes.h  black_holes_debug.h  black_holes_iact.h  black_holes_io.h  black_holes_properties.h  black_holes_struct.h だけでなく、part.h の編集が必要です。  
> これは、part.h 内にブラックホールサブグリッドの関するマクロ変数 "BLACK_HOLES_*" が定義されているためです。
> 編集し忘れがないように変数名で検索することをおすすめします：`grep BLACK_HOLES *`



### ヘッダーファイルを src/Makefile.am に追記
具体的なやり方は公式ドキュメントに従えばほぼできるはずです。ただし、一部のサブグリッドに関しては追加でいじる必要がある箇所が複数あります。（編集が必要な場所や変数にはEAGLEの文字列が入っているので検索して愚直に直しました。）
```Makefile.am
nobase_noinst_HEADERS += star_formation/TSUKUBA/star_formation.h star_formation/TSUKUBA/star_formation_struct.h
nobase_noinst_HEADERS += star_formation/TSUKUBA/star_formation_io.h star_formation/TSUKUBA/star_formation_iact.h
nobase_noinst_HEADERS += star_formation/TSUKUBA/star_formation_debug.h
```



### configure.ac の編集
"with_star_formation"という変数の下に case文でどのフィードバックモデルを使うかの場合分けがされています。そこをTSUKUBA用に編集します。今回はEAGLEのほぼコピーです。
```M4
#  Star formation
AC_ARG_WITH([star-formation],
    [AS_HELP_STRING([--with-star-formation=<sfm>],
       [star formation @<:@none, QLA, EAGLE, GEAR, TSUKUBA, default: none@:>@]
    )],
    [with_star_formation="$withval"],
    [with_star_formation="none"]
)
if test "$with_subgrid" != "none"; then
   if test "$with_star_formation" != "none"; then
      AC_MSG_ERROR([Cannot provide with-subgrid and with-star-formation together])
   else
      with_star_formation="$with_subgrid_star_formation"
   fi
fi

case "$with_star_formation" in
   none)
      AC_DEFINE([STAR_FORMATION_NONE], [1], [No star formation])
   ;;
   QLA)
      AC_DEFINE([STAR_FORMATION_QLA], [1], [Quick Lyman-alpha star formation model)])
   ;;
   EAGLE)
      AC_DEFINE([STAR_FORMATION_EAGLE], [1], [EAGLE star formation model (Schaye and Dalla Vecchia (2008))])
   ;;
   GEAR)
      AC_DEFINE([STAR_FORMATION_GEAR], [1], [GEAR star formation model (Revaz and Jablonka (2018))])
   ;;
   TSUKUBA)
      AC_DEFINE([STAR_FORMATION_TSUKUBA], [1], [TSUKUBA star formation model (Schaye and Dalla Vecchia (2008))])
   ;;
   *)
      AC_MSG_ERROR([Unknown star formation model])
   ;;
esac
```
> [!CAUTION]
> coolingとfeedbackに関しては以下の追加の編集が必要です。
> ```configure.ac
> # Check if using TSUKUBA cooling
> AM_CONDITIONAL([HAVETSUKUBACOOLING], [test "$with_cooling" = "TSUKUBA"])
>
> # Check if using TSUKUBA feedback
> AM_CONDITIONAL([HAVETSUKUBATHERMALFEEDBACK], [test "${with_feedback%-thermal}" = "TSUKUBA"])
> AM_CONDITIONAL([HAVETSUKUBAKINETICFEEDBACK], [test "$with_feedback" = "TSUKUBA-kinetic"])
> ```




### パラメータファイルの設定
ここまで行うと star_formation に TSKUBAモデルが使えるようになります。あとはパラメータファイル("***.yml")に必要なパラメータ値を指定するだけです。
星形成モデルに関するパラメータファイルの中身は以下の通りです。
```
# TSUKUBA star formation parameters
TSUKUBAStarFormation:
  SF_threshold:                      Zdep         # Zdep (Schaye 2004) or Subgrid
  SF_model:                          PressureLaw  # PressureLaw (Schaye et al. 2008) or SchmidtLaw
  KS_normalisation:                  1.515e-4     # The normalization of the Kennicutt-Schmidt law in Msun / kpc^2 / yr.
  KS_exponent:                       1.4          # The exponent of the Kennicutt-Schmidt law.
  ......
  ......
```

## 新たなソースコードを追加する場合
簡単のため、"Hello, world." を出力する hello関数を定義した hello.c とヘッダー hello.h をSWIFTに追加する場合を考えます。  

```hello.c
/* Config parameters. */
#include <config.h>

/* MPI headers. */
#ifdef WITH_MPI
#include <mpi.h>
#ifdef HAVE_METIS
#include <metis.h>
#endif
#ifdef HAVE_PARMETIS
#include <parmetis.h>
#endif
#endif

#ifdef HAVE_HDF5
#include <hdf5.h>
#endif

#ifdef HAVE_FFTW
#include <fftw3.h>
#endif

#ifdef HAVE_LIBGSL
#include <gsl/gsl_version.h>
#endif

#ifdef HAVE_SUNDIALS
#include <sundials/sundials_version.h>
#endif

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include "hello.h"

void hello(void) {

	printf ("Hello, world!\n");

}
```

```hello.h
#ifndef SWIFT_HELLO_H
#define SWIFT_HELLO_H

void hello(void);

#endif /* SWIFT_HELLO_H */
```

ソースを置くべきディレクトリは src/ です。  
編集すべきファイルは、
- src/Makefile.am
- src/swift.h

の2つです。  
まず、Makefile.am のAM_SOURCESとnobase_noinst_HEADERSを編集します。  
``` Makefile.am
AM_SOURCES += hello.c
nobase_noinst_HEADERS += hello.h
```

次に swift.h に #include "hello.h" を追加します。  
あとは、swift.c のmain関数の好きなところにhello関数を追加すると、無事出力されます。




<a name="logo"/>
<div align="center">
<a href="https://www.swiftsim.com/" target="_blank">
<img src="https://swift.strw.leidenuniv.nl/SWIFT_banner.jpg" alt="SWIFT banner" width="1016" height="242"></img>
</a>
</div>

SWIFT: SPH WIth Fine-grained inter-dependent Tasking
====================================================

[![Build Status](https://gitlab.cosma.dur.ac.uk/jenkins/job/GNU%20SWIFT%20build/badge/icon)](https://gitlab.cosma.dur.ac.uk/jenkins/job/GNU%20SWIFT%20build/)

SWIFT is a gravity and SPH solver designed to run cosmological simulations
on peta-scale machines, scaling well up to 10's of thousands of compute
node.

More general information about SWIFT is available on the project
[webpages](http://www.swiftsim.com).

For information on how to _run_ SWIFT, please consult the onboarding guide
available [here](https://swift.strw.leidenuniv.nl/onboarding.pdf). This includes
dependencies, and a few examples to get you going.

We suggest that you use the latest release branch of SWIFT, rather than the
current master branch as this will change rapidly. We do, however, like to
ensure that the master branch will build and run.

This GitHub repository is designed to be an issue tracker, and a space for
the public to submit patches through pull requests. It is synchronised with
the main development repository that is available on the
[ICC](http://icc.dur.ac.uk)'s GitLab server which is available
[here](https://gitlab.cosma.dur.ac.uk/swift/swiftsim).

Please feel free to submit issues to this repository, or even pull
requests. We will try to deal with them as soon as possible, but as the
core development team is quite small this could take some time.

Disclaimer
----------

We would like to emphasise that SWIFT comes without any warranty of accuracy,
correctness or efficiency. As mentioned in the license, the software comes
`as-is` and the onus is on the user to get meaningful results. Whilst the
authors will endeavour to answer questions related to using the code, we
recommend users build and maintain their own copies. This documentation contains
the most basic information to get started. Reading it and possibly also the
source code is the best way to start running simulations.

The users are responsible to understand what the code is doing and for the
results of their simulation runs.

Note also that the values of the parameters given in the examples are only
indicative. We recommend users experiment by themselves and a campaign of
experimentation with various values is highly encouraged. Each problem will
likely require different values and the sensitivity to the details of the
physical model is something left to the users to explore.

Acknowledgment & Citation
-------------------------

The SWIFT code was last described in this paper:
https://ui.adsabs.harvard.edu/abs/2023arXiv230513380S.  The core solver, the
numerical methods as well as many extensions where described there. We ask users
running SWIFT for their research to please cite this paper when they present
their results.

In order to keep track of usage and measure the impact of the software, we
kindly ask users publishing scientific results using SWIFT to add the following
sentence to the acknowledgment section of their papers:

"The research in this paper made use of the SWIFT open-source
simulation code (http://www.swiftsim.com, Schaller et al. 2018)
version X.Y.Z."

with the version number set to the version used for the simulations and the
reference pointing to the ASCL entry of the code: https://ascl.net/1805.020.



Contribution Guidelines
-----------------------

The SWIFT source code uses a variation of the 'Google' formatting style.
The script 'format.sh' in the root directory applies the clang-format-10
tool with our style choices to all the SWIFT C source file. Please apply
the formatting script to the files before submitting a pull request.

Please check that the test suite still runs with your changes applied before
submitting a pull request and add relevant unit tests probing the correctness
of new modules. An example of how to add a test to the suite can be found by
considering the tests/testGreeting case.

Any contributions that fail any of the automated tests will not be accepted.
Contributions that include tests of the proposed modules (or any current ones!)
are highly encouraged.

Runtime parameters
------------------

```
 Welcome to the cosmological hydrodynamical code
    ______       _________________
   / ___/ |     / /  _/ ___/_  __/
   \__ \| | /| / // // /_   / /
  ___/ /| |/ |/ // // __/  / /
 /____/ |__/|__/___/_/    /_/
 SPH With Inter-dependent Fine-grained Tasking

 Version : 1.0.0
 Website: www.swiftsim.com
 Twitter: @SwiftSimulation

See INSTALL.swift for install instructions.

Usage: swift [options] [[--] param-file]
   or: swift [options] param-file
   or: swift_mpi [options] [[--] param-file]
   or: swift_mpi [options] param-file

Parameters:

    -h, --help                        show this help message and exit

  Simulation options:

    -b, --feedback                    Run with stars feedback.
    -c, --cosmology                   Run with cosmological time integration.
    --temperature                     Run with temperature calculation.
    -C, --cooling                     Run with cooling (also switches on --temperature).
    -D, --drift-all                   Always drift all particles even the ones
                                      far from active particles. This emulates
                                      Gadget-[23] and GIZMO's default behaviours.
    -F, --star-formation              Run with star formation.
    -g, --external-gravity            Run with an external gravitational potential.
    -G, --self-gravity                Run with self-gravity.
    -M, --multipole-reconstruction    Reconstruct the multipoles every time-step.
    -s, --hydro                       Run with hydrodynamics.
    -S, --stars                       Run with stars.
    -B, --black-holes                 Run with black holes.
    -k, --sinks                       Run with sink particles.
    -u, --fof                         Run Friends-of-Friends algorithm to
                                      perform black hole seeding.
    --lightcone                       Generate lightcone outputs.
    -x, --velociraptor                Run with structure finding.
    --line-of-sight                   Run with line-of-sight outputs.
    --limiter                         Run with time-step limiter.
    --sync                            Run with time-step synchronization
                                      of particles hit by feedback events.
    --csds                            Run with the Continuous Simulation Data
                                      Stream (CSDS).
    -R, --radiation                   Run with radiative transfer.
    --power                           Run with power spectrum outputs.

  Simulation meta-options:

    --quick-lyman-alpha               Run with all the options needed for the
                                      quick Lyman-alpha model. This is equivalent
                                      to --hydro --self-gravity --stars --star-formation
                                      --cooling.
    --eagle                           Run with all the options needed for the
                                      EAGLE model. This is equivalent to --hydro
                                      --limiter --sync --self-gravity --stars
                                      --star-formation --cooling --feedback
                                      --black-holes --fof.
    --gear                            Run with all the options needed for the
                                      GEAR model. This is equivalent to --hydro
                                      --limiter --sync --self-gravity --stars
                                      --star-formation --cooling --feedback.
    --agora                           Run with all the options needed for the
                                      GEAR model. This is equivalent to --hydro
                                      --limiter --sync --self-gravity --stars
                                      --star-formation --cooling --feedback.
                                      
  Control options:

    -a, --pin                         Pin runners using processor affinity.
    --nointerleave                    Do not interleave memory allocations across
                                      NUMA regions.
    -d, --dry-run                     Dry run. Read the parameter file, allocates
                                      memory but does not read the particles
                                      from ICs. Exits before the start of time
                                      integration. Checks the validity of
                                      parameters and IC files as well as memory
                                      limits.
    -e, --fpe                         Enable floating-point exceptions (debugging
                                      mode).
    -f, --cpu-frequency=<str>         Overwrite the CPU frequency (Hz) to be
                                      used for time measurements.
    -n, --steps=<int>                 Execute a fixed number of time steps.
                                      When unset use the time_end parameter
                                      to stop.
    -o, --output-params=<str>         Generate a parameter file with the options
                                      for selecting the output fields.
    -P, --param=<str>                 Set parameter value, overiding the value
                                      read from the parameter file. Can be used
                                      more than once {sec:par:value}.
    -r, --restart                     Continue using restart files.
    -t, --threads=<int>               The number of task threads to use on each
                                      MPI rank. Defaults to 1 if not specified.
    --pool-threads=<int>              The number of threads to use on each MPI
                                      rank for the threadpool operations.
                                      Defaults to the numbers of task threads
                                      if not specified.
    -T, --timers=<int>                Print timers every time-step.
    -v, --verbose=<int>               Run in verbose mode, in MPI mode 2 outputs
                                      from all ranks.
    -y, --task-dumps=<int>            Time-step frequency at which task graphs
                                      are dumped.
    --cell-dumps=<int>                Time-step frequency at which cell graphs
                                      are dumped.
    -Y, --threadpool-dumps=<int>      Time-step frequency at which threadpool
                                      tasks are dumped.
    --dump-tasks-threshold=<flt>      Fraction of the total step's time spent
                                      in a task to trigger a dump of the task plot
                                      on this step

See the file examples/parameter_example.yml for an example of parameter file.
```
