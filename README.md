# Label Generator

Generates labeled training data fro text detector. The input files (json and png) are generated by [pdffigures](http://pdffigures.allenai.org/).


## Usage

* Set up your access and private key for S3
* Compile pdffigures
* Get a list of all papers
  * `aws s3 --region=us-west-2 ls s3://escience.washington.edu.viziometrics/acl_anthology/pdf/ | awk '{ print $4 }' > paper_list.txt`
    (for testing: `aws s3 ls escience.washington.edu.viziometrics/test/pdf/ | awk '{ print $4 }' > paper_list.txt`)
  * or locally `ls ~/Downloads/papers | cat > paper_list.txt`
* If you use S3, create a ramdisk to speed up file operations.
```
mkdir -p /tmp/ram
sudo mount -t tmpfs -o size=2G tmpfs /tmp/ram/
```
* Run in parallel `cat paper_list.txt | parallel --no-run-if-empty --bar -j 2% --joblog /tmp/par.log python main.py read-s3 escience.washington.edu.viziometrics test/pdf/{} test`

### Resume parallel jobs

Add `--resume` or `--resume-failed` to the command.

### Kill if one fails

`--halt 1`

### Monitor progress

`tail -f /tmp/par.log`


## Requirements

Install OpenCV with python support. Also install freetype, ghostscript, and imagemagic.

## AWS instructions

```
sudo apt-get update
sudo apt-get install git python-pip python-opencv ghostscript libmagickwand-dev libfreetype6 git parallel

git clone https://github.com/domoritz/label_generator.git
cd label_generator
sudo pip install -r requirements.txt
git submodule init
git submodule update

sudo apt-get install libpoppler-dev libleptonica-dev pkg-config

# we need gcc 4.9
sudo add-apt-repository ppa:ubuntu-toolchain-r/test
sudo apt-get update
sudo apt-get install g++-4.9

make -C pdffigures DEBUG=0 CC='g++-4.9 -std=c++11'

# Test with one file
python main.py read-s3 escience.washington.edu.viziometrics acl_anthology/pdf/C08-1099.pdf acl_anthology

# use tmux (maybe with attach)
tmux

# get list of documents to process
aws s3 --region=us-west-2 ls s3://escience.washington.edu.viziometrics/acl_anthology/pdf/ | awk '{ print $4 }' > acl_papers.txt

# now run for real
parallel --resume -j +6 --no-run-if-empty --eta --joblog /tmp/par.log python main.py read-s3 escience.washington.edu.viziometrics acl_anthology/pdf/{} escience.washington.edu.viziometrics acl_anthology --dbg-image :::: acl_papers.txt

# find bad labels
python find_bad.py read-s3 escience.washington.edu.viziometrics acl_anthology/json > anthology_bad.txt
# you probably want to use this file to delete bad labels before you use it to train the CNN
# Use: parallel rm -f data/{}-label.png :::: anthology_bad.txt
```

## FAQ

**I don't see my output** Try `--debug` and make sure that you have the correct folders set up if you use S3.

**Failed to initialize libdc1394** `sudo ln /dev/null /dev/raw1394` https://stackoverflow.com/questions/12689304/ctypes-error-libdc1394-error-failed-to-initialize-libdc1394

## Try

### Local

`python main.py read testdata/paper.pdf /tmp/test --dbg-image`

### S3

`python main.py read-s3 escience.washington.edu.viziometrics test/pdf/C08-1092.pdf test/ --debug`
