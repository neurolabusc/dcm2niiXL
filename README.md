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
 - dcm2niiXL spawns a separate instance of dcm2niix for every subfolder. Therefore, if you provide the input folder "/dcm" which contains images in the folders "/dcm/s1", "/dcm/s2" and "/dcm/s3", a total of 3 instances will be generated. This is more efficient than the classic method of running one instance that searches all the folders at once if you have large datasets. However, dcm2niiXL will be slower for small datasets. It also assumes your storage system is separating DICOM images into different folders (e.g. one per participant).
 - dcm2niiXL assumes that ALL the files for an image are contained in a single folder. If this is not the case, you should use the standard dcm2niix to combine images across folders. For example, Philips classic data is typically stored with a maximum of 2048 2D slices per folder. Any session where more than 2048 slices are acquired may be stored in multiple folders. For this reason, Philips users may want to export their data as enhanced DICOM format.
 - dcm2niiXL is a shell script. It runs on MacOS and Linux, but does not support Windows. It leverages multiple threads, so it benefits from a CPU with multiple cores.
 - You can use all the normal dcm2niix options with the exception of the "-d" (maximum directory search depth) parameter. You can edit the first couple lines of the dcm2niiXL script if you wish to change this. In essence, dcm2niiXL will spawn one copy of dcm2niix for every folder, and each instance is limited to a search depth of 0 (it only converts the local folder).
 - It is generally a good idea to specify an output folder ("-o ~/myOutputDir"), otherwise each instance of dcm2niix will save files in their local input folder.
 - Since each folder is processed independently and multiple folders are processed in parallel, you should ensure your input data does not have naming conflicts (e.g. images from two different people that are in two different folders will both be named 'T1.nii'). The classic single-threaded dcm2niix detects naming conflicts and will save files appropriately (e.g. "T1.nii', 'T1a.nii'), but it is theoretically possible (though unlikely) for this process to be disrupted if two instances are running simultaneously.


## A Simpler Alternative: Parallel Compression

The bulk of this page describes running multiple single-threaded instances of dcm2niix on a dataset. This requires the user to make sure that the data is carefully sorted so that each thread processes a whole dataset. An easier approach is to ensure you have fast parallel compression. The rationale is simple: if you create compressed .nii.gz images the software will spend most of its time compressing your data rather than converting your data. Therefore, a good strategy is to use the parallel pigz to accelerate the compression stage. The data below illustrates this on a six core (12 thread) computer where the single-threaded cloud-flare library is ten times slower than saving raw data to disk, while pigz is just 3.2 times slower. In this test, we converted about an hours worth of MRI data. A slow spinning hard-disk was used (as is typical for servers). The raw data required 2.234 seconds to save, but in the table we have scaled everything relative to this time (so this uncompressed time is listed as 1).

One consideration for optimal performance. Versions of dcm2niix from v1.0.20181225 include a new optimal compression option (`-z o`) for Unix computers. This requires you to have a version of pigz installed that is 2.3.4 or later. This option pipes data directly to pigz. In contrast, the conventional usage of pigz (`-z y`) saves the raw data to disk and has pigz read this data from disk to save a compressed version. This two-stage approach carries a huge penalty for slow disks.

| Method                                         | Speed |
| ---------------------------------------------- | ----- |
| `-z n` (`n`o compression: raw .nii files)      |  1.0  |
| `-z o` (`o`ptimal piped pigz)                  |  3.2  |
| `-z y` (`y`es pigz)                            |  4.4  |
| `-z i` (`i`nternal, compiled to cloudflare)    | 10.1  |
| `-z i` (`i`nternal, compiled to zlib)          | 20.6  |


## Compiling dcm2niix for accelerated gzip creation

The following command should create a `dcm2niix` executable in the dcm2niix\build\bin folder that uses the accelerated cloudflare library.

```
git clone https://github.com/rordenlab/dcm2niix.git
cd dcm2niix
mkdir build && cd build
cmake -DZLIB_IMPLEMENTATION=Cloudflare -DUSE_JPEGLS=ON -DUSE_OPENJPEG=ON ..
make
```

