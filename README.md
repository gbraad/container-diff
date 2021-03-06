# container-diff

[![Build
Status](https://travis-ci.org/GoogleCloudPlatform/container-diff.svg?branch=master)](https://travis-ci.org/GoogleCloudPlatform/container-diff)

## What is container-diff?

container-diff is an image analysis command line tool.  container-diff can analyze images along several different criteria, currently including:
- Docker Image History
- Image file system
- apt-get installed packages
- pip installed packages
- npm installed packages
The above analyses can be performed on a single image, or a diff can be performed on two images to compare images.

This tool can help you as a developer better understand what is changing within your images and better understand what your images contain.

## Installation

### macOS
```shell
curl -LO https://storage.googleapis.com/container-diff/v0.2.0/container-diff-amd64-darwin && chmod +x container-diff-amd64-darwin && sudo mv container-diff-amd64-darwin /usr/local/bin/container-diff
```

### Linux
```shell
curl -LO https://storage.googleapis.com/container-diff/v0.2.0/container-diff-amd64-linux && chmod +x container-diff-amd64-linux && sudo mv container-diff-amd64-linux /usr/local/bin/container-diff
```

OR, if you want to avoid using sudo:

```shell
curl -LO https://storage.googleapis.com/container-diff/v0.2.0/container-diff-linux-amd64 && chmod +x container-diff-linux-amd64 && mkdir $HOME/bin && export PATH=$PATH:$HOME/bin && mv container-diff-linux-amd64 $HOME/bin/container-diff
```

### Windows
Download the [container-diff-windows-amd64.exe](https://storage.googleapis.com/container-diff/v0.2.0/container-diff-amd64-windows.exe) file, rename it to `container-diff.exe` and add it to your path


## Quickstart

To use container-diff to perform analysis on a single image, you need one Docker image (in the form of an ID, tarball, or URL from a repo).  Once you have that image, you can run any of the following analyzers:

```
container-diff <img>     [Run all analyzers]
container-diff <img> -d  [History]
container-diff <img> -f  [File System]
container-diff <img> -p  [Pip]
container-diff <img> -a  [Apt]
container-diff <img> -n  [Node]
```

To use container-diff to perform a diff analysis on two images, you need two Docker images (in the form of an ID, tarball, or URL from a repo).  Once you have those images, you can run any of the following differs:
```
container-diff <img1> <img2>     [Run all differs]
container-diff <img1> <img2> -d  [History]
container-diff <img1> <img2> -f  [File System]
container-diff <img1> <img2> -p  [Pip]
container-diff <img1> <img2> -a  [Apt]
container-diff <img1> <img2> -n  [Node]
```

You can similarly run many differs or analyzers at once:

```
container-diff <img1> <img2> -d -a -n [History, Apt, and Node]
```

All of the analyzer flags with their long versions can be seen below:

| Differ                    | Short flag | Long Flag  |
| ------------------------- |:----------:| ----------:|
| File system diff          | -f         | --file     |
| History                   | -d 	 | --history  |
| npm installed packages    | -n 	 | --node     |
| pip installed packages    | -p 	 | --pip      |
| apt-get installed packages| -a 	 | --apt      |




## Other Flags

To get a JSON version of the container-diff output add a `-j` or `--json` flag.

```container-diff <img1> <img2> -j```

To use the docker client instead of shelling out to your local docker daemon, add a `-e` or `--eng` flag.

```container-diff <img1> <img2> -e```

To order files and packages by size (in descending order) when performing file system or package analyses/diffs, add a `-o` or `--order` flag.

```container-diff <img1> <img2> -o```


## Analysis Result Format

The JSONs for analysis results are in the following format:
```
{
    "Image": "foo",
    "AnalyzeType": "Apt",
    "Analysis": {},
}
```
The possible structures of the `Analysis` field are detailed below.

### History Analysis

The history analyzer outputs a list of strings representing descriptions of how an image layer was created.

### File System Analysis

The file system analyzer outputs a list of strings representing file system contents.

### Package Analysis

Package analyzers such as pip, apt, and node inspect the packages installed within the image provided.  All package analyses leverage the PackageOutput struct, which contains the version and size for a given package instance (and a potential installation path for a specific instance of a package where multiple versions are allowed to be installed), as detailed below:
```
type PackageOutput struct {
	Name	string
	Path	string
	Version string
	Size    int64
}
```

#### Single Version Package Analysis

Single version package analyzers (apt) have the following output structure: `[]PackageOutput`

Here, the `Path` field is omitted because there is only one instance of each package.


#### Multi Version Package Analysis

Multi version package analyzers (pip, node) have the following output structure: `[]PackageOutput`

Here, the `Path` field is included because there may be more than one instance of each package, and thus the path exists to pinpoint where the package exists in case additional investigation into the package instance is desired.


## Diff Result Format

The JSONs for diff results are in the following format:
```
{
    "Image1": "foo",
    "Image2": "bar",
    "DiffType": "Apt",
    "Diff": {},
}
```
The possible structures of the `Diff` field are detailed below.

### History Diff

The history differ has the following json output structure:

```
type HistDiff struct {
	Adds   []string
	Dels   []string
}
```

### File System Diff

The file system differ has the following json output structure: 

```
type DirDiff struct {
	Adds   []string
	Dels   []string
	Mods   []string
}
```

### Package Diffs

Package differs such as pip, apt, and node inspect the packages contained within the images provided.  All packages differs currently leverage the PackageInfo struct which contains the version and size for a given package instance, as detailed below:
```
type PackageInfo struct {
	Version string
	Size    string
}
```

#### Single Version Package Diffs

Single version differs (apt) have the following json output structure:

```
type PackageDiff struct {
	Packages1 []PackageOutput
	Packages2 []PackageOutput
	InfoDiff  []Info
}
```

Packages1 and Packages2 detail which packages exist uniquely in Image1 and Image2, respectively, with package name, version and size info.  InfoDiff contains a list of Info structs, each of which contains the package name (which occurred in both images but had a difference in size or version), and the PackageInfo struct for each package instance. 

#### Multi Version Package Diffs

The multi version differs (pip, node) support processing images which may have multiple versions of the same package.  Below is the json output structure:

```
type MultiVersionPackageDiff struct {
	Packages1 []PackageOutput
	Packages2 []PackageOutput
	InfoDiff  []MultiVersionInfo
}
```

Packages1 and Packages2 detail which packages exist uniquely in Image1 and Image2, respectively, with package name, installation path, version and size info.  InfoDiff here is exanded to allow for multiple versions to be associated with a single package.  In this case, a package of the same name is considered to differ between two images when there exist one or more instances of it installed in one image but not the other (i.e. have a unique version and/or size).

```
type MultiVersionInfo struct {
	Package string
	Info1   []PackageInfo
	Info2   []PackageInfo
}
```

## Known issues

To run container-diff on image IDs, docker must be installed.


## Example Run

```
$ container-diff gcr.io/google-appengine/python:2017-07-21-123058 gcr.io/google-appengine/python:2017-06-29-190410 -a -n -p

-----AptDiffer-----

Packages found only in gcr.io/google-appengine/python:2017-07-21-123058: None

Packages found only in gcr.io/google-appengine/python:2017-06-29-190410: None

Version differences:
PACKAGE             IMAGE1 (gcr.io/google-appengine/python:2017-07-21-123058)        IMAGE2 (gcr.io/google-appengine/python:2017-06-29-190410)
-libgcrypt20        1.6.3-2 deb8u4, 998K                                             1.6.3-2 deb8u3, 1002K

-----NodeDiffer-----

Packages found only in gcr.io/google-appengine/python:2017-07-21-123058: None

Packages found only in gcr.io/google-appengine/python:2017-06-29-190410: None

Version differences: None

-----PipDiffer-----

Packages found only in gcr.io/google-appengine/python:2017-07-21-123058: None

Packages found only in gcr.io/google-appengine/python:2017-06-29-190410: None

Version differences: None

```
## Example Run with json post-processing
The following example demonstrates how one might selectively display the output of their diff, such that version differences are ignored and only package absence/presence is displayed and the packages present in only one image are sorted by size in descending order.  A small piece of the json being post-processed can be seen below:
```
[
  {
    "DiffType": "AptDiffer",
    "Diff": {
      "Image1": "gcr.io/gcp-runtimes/multi-base",
      "Packages1": {},
      "Image2": "gcr.io/gcp-runtimes/multi-modified",
      "Packages2": {
        "dh-python": {
          "Version": "1.20141111-2",
          "Size": "277"
        },
        "libmpdec2": {
          "Version": "2.4.1-1",
          "Size": "275"
        },
        ...
```
The post-processing script used for this example is below:

```import sys, json

def main():
  data = json.loads(sys.stdin.read())
  img1packages = []
  img2packages = []
  for differ in data:
    diff = differ['Diff']

    if len(diff['Packages1']) > 0:
      for package in diff['Packages1']:
        Size = package['Size']
        img1packages.append((str(package), int(str(Size))))

    if len(diff['Packages2']) > 0:
      for package in diff['Packages2']:
        Size = package['Size']
        img2packages.append((str(package), int(str(Size))))
    
    img1packages = reversed(sorted(img1packages, key=lambda x: x[1]))
    img2packages = reversed(sorted(img2packages, key=lambda x: x[1]))


    print "Only in image1\n"
    for pkg in img1packages:
      print pkg
    print "Only in image2\n"
    for pkg in img2packages:
      print pkg
    print

if __name__ == "__main__":
  main()
```

Given the above python script to postprocess json output, you can produce the following behavior:
```
container-diff gcr.io/gcp-runtimes/multi-base gcr.io/gcp-runtimes/multi-modified -a -j | python pyscript.py

Only in image1

Only in image2

('libpython3.4-stdlib', 9484)
('python3.4-minimal', 4506)
('libpython3.4-minimal', 3310)
('python3.4', 336)
('dh-python', 277)
('libmpdec2', 275)
('python3-minimal', 96)
('python3', 36)
('libpython3-stdlib', 28)

```
## Make your own differ

Feel free to develop your own analyzer leveraging the utils currently available.  PRs are welcome.

### Custom Analyzer Quickstart

In order to quickly make your own analyzer, follow these steps:

1. Add your analyzer identifier to the flags in [root.go](https://github.com/GoogleCloudPlatform/container-diff/blob/ReadMe/cmd/root.go)
2. Determine if you can use existing analyzing or diffing tools.  If you can make use of existing tools, you then need to construct the structs to feed into the tools by getting all of the packages for each image or the analogous quality to be analyzed.  To determine if you can leverage existing tools, think through these questions:
- Are you trying to analyze packages?
    - Yes: Does the relevant package manager support different versions of the same package on one image?
        - Yes: Implement `getPackages` to collect all versions of all packages within an image in a `map[string]map[string]PackageInfo`. Use `GetMultiVerisonMapDiff` to diff map objects.  See [nodeDiff.go](https://github.com/GoogleCloudPlatform/container-diff/blob/master/differs/nodeDiff.go#L33)  or [pipDiff.go](https://github.com/GoogleCloudPlatform/container-diff/blob/master/differs/pipDiff.go#L23) for examples.
        -  No: Implement `getPackages` to collect all versions of all packages within an image in a `map[string]PackageInfo`. Use `GetMapDiff` to diff map objects.  See [aptDiff.go](https://github.com/GoogleCloudPlatform/container-diff/blob/master/differs/aptDiff.go#L29). 
    - No: Look to [History](https://github.com/GoogleCloudPlatform/container-diff/blob/ReadMe/differs/historyDiff.go) and [File System](https://github.com/GoogleCloudPlatform/container-diff/blob/ReadMe/differs/fileDiff.go) differs as models for diffing.

3. Write your analyzer driver in the `differs` directory, such that you have a struct for your analyzer type and two methods for that analyzer: `Analyze` for single image analysis and `Diff` for comparison between two images:

```
type YourAnalyzer struct {}

func (a YourAnalyzer) Analyze(image utils.Image) (utils.Result, error) {...}
func (a YourAnalyzer) Diff(image1, image2 utils.Image) (utils.Result, error) {...}
```
The image arguments passed to your analyzer contain the path to the unpacked tar representation of the image, as well as certain configuration information (e.g. environment variables upon image creation and image history).

If using existing package tools, you should create the appropriate structs (e.g. `SingleVersionPackageAnalyzeResult` or `SingleVersionPackageDiffResult`) to analyze or diff.  Otherwise, create your own structs which should yield information to fill an AnalyzeResult or DiffResult as the return type for Analyze() and Diff(), respectively, and should implement the `Result` interface, as in the next step.

4. Create a struct following the `Result` interface by implementing the following two methods.
```
      GetStruct() interface{}
      OutputText(diffType string) error
```

This is where you define how your analyzer should output for a human readable format (`OutputText`) and as a struct which can then be written to a `.json` file.  See [diff_output_utils.go](https://github.com/GoogleCloudPlatform/container-diff/blob/master/utils/diff_output_utils.go) and [analyze_output_utils.go](https://github.com/GoogleCloudPlatform/container-diff/blob/master/analyze_output_utils.go).

5. Add your analyzer to the `analyses` map in [differs.go](https://github.com/GoogleCloudPlatform/container-diff/blob/master/differs/differs.go#L22) with the corresponding Analyzer struct as the value.





