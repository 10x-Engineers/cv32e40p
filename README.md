


<!-- ABOUT THE PROJECT -->
## About The Project


This project presents an open-source design of a low-cost, modular BFloat16 (BF16) Floating Point Unit (FPU) micro-architecture. The design targets RISC-V based embedded cores and is integrated with the [CV32E40P](https://github.com/openhwgroup/cv32e40p/tree/master) core. The RTL for the FPU is present inside rtl/vendor/bf16_acc.

The BF16 FPU includes custom instructions for the following operations:
* **Add:** Perform a BF16 floating-point addition.
* **Subtract:** Perform a BF16 floating-point subtraction.
* **Maximum:** Find the maximum of two BF16 floating-point numbers.
* **Minimum:** Find the minimum of two BF16 floating-point numbers.
* **Bf16 to Fp32 conversion:** Convert a BF16 floating-point number to FP32 format.
* **Fp32 to Bf16 conversion:** Convert an FP32 floating-point number to BF16 format.
* **FMADD (Floating point multiply add):** Multiply two BF16 numbers and add a 32-bit accumulation result.
* **FMSUB (Floating point multiply subtract):** Multiply two BF16 numbers and subtract from a 32-bit accumulation result.
* **FNMADD (Floating point negative multiply add):** Multiply two BF16 numbers, negate the result, and add a 32-bit accumulation result.
* **FNMSUB (Floating point negative multiply subtract):** Multiply two BF16 numbers, negate the result, and subtract from a 32-bit accumulation result.

Our design relaxes certain features of the IEEE Floating-Point Standard to realize
a cost-effective hardware implementation achieving almost 35% reduction in silicon
area compared to an IEEE compliant FP32 implementation with minimal impact on computational accuracy.

The following optimizations have been made:
* Zero-flushing for subnormal operands.
* Single rounding mode: Round to Nearest, Ties to Even (RNE).
* No propagation of input NaNs.
* Zero-skipping for sparse networks.

This work was presented in the Risc-V Europe Summit 2024. The paper and poster will be present on the summit [website](https://github.com/openhwgroup/cv32e40p/tree/master) or check references below.



<!-- GETTING STARTED -->
## Getting Started

The gcc toolchain has already been built in the linked gcc toolchain with the custom instructions included. The verification enviroment has been set in core-v-verif with the cv32e40p containing the fpu under the core-v-cores folder in the repository. This has been done for ease of setup.


### Setting up toolchain

You can setup the toolchain using the following steps.

1. Clone the gcc toolchain and build it as directed in the (`Readme`) of the riscV gcc toolchain repository linked [here](https://github.com/riscv-collab/riscv-gnu-toolchain).
2. After building the toolchain, two files need to be updated in order to add the BF16 custom instructions, (`binutils/include/opcode/riscv-opc.h`) and (`binutils/opcodes/riscv-opc.c`).

   Add the following lines in (`riscv-opc.h`)
   ```sh
    #define MATCH_BF16_MAX 0x7c000053
    #define MASK_BF16_MAX 0xfe00707f
    #define MATCH_BF16_MIN 0x6c000053
    #define MASK_BF16_MIN 0xfe00707f
    
    #define MATCH_BF16_FP32_CONV 0x44800053
    #define MASK_BF16_FP32_CONV 0xfff0007f
    #define MATCH_FP32_BF16_CONV 0x40600053
    #define MASK_FP32_BF16_CONV 0xfff0007f
    
    #define MATCH_BF16_ADD 0x400006b
    #define MASK_BF16_ADD 0xfe00007f
    #define MATCH_BF16_SUB 0x400007b
    #define MASK_BF16_SUB 0xfe00007f
    
    #define MATCH_BF16_FMADD 0x400001b
    #define MASK_BF16_FMADD 0x600007f
    #define MATCH_BF16_FMSUB 0x400002b
    #define MASK_BF16_FMSUB 0x600007f
    #define MATCH_BF16_FMNADD 0x400003b
    #define MASK_BF16_FMNADD 0x600007f
    #define MATCH_BF16_FMNSUB 0x400005b
    #define MASK_BF16_FMNSUB 0x600007f
   ```

   Add the following code to (`riscv-opc.c`) under (`const struct riscv_opcode riscv_opcodes[] ={`)
   ```sh
    {"bf16.min",     0, INSN_CLASS_ZFH_INX,   "D,S,T",     MATCH_BF16_MIN, MASK_BF16_MIN, match_opcode, 0 },
    {"bf16.max",     0, INSN_CLASS_ZFH_INX,   "D,S,T",     MATCH_BF16_MAX, MASK_BF16_MAX, match_opcode, 0 },
    {"bf16.fp32.conv",   0, INSN_CLASS_ZFHMIN_INX, "D,S",     MATCH_BF16_FP32_CONV|MASK_RM, MASK_BF16_FP32_CONV|MASK_RM, match_opcode, 0 },
    {"bf16.fp32.conv",   0, INSN_CLASS_ZFHMIN_INX, "D,S,m",   MATCH_BF16_FP32_CONV, MASK_BF16_FP32_CONV, match_opcode, 0 },
    {"fp32.bf16.conv",   0, INSN_CLASS_ZFHMIN_INX, "D,S",     MATCH_FP32_BF16_CONV, MASK_FP32_BF16_CONV|MASK_RM, match_opcode, 0 },
    {"bf16.add",     0, INSN_CLASS_ZFH_INX,   "D,S,T",     MATCH_BF16_ADD|MASK_RM, MASK_BF16_ADD|MASK_RM, match_opcode, 0 },
    {"bf16.add",     0, INSN_CLASS_ZFH_INX,   "D,S,T,m",   MATCH_BF16_ADD, MASK_BF16_ADD, match_opcode, 0 },
    {"bf16.sub",     0, INSN_CLASS_ZFH_INX,   "D,S,T",     MATCH_BF16_SUB|MASK_RM, MASK_BF16_SUB|MASK_RM, match_opcode, 0 },
    {"bf16.sub",     0, INSN_CLASS_ZFH_INX,   "D,S,T,m",   MATCH_BF16_SUB, MASK_BF16_SUB, match_opcode, 0 },
    
    {"bf16.fmadd",    0, INSN_CLASS_ZFH_INX,   "D,S,T,R",   MATCH_BF16_FMADD|MASK_RM, MASK_BF16_FMADD|MASK_RM, match_opcode, 0 },
    {"bf16.fmadd",    0, INSN_CLASS_ZFH_INX,   "D,S,T,R,m", MATCH_BF16_FMADD, MASK_BF16_FMADD, match_opcode, 0 },
    {"bf16.fmsub",    0, INSN_CLASS_ZFH_INX,   "D,S,T,R",   MATCH_BF16_FMSUB|MASK_RM, MASK_BF16_FMSUB|MASK_RM, match_opcode, 0 },
    {"bf16.fmsub",    0, INSN_CLASS_ZFH_INX,   "D,S,T,R,m", MATCH_BF16_FMSUB, MASK_BF16_FMSUB, match_opcode, 0 },
    {"bf16.fnmadd",   0, INSN_CLASS_ZFH_INX,   "D,S,T,R",   MATCH_BF16_FMNADD|MASK_RM, MASK_BF16_FMNADD|MASK_RM, match_opcode, 0 },
    {"bf16.fnmadd",   0, INSN_CLASS_ZFH_INX,   "D,S,T,R,m", MATCH_BF16_FMNADD, MASK_BF16_FMNADD, match_opcode, 0 },
    {"bf16.fnmsub",   0, INSN_CLASS_ZFH_INX,   "D,S,T,R",   MATCH_BF16_FMNSUB|MASK_RM, MASK_BF16_FMNSUB|MASK_RM, match_opcode, 0 },
    {"bf16.fnmsub",   0, INSN_CLASS_ZFH_INX,   "D,S,T,R,m", MATCH_BF16_FMNSUB, MASK_BF16_FMNSUB, match_opcode, 0 },
   ```
3. After updating both the files, build the toolchain again. Now the RISC-V toolchain has been modified to provide compilation support for newly added      BF16 instructions.

### Setting up verification environement
1. Clone the core-v-verif repository.
   ```sh
   git clone https://github.com/10x-Engineers/core-v-verif.git
   ```
   Checkout to the Bf16_Optimized branch.
2. Navigate to the (`core-v-cores`) folder and clone [this](https://github.com/10x-Engineers/cv32e40p/tree/BF16_Optimized) repository in it.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

### Running a simple program
If you have verilator and gtkwave installed, move to the steps below. If not then install verilator from [here](https://verilator.org/guide/latest/install.html). Install gtkwave from [here](https://sourceforge.net/projects/gtkwave/)

1. Tests for cv32e40p are located in (`cv32e40p/tests/programs/custom`). To run a simple test with BF16 custom instructions, we will run the scrap test present in (`cv32e40p/tests/programs/custom/scrap`). For that, use the command:
```sh
   make test TEST=scrap
```
This test runs all BF16 instructions one after the other.
2. To view the waveform, navigate to (`cv32e40p/tests/programs/custom/scrap`) and run the command:
```sh
gtkwave verilator_tb.vcd
```
For other make options, check out (`Available Test Programs`) in the (`Readme`) [here](https://github.com/10x-Engineers/core-v-verif/tree/master/mk)

<!-- USAGE EXAMPLES -->
## Usage

The custom instrcutions can be used as assembly or in inline assembly. The format used for the custom instructions is as follows:

* **Add**: bf16.add fd,fs1,fs2
* **Subtract**: bf16.sub fd,fs1,fs2
* **Minimum**: bf16.min fd,fs1,fs2
* **Maximum**: bf16.max fd,fs1,fs2
* **bf16 to fp32**: bf16.fp32.conv fd,fs1
* **fp32 to bf16**: fp32.bf16.conv fd,fs1
* **fmadd**: bf16.fmadd fd,fs1,fs2,fs3
* **fmsub**: bf16.fmsub fd,fs1,fs2,fs3
* **fnmadd**: bf16.fnmadd fd,fs1,fs2,fs3
* **fnmsub**: bf16.fnmsub fd,fs1,fs2,fs3

Example tests are given in the core-v-verif repository.

<p align="right">(<a href="#readme-top">back to top</a>)</p>


<!-- CONTRIBUTING -->
## Contributing

Contributions are what make the open source community such an amazing place to learn, inspire, and create. Any contributions you make are **greatly appreciated**.

If you have a suggestion that would make this better, please fork the repo and create a pull request. You can also simply open an issue with the tag "enhancement".
Don't forget to give the project a star! Thanks again!

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3. Commit your Changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the Branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

<p align="right">(<a href="#readme-top">back to top</a>)</p>





<!-- CONTACT -->
## Contact

Ruhma Rizwan - ruhma.rizwan@10xEngineers.ai

Zeshan Rehman - zeshan.rehman@10xEngineers.ai


<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- References -->
## References


* [10xEngineers website](https://10xengineers.ai/)


<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- MARKDOWN LINKS & IMAGES -->
<!-- https://www.markdownguide.org/basic-syntax/#reference-style-links -->
[contributors-shield]: https://img.shields.io/github/contributors/othneildrew/Best-README-Template.svg?style=for-the-badge
[contributors-url]: https://github.com/othneildrew/Best-README-Template/graphs/contributors
[forks-shield]: https://img.shields.io/github/forks/othneildrew/Best-README-Template.svg?style=for-the-badge
[forks-url]: https://github.com/othneildrew/Best-README-Template/network/members
[stars-shield]: https://img.shields.io/github/stars/othneildrew/Best-README-Template.svg?style=for-the-badge
[stars-url]: https://github.com/othneildrew/Best-README-Template/stargazers
[issues-shield]: https://img.shields.io/github/issues/othneildrew/Best-README-Template.svg?style=for-the-badge
[issues-url]: https://github.com/othneildrew/Best-README-Template/issues
[license-shield]: https://img.shields.io/github/license/othneildrew/Best-README-Template.svg?style=for-the-badge
[license-url]: https://github.com/othneildrew/Best-README-Template/blob/master/LICENSE.txt
[linkedin-shield]: https://img.shields.io/badge/-LinkedIn-black.svg?style=for-the-badge&logo=linkedin&colorB=555
[linkedin-url]: https://linkedin.com/in/othneildrew
[product-screenshot]: images/screenshot.png
[Next.js]: https://img.shields.io/badge/next.js-000000?style=for-the-badge&logo=nextdotjs&logoColor=white
[Next-url]: https://nextjs.org/
[React.js]: https://img.shields.io/badge/React-20232A?style=for-the-badge&logo=react&logoColor=61DAFB
[React-url]: https://reactjs.org/
[Vue.js]: https://img.shields.io/badge/Vue.js-35495E?style=for-the-badge&logo=vuedotjs&logoColor=4FC08D
[Vue-url]: https://vuejs.org/
[Angular.io]: https://img.shields.io/badge/Angular-DD0031?style=for-the-badge&logo=angular&logoColor=white
[Angular-url]: https://angular.io/
[Svelte.dev]: https://img.shields.io/badge/Svelte-4A4A55?style=for-the-badge&logo=svelte&logoColor=FF3E00
[Svelte-url]: https://svelte.dev/
[Laravel.com]: https://img.shields.io/badge/Laravel-FF2D20?style=for-the-badge&logo=laravel&logoColor=white
[Laravel-url]: https://laravel.com
[Bootstrap.com]: https://img.shields.io/badge/Bootstrap-563D7C?style=for-the-badge&logo=bootstrap&logoColor=white
[Bootstrap-url]: https://getbootstrap.com
[JQuery.com]: https://img.shields.io/badge/jQuery-0769AD?style=for-the-badge&logo=jquery&logoColor=white
[JQuery-url]: https://jquery.com 


# OpenHW Group CORE-V CV32E40P RISC-V IP

CV32E40P is a small and efficient, 32-bit, in-order RISC-V core with a 4-stage pipeline that implements
the RV32IM\[F|Zfinx\]C instruction set architecture, and the PULP custom extensions for achieving
higher code density, performance, and energy efficiency \[[1](https://doi.org/10.1109/TVLSI.2017.2654506)\], \[[2](https://doi.org/10.1109/PATMOS.2017.8106976)\].
It started its life as a fork of the OR10N CPU core that is based on the OpenRISC ISA.
Then, under the name of RI5CY, it became a RISC-V core (2016), and it has been maintained
by the [PULP platform](https://www.pulp-platform.org/) team until February 2020,
when it has been contributed to [OpenHW Group](https://www.openhwgroup.org/).

<p align="center"><img src="docs/images/CV32E40P_Block_Diagram.svg" width="750"></p>

## Documentation

The CV32E40P user manual can be found in the _docs_ folder and it is
captured in reStructuredText, rendered to html using [Sphinx](https://docs.readthedocs.io/en/stable/intro/getting-started-with-sphinx.html).
These documents are viewable using readthedocs and can be viewed [here](https://docs.openhwgroup.org/projects/cv32e40p-user-manual/).

## Verification
The verification environment for the CV32E40P is _not_ in this Repository.  There is a small, simple testbench here which is
useful for experimentation only and should not be used to validate any changes to the RTL prior to pushing to the master
branch of this repo.

The verification environment for this core as well as other cores in the OpenHW Group CORE-V family is at the
[core-v-verif](https://github.com/openhwgroup/core-v-verif) repository on GitHub.

The Makefiles supported in the **core-v-verif** project automatically clone the appropriate version of the **cv32e40p**  RTL sources.

## Changelog

A changelog is generated automatically in the documentation from the individual pull requests.
In order to enable automatic changelog generation within the CV32E40P documentation, the committer is required to label each pull request
that touches any file in 'rtl' (or any of its subdirectories) with *Component:RTL* and label each pull request that touches any file in
'docs' (or any of its subdirectories) with *Component:Doc*. Pull requests that are not labeled or labeled with *ignore-for-release* are
ignored for the changelog generation.

Only the person who actually performs the merge can add these labels (you need committer rights). The changelog flow only works if at most
1 label is applied and therefore pull requests that touches both RTL and documentation files in the same pull request are not allowed.

## Constraints
Example synthesis constraints for the CV32E40P are provided.

## Contributing

We highly appreciate community contributions. We are currently using the lowRISC contribution guide.
To ease our work of reviewing your contributions,
please:

* Create your own fork to commit your changes and then open a Pull Request to the **dev** branch.
* Split large contributions into smaller commits addressing individual changes or bug fixes. Do not
  mix unrelated changes into the same commit!
* Do not mix updates within the 'rtl' directory with updates within the 'docs' directory ino the same pull request.
* Write meaningful commit messages. For more information, please check out the [the Ibex contribution
  guide](https://github.com/lowrisc/ibex/blob/master/CONTRIBUTING.md).
* If asked to modify your changes, do fixup your commits and rebase your branch to maintain a
  clean history.
* If the PR gets accepted and merged into the the **dev** branch, an action is triggered automatically to check whether the changes are logically equivalent to the frozen RTL on a given set of parameters. If the changes are logically equivalent, the **dev** branch is automatically merged into the **master** branch. Otherwise, we need to investigate manually. If a bug is found, thus the changes are not logically equivalent, we follow the procedure documented [here](https://docs.openhwgroup.org/projects/cv32e40p-user-manual/core_versions.html). 

For more details on how this is implemented, have a look at this [page](https://github.com/openhwgroup/cv32e40p/blob/master/.github/workflows/aws_cv32e40p.md).

When contributing SystemVerilog source code, please try to be consistent and adhere to [the lowRISC Verilog
coding style guide](https://github.com/lowRISC/style-guides/blob/master/VerilogCodingStyle.md).

To get started, please check out the ["Good First Issue"
 list](https://github.com/openhwgroup/cv32e40p/issues?q=is%3Aissue+is%3Aopen+-label%3Astatus%3Aresolved+label%3A%22good+first+issue%22).

The RTL code has been formatted with ["Verible"](https://github.com/google/verible) v0.0-1149-g7eae750.
Run `./util/format-verible` to format all the files.

## Issues and Troubleshooting

If you find any problems or issues with CV32E40P or the documentation, please check out the [issue
 tracker](https://github.com/openhwgroup/cv32e40p/issues) and create a new issue if your problem is
not yet tracked.

## References

1. [Gautschi, Michael, et al. "Near-Threshold RISC-V Core With DSP Extensions for Scalable IoT Endpoint Devices."
 in IEEE Transactions on Very Large Scale Integration (VLSI) Systems, vol. 25, no. 10, pp. 2700-2713, Oct. 2017](https://doi.org/10.1109/TVLSI.2017.2654506)

2. [Schiavone, Pasquale Davide, et al. "Slow and steady wins the race? A comparison of
 ultra-low-power RISC-V cores for Internet-of-Things applications."
 _27th International Symposium on Power and Timing Modeling, Optimization and Simulation
 (PATMOS 2017)_](https://doi.org/10.1109/PATMOS.2017.8106976)


