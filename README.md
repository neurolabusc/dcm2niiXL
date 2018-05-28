## About

dcm2niiXL is a script for running [dcm2niix](https://github.com/rordenlab/dcm2niix) in parallel. It is useful for extra large (XL) datasets. Relative to other converters, [dcm2niix is fast](https://www.nitrc.org/plugins/mwiki/index.php/dcm2nii:MainPage#Alternatives). However, the basic tool is optimized for converting moderate sized datasets on a local machine (where disk access is very fast). In contrast, dcm2niiXL spawns multiple dcm2niix converters and can offer better cluster performance in situations where disk access is slow.

To be clear, for most users the classic dcm2niix is the best choice. The dcm2niiXL wrapper is for the specific case of very large datasets that are stored in multiple subdirectories and systems with slower disk access. This script is for advanced users, and you should read the requirements and limitations section carefully.

## Usage

You use [dcm2niiXL just like dcm2niix](https://www.nitrc.org/plugins/mwiki/index.php/dcm2nii:MainPage#General_Usage).

```
dcm2niiXL -f %p_%s -i y -z y -o ~/tst/out ~/dcm_qa/In/
```

## Installation

dcm2niiXL is a simple shell script that calls specially compiled versions of dcm2niix. Just download the files and place them in your path.

## Requirements and Limitations

This script is for advanced users and specific situations. You should read this section carefully. In many situations dcm2niix will be faster, and it is always less complicated.

- dcm2niiXL comes with a special build of dcm2niix, which uses the [accelerated single-threaded gzip](https://github.com/cloudflare/zlib) which requires Sandy Bridge or later CPUs (released in 2011). In contrast, the regular build of dcm2niix uses pigz for parallel gzip compression. The disadvantage of pigz is that the data must be saved to disk prior to compression (which can be slow), and multiple threads are used which can hinder running multiple instances of dcm2niix simultaneously.
 - dcm2niiXL spawns a separate instance of dcm2niix for every subfolder. Therefore, if you provide the input folder "/dcm" which contains images in the folders "/dcm/s1", "/dcm/s2" and "/dcm/s3", a total of 3 instances will be generated. This is more efficient than the classic method of running one instance that searches all the folders at once if you have large datasets. However, dcm2niiXL will be slower for small datasets. It also assumes your storage system is separating DICOM images into different folders (e.g. one per participant). Finally, in ancient times GE scanners would put a maximum of 1000 slices into a single folder and therefore would create multiple folders for a single series if it included more than 1000 slices. If you have these archival datasets, you should use the traditional dcm2niix.
 - dcm2niiXL is a shell script. It runs on MacOS and Linux, but does not support Windows. It leverages multiple threads, so it benefits from a CPU with multiple cores.
 - You can use all the normal dcm2niix options with the exception of the "-d" (maximum directory search depth) parameter. You can edit the first couple lines of the dcm2niiXL script if you wish to change this. In essence, dcm2niiXL will spawn one copy of dcm2niix for every folder, and each instance is limited to a search depth of 0 (it only converts the local folder).
  - It is generally a good idea to specify an output folder ("-o ~/myOutputDir"), otherwise each instance of dcm2niix will save files in their local input folder.
  - Since each folder is processed independently and multiple folders are processed in parallel, you should ensure your input data does not have naming conflicts (e.g. images from two different people that are in two different folders will both be named 'T1.nii'). The classic single-threaded dcm2niix detects naming conflicts and will save files appropriately (e.g. "T1.nii', 'T1a.nii'), but it is theoretically possible (though unlikely) for this process to be disrupted if two instances are running simultaneously.

## Compiling dcm2niix for accelerated gzip creation

Pre-compiled versions of dcm2niix with the cloudflare library are provided. However, if you wish, you can build your own.

- [download, configure and make your cloudflare zlib library](https://github.com/cloudflare/zlib)
- [dcm2niix](https://github.com/rordenlab/dcm2niix). Read the [compile page](https://github.com/rordenlab/dcm2niix/blob/master/COMPILE.md) for notes on a custom compile. Specifically, you will want to include the `-DmyDisableMiniZ` directive and include the path to your cloudflare zlib. One trick is to run the default build outlined on the dcm2niix homepage first: this will install OpenJPEG for you, which can be tricky. The compile notes also describe how to generate a version without OpenJPEG (though you will not be able to convert DICOM images that use the rare JPEG2000 transfer syntaxes). Here are examples, though your paths will be different:

MacOS

```
g++ -O3 -I. main_console.cpp nii_dicom.cpp nifti1_io_core.cpp nii_ortho.cpp nii_dicom_batch.cpp jpg_0XC3.cpp ujpeg.cpp nii_foreign.cpp -dead_strip -o dcm2niixMacOS -DmyDisableMiniZ -I/usr/local/include/openjpeg-2.1 /usr/local/lib/libopenjp2.a -I/Users/rorden/Documents/cocoa/zlib /Users/rorden/Documents/cocoa/zlib/libz.a
```

Linux
```
g++ -O3 -I. main_console.cpp nii_dicom.cpp nifti1_io_core.cpp nii_ortho.cpp nii_dicom_batch.cpp jpg_0XC3.cpp ujpeg.cpp nii_foreign.cpp -o dcm2niix -DmyDisableMiniZ -I~/zlib ~/zlib/libz.a -I/home/research/dcm2niix/build/include/openjpeg-2.1 /home/research/dcm2niix/build/lib/libopenjp2.a
```
