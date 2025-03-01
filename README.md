# CRISPOR - a CRISPR/Cas9 assistant 

CRISPOR predicts off-targets in the genome, ranks guides, highlights
problematic guides, designs primers and helps with cloning.  Try it on
http://crispor.org

CRISPOR uses BWA, a few tools from the UCSC Genome Browser (twoBitToFa, bedClip),
various R packages and a huge collection of external packages and source code files
from published articles, see the file crisporEffScores.py for the exact references or 
the tool tips when you mouse over the scores on the interactive website or the user's
manual http://crispor.org/manual/.

If you need to analyze hundreds of thousands of guides for a library, the tool FlashFry is
probably the better tool for you, see https://github.com/aaronmck/FlashFry. That being said, CRISPOR 
now has .bed input, so as long as you are not running on thousands of exons, it is probably
fast enough for most applications.

If you only need efficiency scores and no interactive website, try "python
crisporEffScores.py", it is a python module but also has a command line
interface that may be sufficient for programmers. 

# Installation of a CRISPOR website mirror or as a command line tool

In this section, I assume that you are root and you want to setup a local CRISPOR website. If you only want
to use the command line tools, the installation commands below would be the
same, but 1) you don't need the sudo commands for pip and 2) you can use the 
option '--user' when running pip to install the tools into your own home directory
~/.local instead of /usr/ and /var/.

If you are unsure what the things below mean or if you just want to try it and
not install it or modify your server setup, you may want to try the virtual
machine, which is a complete installation of CRISPOR with everything included:
http://crispor.org/downloads/

CRISPOR uses python2.7. Change `pip` to `pip2` in the commands below if your default python is python3.
I do not have plans to change it to python3 right now as it is still pretty easy on most systems 
to get a python2 installed. Let me know if this is not the case for your Unix version.

Install BWA and a few required python modules:
    
    # Debian/Ubuntu
    apt-get install bwa python-pip python-matplotlib
    sudo pip install biopython numpy==1.14.0 scikit-learn==0.16.1 pandas twobitreader
    
or 
   
    # Fedora/Centos/Redhat/Scientific Linux
    yum install bwa python-pip python-devel tkinter
    sudo pip install biopython numpy==1.14.0 scikit-learn==0.16.1 pandas matplotlib twobitreader
    
The version of scikit-learn is important. If you want to use a newer version, the Microsoft Azimuth
data files have to be upgraded. Contact me or the Microsoft people if you want to do this.

For the Cpf1 scoring model:

    sudo pip install keras tensorflow h5py

I'm using the versions 2.1.5, 1.7.0 and 2.7.1 for these, but I hope that the exact version for these is 
not important.

Install required R libraries for the WangSVM efficiency score:
   
    sudo Rscript -e 'install.packages(c("e1071"),  repos="http://cran.rstudio.com/")'
    sudo Rscript -e 'if (!require("BiocManager", quietly = TRUE)) install.packages("BiocManager"); BiocManager::install(); BiocManager::install("limma")'

**Anaconda installation:**

    conda env create -f crispor.yml

The R packages have not changed in many years. The version should really not matter at all. In principle,
you can remove the wang score from crispor.py in the global variable where the scores are defined 
and not worry about R anymore. I don't think that as of 2020 anyone is still using this score 
for designing their guides.

When you run crispor.py, it should then show the usage message:
```
Usage: crispor.py [options] org fastaInFile guideOutFile 

Command line interface for the Crispor tool.

    org          = genome identifier, like hg19 or ensHumSap
    fastaInFile  = Fasta file
    guideOutFile = tab-sep file, one row per guide

    Use "noGenome" if you only want efficiency scoring (a LOT faster). This option 
    will use BWA only to match the sequence to the genome, extend it and obtain
    efficiency scores.

    If many guides have to be scored in batch: Add GGG to them to make them valid
    guides, separate these sequences by at least one "N" character and supply as a single
    fasta sequence, a few dozen to ~100 per file.
    

Options:
  -h, --help            show this help message and exit
  -d, --debug           show debug messages, do not delete temp directory
  -t, --test            run internal tests
  -p PAM, --pam=PAM     PAM-motif to use, default NGG. TTTN triggers special
                        Cpf1 behavior: no scores anymore + the PAM is assumed
                        to be 5' of the guide. Common PAMs are:
                        NGG,TTTN,NGA,NGCG,NNAGAA,NGGNG,NNGRRT,NNNNGMTT,NNNNACA
  -o OFFTARGETFNAME, --offtargets=OFFTARGETFNAME
                        write offtarget info to this filename
  -m MAXOCC, --maxOcc=MAXOCC
                        MAXOCC parameter, guides with more matches are
                        excluded
  --mm=MISMATCHES       maximum number of mismatches, default 4
  --bowtie              new: use bowtie as the aligner. Do not use. Bowtie
                        misses many off-targets.
  --skipAlign           do not align the input sequence. The on-target will be
                        a random match with 0 mismatches.
  --noEffScores         do not calculate the efficiency scores
  --minAltPamScore=MINALTPAMSCORE
                        minimum MIT off-target score for alternative PAMs, default
                        1.0
  --worker              Run as worker process: watches job queue and runs jobs
  --clear               clear the worker job table and exit
  -g GENOMEDIR, --genomeDir=GENOMEDIR
                        directory with genomes, default ./genomes
```
    
# Testing the script

To test the program, first make sure that there is a directory "../genomes".
If it's not there, rename "genomes.sample" to "genomes":

    mv ../genomes.sample ../genomes

Then run this command:

    mkdir -p sampleFiles/mine/
    crispor.py sacCer3 sampleFiles/in/sample.sacCer3.fa sampleFiles/mine/sample.sacCer3.tsv -o sampleFiles/mine/sample.sacCer3.mine.offs.tsv

The files in sampleFiles/mine should be identical to the files in sampleFiles/out/

The file testInHg19.fa contains a sample for the hg19 genome, the output is in testOutHg19.tab 
and testOutHg19Offtargets.tab

    ../crispor.py hg19 testInHg19.fa testOutHg19.mine.tab -o testOutHg19Offtargets.mine.tab

To add more genomes than yeast, skip the next section. If you want to run your script now as a web service, continue reading with the next section. 

# Running the script as a CGI under Apache with the job queue

Make sure you can execute CGI scripts somewhere. Your Apache config (e.g. /etc/apache2/sites-enabled/000-default) should contain a section like this:

    <Directory "/var/www/html">
         AllowOverride All
         Options +ExecCGI (...)
         AddHandler cgi-script .cgi .pl .py

Also make sure you have the CGI module enabled:

    sudo a2enmod cgi
    sudo service apache2 restart

If using SElinux, especially on Fedora/CentOS/RedHat, please switch it off or set it to permissive mode.

Clone the repo into such a directory:

    cd /var/www/html/
    git clone https://github.com/maximilianh/crisporWebsite
    
Use the sample E. coli genome for a start:

    mv genomes.sample genomes

Create a temp directory with the right permissions:
        
    mkdir temp
    chmod a+rw temp

Make sure that Apache is allowed to execute the crispor.py script, it should have x and r permissions for all:

    ls -la crispor.py
    # if not ...
    chmod a+rx crispor.py

By default, the jobs database is a SQlite file, /tmp/crisporJobs.db. The Apache
user has to be able to write to it so let us create it now:

    ./crispor.py --clear
    Worker queue now empty

Now start a single worker job. It will watch the job queue and process jobs:

    ./startWorkers.sh 1

Check that your worker is indeed running:
  
    cat log/worker1.log
    ps aux | grep crispor

Now try to access the script from a webbrowser, http://localhost/crispor.py and click "Submit"

# Adding a genome

If you want to add to your own crispor.py installation a genome that is already on crispor.org, that's very easy. All genomes available on crispor.org (except a few pre-publication ones) are provided as pre-indexed and correctly formatted files for download at http://crispor.tefor.net/genomes/. To get one of these into the current directory, use a command like this (replace hg38 with your genome code):

    mkdir genomes
    cd genomes
    mkdir hg38
    cd hg38
    wget -r -l1 --no-parent -nd  --reject 'index*' --reject 'robots*' http://crispor.tefor.net/genomes/hg38/
    
If you need to add a new genomes, this is quite a bit more involved. Ideally you want gene models in the right format (GFF), a fastsa file and various tools to convert and index these. In most cases, it's much easier to email crispor@tefor.net and ask me to add the genome, then you can download it as above. If this is not what you want, you can add a genome yourself, there even is a script for it. Look into the "tools" directory https://github.com/maximilianh/crisporWebsite/tree/master/tools, try the script crisprAddGenome. You will need to download the UCSC tools `twoBitToFa` and `bedToBigBed` from http://hgdownload.cse.ucsc.edu/admin/exe/linux.x86_64/ and install the tool `gffread` by installing cufflinks on your machine (e.g. with `apt-get install cufflinks`). 

The subdirectory usrLocalBin contains other required tools for this script, you can copy them into /usr/local/bin of your machine, they are 64bit static linux binaries and should work on most current machines.

The script can auto-download genomes from Ensembl, UCSC or NCBI or allows you to add
your own custom genome in .fasta format and .gff.

E.g. to add the X. laevis genome:

    sudo crisprAddGenome fasta /tmp2/LAEVIS_7.1.repeatMasked.fa --desc 'xenBaseLaevis71|Xenopus laev.e V7.1' --gff geneModels.gff3

The four `|`-split values for the --desc option are: `internalDatabaseName`, `scientificName`, `commonOrDisplayName`, `VersionNameOfAssembly`

Make sure that `internalDatabaseName` does not include special characters, spaces etc. as it is used as a directory name.

# "I am running many thousands of guides and it is very slow"

The .bed input is always fastest, as it saves the initial BWASW step where crispor maps to the target genome.

If you are using the FASTA input, instead of feeding it a multi-fasta file (where crispor will map every piece to
the genome first), try to feed it a single sequence and separate every 23bp-target in it with NN.
This means that you will not get the efficiency scores but you can run these separately or in parallel with 
crisporEfficiencyScores.py. 

For a major speedup in processing time, try to put the genome onto the ramdisk:

    twoBitToFa genomes/hg19/hg19.2bit /dev/shm/hg19.fa

crispor.py will find the genome file and use bedtools to get the
flanking sequences. This is almost 10x faster than the twoBitToFa command (at
the cost of more RAM).

Alternatively, you may want to give flashfry by Aaron McKenna a try. It is
optimized for large libraries, it uses much more RAM and has fewer scores but
is sufficient for most large-library-design applications.

# Thanks!
* Jean-Paul Concordet for numerous ideas on the user interface
* Alberto Stolfi for finding the N-SNP-bug
* Mark Diekhans for patching twoBitToFa and making it 100 times faster
* See the file changes.html for the full list of acknowledgements for every feature

# Licenses

Included software:

* BWA is under GPL3
* libSVM: under copyright by Chih-Chung Chang and Chih-Jen Lin see http://www.csie.ntu.edu.tw/~cjlin/libsvm/COPYRIGHT
* svmlight: free for non-commercial use, see http://svmlight.joachims.org/
* SSC: no license specified
* primer3: GPL2.
* Fusi/Doench score: see LICENSE.txt, (c) by Microsoft Research

CRISPOR itself:

* the two files crispor.py and crisporEffScores.py are released under a special license, see LICENSE.txt in this directory
