# UToronto OPAE Guide
This guide will be using OPAE v1.1.0 (current latest version is 1.3.0)

Make new directory for intalling libraries (will be named "intel" in this guide):  
`$ mkdir intel`  
`$ cd intel`

## 1 - Install OPAE

### Guide: https://opae.github.io/latest/docs/install_guide/installation_guide.html  

### To download v1.1 release tar:  
`$ wget "https://github.com/OPAE/opae-sdk/releases/download/1.1.0-2/opae-1.1.0-2.tar.gz"`

### Untar & build:  
Install in local directory, replace `<new prefix>` in cmake command to install location (will be "~/intel/OPAE_1_1" in this guide)
```
$ tar zxvf opae-sdk-<release>.tar.gz
$ cd opae-sdk-<release>
$ cd usr
$ mkdir mybuild
$ cd mybuild
$ cmake .. -DBUILD_ASE=1 -DCMAKE_INSTALL_PREFIX=<new prefix>
$ make 
$ make install
```

## 2 - Install FPGA-BBB

Return to instalation directory: `$ cd ~/intel`

### Guide: https://github.com/OPAE/intel-fpga-bbb/wiki/Installation

### Clone branch matching OPAE release installed above (in this guie: 1.1.0):  
`$ git clone -b release/1.1.0 https://github.com/OPAE/intel-fpga-bbb`

### Build:


**!! IMPORTANT !!**  
Before calling `cmake ...` edit "intel-fpga-bbb/CMakeLists.txt" adding:  
`SET(CMAKE_CXX_FLAGS "-std=c++11")`  
right below the line `project("BBB")`

Make sure the cmake prefix is the same as used in step 1 (in this guide: "~/intel/OPAE_1_1")  
```
$ cd intel-fpga-bbb
$ mkdir mybuild
$ cd mybuild
$ cmake .. -DCMAKE_INSTALL_PREFIX=<step 1 prefix>
$ make
$ make install
```

## 3 - Environment
I suggest creating a file fore OPAE-specific environment setup; name it "~/.opae_env" or "~/.bash_opae". The following steps will assume a bash terminal.

`touch ~/.bash_opae`

in ".bashrc" add:
```
if [ -f ~/.bash_opae ]; then
    . ~/.bash_opae
fi
```

in ".bash_opae" add the following (use absolute paths, **do not use '~'**):

```
#Modify these lines:

    #Replace with correct username
        export USR_HOME_ABS="/home/<username>"
    
    #Replace with the prefix you used to install OPAE & BBB
        export OPAE_INSTALL="${USR_HOME_ABS}/intel/OPAE_1_1"
    
    #if on snail use this CAD_TOOL_LOC otherwise inquire about location
        export CAD_TOOL_LOC="/home/tsa"
    
    #Replace the following folders with the location of the source files
        export FPGA_BBB_CCI_SRC="${USR_HOME_ABS}/intel/intel-fpga-bbb"
        export OPAE_BASEDIR="${USR_HOME_ABS}/intel/opae-1.1.0-2"

#Do not modify below

PATH=$PATH:${OPAE_INSTALL}/bin:${OPAE_INSTALL}/share/opae/ase
PATH=$PATH:${CAD_TOOL_LOC}/intelFPGA_pro/16.0/quartus/bin:${CAD_TOOL_LOC}/modelsim105b/modeltech/linux_x86_64

export C_INCLUDE_PATH="${OPAE_INSTALL}/include/"
export CPLUS_INCLUDE_PATH="${OPAE_INSTALL}/include/"
export LD_LIBRARY_PATH="${OPAE_INSTALL}/lib"
export LIBRARY_PATH="${OPAE_INSTALL}/lib"

export ALTERAOCLSDKROOT="${CAD_TOOL_LOC}/intelFPGA_pro/16.0/hld"
export QSYS_ROOTDIR="${CAD_TOOL_LOC}/intelFPGA_pro/16.0/quartus/sopc_builder/bin"
export QUARTUS_HOME="${CAD_TOOL_LOC}/intelFPGA_pro/16.0/quartus/"
export QUARTUS_ROOTDIR="${CAD_TOOL_LOC}/intelFPGA_pro/16.0/quartus/"
export QUARTUS_64BIT="1"
export QUARTUS_ROOTDIR_OVERRIDE="${CAD_TOOL_LOC}/intelFPGA_pro/16.0/quartus/"
export MTI_HOME="${CAD_TOOL_LOC}/modelsim105b/modeltech"
alias quartus="quartus --64bit"

export LM_LICENSE_FILE="ENQUIRE ABOUT LISCENSE SERVER"
export MGLS_LICENSE_FILE="ENQUIRE ABOUT LISCENSE SERVER"
```

## 2.5 - Test Installation
Quick test to make sure everything works as of this step:
```
$ cd ~/intel/intel-fpga-bbb/samples/tutorial/01_hello_world
$ ./hw/sim/setup_ase build_sim
$ cd build_sim
$ make
$ make sim
```
Green text should appear; fin the line that looks like this:  
`#   [SIM]  bash/zsh | export ASE_WORKDIR=/home/<username>/intel/intel-fpga-bbb/samples/tutorial/01_hello_world/buil_sim/work`  
Copy the export command

Open a second terminal:
```
$ <paste text copied from first terminal here>
$ cd intel/intel-fpga-bbb/samples/tutorial/01_hello_world/sw/
$ make
$ cd build_sim
$ make
$ ./cci_hello_ase
```

Should print "Hello world!" and finally `[APP]  Session ended` without errors

## 3 - Migrating OPAE v0.9.0 Project to v1.1.0

Suggestion 1: use my seed project and replace the source files with your own.

Suggestion 2: use one of the sample applications, and only replace the source files; try to make, and solve issues as they arise. Below are some examples / suggestions on how to fix some of the issues.

### Software
**Buffer allocation**  
old:
```
uint64_t* p_workspace = (uint64_t*)fpgaHandle->allocBuffer(buffer_size);
```
New:
```
shared_ptr<opae::fpga::types::shared_buffer> buf_handle = fpgaHandle->allocBuffer(buffer_size);;
volatile uint64_t* p_workspace = reinterpret_cast<volatile uint64_t*>(buf_handle->c_type());
```

**Make Process**  
The make process requires a set of base include files. I provide a folder called "opae-common". I suggest adding this folder to your include directory and have your makefile point to the local project files. One reason for this strategy, is that some of the files have different version depending on which project they are included in. Also, including these files from other projects requires that the paths to these files be identical on every machine on which the code runs (hint: they wont be). Keeping them local to the project makes the entire make process portable and entirely reliant on the base OPAE install directiores; not some sample-specific files.

I include thses files using the following path: `<project_dir>/include/external/opae-common`  
Replace the first line of the Makefile with: `include ./include/external/opae-common/base_include.mk`  
\*** If you use my included makefile, this change is already done

This should solve most of the software-based make-related issues

**AFU Config JSON**  
AFU configurations are now stored in a JSON, this allows the accelerator UUID to be determined at make-time by reading from the JSON and storing the value in a macro.

There should be a file: "<project_dir>/<project_name>.json" which has the same structure as the json included in this repository.

## I - Additional Notes
- If you get this issue `/bin/sh: 1: Bad substitution` add `SHELL = bash` to the top of the Makefile, or change the symlink for bin/sh to point to bin/bash rather than bin/dash (ubuntu specific issue)