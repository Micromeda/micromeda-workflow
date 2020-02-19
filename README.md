Micromeda Platform
==================

Overview
--------

The [Micromeda platform](https://github.com/Micromeda) allows users to predict which [genome properties](https://github.com/ebi-pf-team/genome-properties) are possessed by organisms and then compare the presence and absence of these properties across organisms. The platform has three core components:

- **Micromeda-Visualizer** -- A web-based visualization tool that draws interactive heat maps of genome property and property step assignments. It has two components [Micromeda-[CLI](https://en.wikipedia.org/wiki/Command-line_interface)ent](https://github.com/Micromeda/micromeda-[CLI](https://en.wikipedia.org/wiki/Command-line_interface)ent) and [Micromeda-Server](https://github.com/Micromeda/micromeda-server) and is available at [http://mellea.uwaterloo.ca:5001/index.html](http://mellea.uwaterloo.ca:5001/index.html) (**uWaterloo VPN required**).
- **Pygenprop** -- A [Jupyter Notebooks](https://jupyter.org) compatible Python library that allows for the programmatic comparison of property and step assignments across organisms. Pygenprop also allows users to explore the [InterProScan](https://github.com/ebi-pf-team/interproscan) annotations and protein sequences that support genome property assignments. Pygenprop also has a [command-line interface ([CLI](https://en.wikipedia.org/wiki/Command-line_interface))](https://en.wikipedia.org/wiki/Command-line_interface) for generating **Micromeda files**.
- **Micromeda Files** -- These files allow for the aggregation and transfer of genome property assignments, step assignments, and supporting [InterProScan](https://github.com/ebi-pf-team/interproscan) annotations and protein sequences from multiple organisms. They allow for the transfer of complete property analysis datasets between researchers. **They are also used to transfer datasets to Micromeda-Visualizer.**

Analysis Workflow
-----------------

Analyzing datasets using Micromeda involves the following steps:

- Acquiring an organism's protein sequences
- Annotating an organism's proteins using [InterProScan5](https://github.com/ebi-pf-team/interproscan)
- Creating a Micromeda file using **Pygenprop's** [CLI](https://en.wikipedia.org/wiki/Command-line_interface)
- Uploading the Micromeda file to **Micromeda-Visualizer**

#### An overview of creating Micromeda files.
![Overview of creating Micromeda files](media/how-micromeda-files-are-built.png)

Installing Software and Databases
---------------------------------

The following pieces of software must be installed to generate Micromeda files:

- [InterProScan5](https://github.com/ebi-pf-team/interproscan)
- [Pygenprop](https://github.com/Micromeda/pygenprop)

The following pieces of software are used in the tutorial but are optional:

- [Prodigal](https://github.com/hyattpd/Prodigal)
- [GNU Parallel](https://github.com/Micromeda/pygenprop)

[InterProScan5](https://github.com/ebi-pf-team/interproscan) takes an organism's predicted proteins sequences as input. Genes must first be predicted using a gene prediction application (e.g., [https://en.wikipedia.org/wiki/List\_of\_gene\_prediction\_software](https://en.wikipedia.org/wiki/List_of_gene_prediction_software)) to get these proteins and then translated to proteins either using the same software or the second piece of software. Different types of software must be used on eukaryotic vs prokaryotic genomes. **For this tutorial, it is assumed that prokaryotic organisms are being analyzed. Prodigal is used to predict these organism's proteins.**

### Install Docker and Conda (optional)

InterProScan and Prodigal are more easily installed when you have the following installed:

- [Docker](https://docs.docker.com/install/)
- [Conda](https://docs.conda.io/projects/conda/en/latest/user-guide/install/)

### Install InterProScan

InterProScan can be [installed manually](https://github.com/ebi-pf-team/interproscan/wiki/HowToDownload) or it can be easily installed using our [Docker](https://www.docker.com)-based installation.

```shell
docker build https://raw.githubusercontent.com/Micromeda/InterProScan-Docker/master/Dockerfile -t micromeda/interproscan-docker
```

You should also install our wrapper script that allows you to run [InterProScan](https://github.com/ebi-pf-team/interproscan) on files found outside of its container.

```
wget https://raw.githubusercontent.com/Micromeda/InterProScan-Docker/master/run_docker_interproscan.sh
chmod +x run_docker_interproscan.sh
```

### Install Pygenprop

```shell
pip install numpy # Numpy needs to be installed separately
pip install pygenprop
```

### Install Prodigal (optional)
Prodigal can either be installed using the author's [installation tutorial](https://github.com/hyattpd/Prodigal/wiki/installation). Alternatively, Prodigal can be installed using [Conda](https://docs.conda.io/en/latest/).

```shell
conda install -c bioconda Prodigal
```

### Install GNU Parallel (optional)

GNU parallel can be used to parallelize some workflows such as gene prediction across multiple processes.

```shell
# Debian/Ubuntu
apt-get install parallel

# OSX
brew install parallel 
```

### Download The Genome Properties Database

```shell
wget https://raw.githubusercontent.com/ebi-pf-team/genome-properties/master/flatfiles/genomeProperties.txt
```


Create a Micromeda file for One Organism
------------------------------------------
Below is a tutorial that overviews how to build a single **Micromeda file** for one prokaryotic organism. Later we will discuss how to make a **Micromeda file** for multiple.

### 1. Predict Proteins
Use Prodigal to predict the organism's proteins. The ```-a``` flag is used to write the predicted protein sequences to a file.

```shell
prodigal -i ./genome.fasta -a ./genome.faa 
```

### 2. Remove ```*``` Characters
Prodigal adds ```*``` characters, representing stop codons, to the end of its output protein sequences. The ```*``` character is not in the [IUPAC protein alphabet](https://www.bioinformatics.org/sms2/iupac.html). As a result, [InterProScan5](https://github.com/ebi-pf-team/interproscan) throws an error when annotating Prodigal's output sequences. These ```*``` characters need to be removed to make the previously produced ```.faa``` file compatible.

Use Sed to remove these ```*``` characters. 

```shell
sed -i 's/\*$//' genome.faa
```

### 3. Annotate Domains
Use the [InterProScan5](https://github.com/ebi-pf-team/interproscan) [Docker](https://www.docker.com) container to domain annotate the previously sanitized ```.faa``` file. For convenience, one can use ```run_docker_interproscan.sh```, which simplifies using the container.

```shell
./run_docker_interproscan.sh genome.faa
```

This step produces an [InterProScan](https://github.com/ebi-pf-team/interproscan) ```.tsv``` annotation file called ```genome.tsv```.

### 4. Build a Micromeda File
When Pygenprop is installed, its [CLI](https://en.wikipedia.org/wiki/Command-line_interface) is also installed and can be used to build Micromeda files, among a few other functions. The [CLI](https://en.wikipedia.org/wiki/Command-line_interface)'s ```build``` command takes the previous output ```.tsv``` file generated by ```run_docker_interproscan.sh``` as input. 

```shell
pygenprop build -d ./genomeProperties.txt -i genome.tsv -o ./data.micro -p
```

The ```build``` command's `-p` flag is used to add protein sequences to the output Micromeda file. With this flag active, Pygenprop searches the [FASTA](https://en.wikipedia.org/wiki/FASTA_format) files that were scanned by [InterProScan](https://github.com/ebi-pf-team/interproscan) for proteins that support genome property steps and adds them to the output Micromeda file. **The [FASTA](https://en.wikipedia.org/wiki/FASTA_format) files must be in the same directory as the InterProScan files and share the same basename (e.g., filename without file extension).**

### 5. Overview

As discussed above, creating a Micromeda file involves converting an input genome files through a series of steps.

```
genome.fasta --> genome.faa --> genome.tsv --> data.micro
```


Create a Micromeda file for Multiple Organisms
----------------------------------------------
The above steps can be applied to multiple genomes to create more massive analysis datasets. Below we use ```parallel``` to parallelize specific steps across multiple cores. However, the same task could also be accomplished by placing the above commands, except for step four, in a shell script that loops through a series of input [FASTA](https://en.wikipedia.org/wiki/FASTA_format) file paths. 

For the steps below, let us assume that we have the following directory structure:

```
data/
├── ecoli_one.fasta
├── ecoli_two.fasta
```

### 1. Predict Proteins

The ```find . -maxdepth 1 -name "*.fasta"``` command finds all the [FASTA](https://en.wikipedia.org/wiki/FASTA_format) files in the current working directory. This list is piped to ```parallel```, which runs a Prodigal process on each file in parallel.

```shell
find . -maxdepth 1 -name "*.fasta" | parallel prodigal -i {} -a {.}.faa
```

***Resulting Folder Structure***

```
data/
├── ecoli_one.fasta
├── ecoli_one.faa
├── ecoli_two.fasta
├── ecoli_two.faa
```

### 2. Remove ```*``` Characters

Find all the ```.faa``` files and run Sed on them in parallel to remove ```*``` characters.

```shell
find . -maxdepth 1 -name "*.faa" | parallel sed -i 's/\*$//' {}
```

***Resulting Folder Structure***
N/A


### 3. Annotate Domains
Find all the ```.faa``` files and run [InterProScan](https://github.com/ebi-pf-team/interproscan) on them. The ```-j 1``` flag of ```parallel``` ensures that only one copy of InterProScan is run at a time (equivalent to [xargs](https://en.wikipedia.org/wiki/Xargs)).

```shell
find . -maxdepth 1 -name "*.faa" | parallel -j 1 ./run_docker_interproscan.sh {}
```

***Resulting Folder Structure***

```
data/
├── ecoli_one.fasta
├── ecoli_one.faa
├── ecoli_one.tsv
├── ecoli_two.fasta
├── ecoli_one.tsv
├── ecoli_two.faa
```

### 4. Build a Micromeda File
Build an output Micromeda file from multiple input [InterProScan](https://github.com/ebi-pf-team/interproscan) ```.tsv``` files.

```shell
pygenprop build -d ./genomeProperties.txt -i *.tsv -o ./data.micro -p
```

***Resulting Folder Structure***

```
data/
├── ecoli_one.fasta
├── ecoli_one.faa
├── ecoli_one.tsv
├── ecoli_two.fasta
├── ecoli_one.tsv
├── ecoli_two.faa
├── data.micro
```

Uploading a Micromeda File for Visualization
--------------------------------------------

