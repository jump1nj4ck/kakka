# qrgrader 
This framework allows to generate multiple answers randomized exams and grade them.


## qrworkspace ##

All the process must be execute inside a so-called _workspace_ which is a set of directory with a specific structure. To create a workspace the command `qrworkspace`:

```
$ qrworkspace 
Workspace qrgrading-220825 has been created correctly.
``` 
It is possible to specify the date with the paremeter `-d date`:

```
$ qrworkspace -d 220102
Workspace qrgrading-220102 has been created correctly.
```

In both cases a directory structure will be created:
```
$ ls qrgrading-220102
drwxrwxr-x 7 danilo danilo 4096 ago 25 14:26 ./
drwxrwxr-x 3 danilo danilo 4096 ago 25 11:53 ../
drwxrwxr-x 4 danilo danilo 4096 ago 25 11:53 generated/
drwxrwxr-x 4 danilo danilo 4096 ago 25 11:53 results/
drwxrwxr-x 2 danilo danilo 4096 ago 25 11:53 scanned/
drwxrwxr-x 3 danilo danilo 4096 ago 25 14:26 source/
```

## qrgenerator ##

Once the workspace has been created the latex project must be copied inside the `source` directory for the `qrgenerator` script to generate the randomized exams. The main latex file must be called `main.tex` or must be specified using the `-f` flag.

The script takes several additional flags: 

```
  -p, --process         Process source folder (main.tex)
  -f FILENAME, --filename FILENAME
                        Specify .tex filename
  -n NUMBER, --number NUMBER
                        Number or exams to be generated (1)
  -j THREADS, --threads THREADS
                        Maximum number of threads to use (4)
  -b BEGIN, --begin BEGIN
                        First exam number
  -v, --verbose         Extra verbosity
  -k, --feedback        Generate feedback file
  -P PAGES, --pages PAGES
                        Acceptable number of pages of output PDF
```
The script must be run from the root of the workspace. An example of basic use would be:

```
$ qrgenerator -p -n 100 
```
where we are asking the script to generate 100 exams with filenames from `220102001.pdf` to `220102100.pdf`.

A more advanced use would be:
```
$ qrgenerator -p -n 100 -b 250 -p 10 -j 2
```
where we are instead asking it to generate 100 exams with filenames from `220102250.pdf` to `220102349.pdf` using only 2 threads and specifying the only 10-pages exams are acceptable.

The script will generate the correpondent PDF files and the `results/xls/generated.txt` file that collects the information about all the QR generated by the `qrgenerator` for each exam (position and content).

The `-k` flag is automatically set when the `-p` is used and forces the generation of the `results/xls/220102_feedback.csv` file that contains the relationship between the actual questions with the exam-specific questions. This flag can also be used alone to re-creathe the cited file if it was lost.

## qrscanner ##

Once the exams have been filled by the students, all the pages must be scanned (generally at 400 DPI and using the color/text option) and scanned using the `qrscanner` script. The PDF file containing all the scanned exams must be put inside the `scanned` directory of the workspace. After that the script must be run from the root of the workspace:

```
$ qrscanner -p
```

With the `-p` flag, the script will process the PDF file and will scan **all the PDF files** in the `scanned` directory and will generate all the reconstructed exams in PDF format in `results/publish` and the files `220102_raw.csv` and `220102_nia.csv` in `results/xls`. The first file contains a matrix that collects all the answers given by the student sorted like in the original LaTex file and the second a table that associates the student NIA with the exam id number.

Thus, we will have:
```
$ ls qrgrading-220102/results/xls
-rw-rw-r-- 1 danilo danilo 16297 jul 19 12:28 220102_feedback.csv
-rw-rw-r-- 1 danilo danilo   702 jul 19 12:28 220102_nia.csv
-rw-rw-r-- 1 danilo danilo  8580 jul 19 12:28 220102_raw.csv
```
The `qrscanner` script has several flags that can modify its behavior:
```
  -T THRESHOLD [THRESHOLD ...]            Thresholding values (default: {50, 55, 60, 65, 70, 75, 80}, 0: no thresholding)
  -d DPI, --dpi DPI                       Input file DPI (default: 400)
  -z, --zxing                             Use ZXing
  -b, --zbar                              Use Zbar  
```
Usually, the default values are good enough so, processing using `-p` should be sufficient. The `-b` option forces the use of the `zbar` detector (instead of the default `zxing`) which is incapable of recognizing `aztec` codes.

It is possible to specify the first and last page to be scanned (useful for debugging purposes):
```
  -B FIRST_PAGE, --first-page FIRST_PAGE  First page to be processed (default: 1)
  -E LAST_PAGE, --last-page LAST_PAGE     Last page to be processed (default: last)
```

The `-r` flag allows to mark the reconstructed PDF files in different ways. The default value forced by the `-p` option omits any marking. The other options are:

``` 
  -r 1: Marks all the recognized QR/Aztec
  -r 2: Marks all and only the answers given by the students (red for wrong answer, green for correct answer)
```

This flag can also be used alone, multiples times and with different values **after** the scanned file have been processed but the new generated files will be overwritten.

The `qrscanner` script is also in charge of managing the uploading of the PDF and data to the Google Workspace. The `-U` flags will upload the `result/publish/*.pdf` (i.e. all the reconstructed files) to a Google Drive directory specified by the `uid` (which is the alphanumeric string that identifies a Google Drive directory). 

Assuming a given Google Drive directory `exam` has the address 
```
https://drive.google.com/drive/u/0/folders/1oQXGJ3yStnJ4ocY8moo1X5qZvViYDF3-
```

the command:

```
$ qrscanner -U 1oQXGJ3yStnJ4ocY8moo1X5qZvViYDF3-
```
will upload the cited files to the `exam` directory. The `-U` flag forces the `-l` flag that creates the table `results/xls/220201_pdf.csv` which contains the relationship between the exam number and the Google Drive PDF address.

The `-W` flag allows specifying the Google Sheet Workbook where the data will be uploaded.

Assuming there is a `qrscanner`-compatible workbook online called SAU2021-22 the command:
```
$ qrscanner -W SAU2021-22
```
will:

1. Copies the `master_raw` table into `220102_raw` table
2. Fills the `220201_raw` table with the content of `220102_raw.csv`
3. Creates a table called `220102_feedback` from the file `results/xls/220102_feedback.csv`
4. Creates a table called `220102_nia` from the file `results/xls/220102_nia.csv`
5. Creates a table called `220102_pdf` from the file `results/xls/220102_pdf.csv`



## Cheatsheet ##