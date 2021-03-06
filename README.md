[![rech.io](https://s3.amazonaws.com/get.rech.io/antigen.garnish_build_status.svg)](https://s3.amazonaws.com/get.rech.io/antigen.garnish.test.txt) | [![rech.io](https://img.shields.io/badge/endpoint.svg?url=https://s3.amazonaws.com/get.rech.io/antigen.garnish_coverage.json)](https://s3.amazonaws.com/get.rech.io/antigen.garnish_coverage.html) | ![](https://img.shields.io/badge/version-1.1.1-blue.svg) | ![](https://img.shields.io/docker/pulls/leeprichman/antigen_garnish.svg)

# antigen.garnish

Ensemble tumor neoantigen prediction and multi-parameter quality analysis from direct input, SNVs, indels, or gene fusion variants.

![](https://get.rech.io/antigen.garnish_flowchart.svg)

[Detailed flowchart.](https://get.rech.io/antigen.garnish_flowchart_detailed.svg)

## Description

An R package for [neoantigen](http://science.sciencemag.org/content/348/6230/69) analysis that takes human or murine DNA missense mutations, insertions, deletions, or RNASeq-derived gene fusions and performs ensemble neoantigen prediction using 7 algorithms. Input is a VCF file, [JAFFA](https://github.com/Oshlack/JAFFA) output, or table of peptides or transcripts. Outputs are ranked and summarized by sample. Neoantigens are ranked by MHC I/II binding affinity, clonality, RNA expression, similarity to known immunogenic antigens, and dissimilarity to the normal peptidome.

### Advantages

1. **Thoroughness**:
	* missense mutations, insertions, deletions, and gene fusions
	* human and mouse
	* ensemble MHC class I/II binding prediction using [mhcflurry](https://github.com/hammerlab/mhcflurry), [mhcnuggets](https://github.com/KarchinLab/mhcnuggets-2.0), [netMHC](http://www.cbs.dtu.dk/services/NetMHC/), [netMHCII](http://www.cbs.dtu.dk/services/NetMHCII/), [netMHCpan](http://www.cbs.dtu.dk/services/NetMHCpan/) and [netMHCIIpan](http://www.cbs.dtu.dk/services/NetMHCIIpan/i)
	* ranked by
		* MHC I/II binding affinity
		* clonality
		* RNA expression
		* similarity to known immunogenic antigens
		* dissimilarity to the normal peptidome
2. **Speed and simplicity**:
	* 1000 variants are ranked in a single step in less than five minutes
	* parallelized using [`parallel::mclapply`](https://stat.ethz.ch/R-manual/R-devel/library/parallel/html/mclapply.html) and [data.table::setDTthreads](https://github.com/Rdatatable/data.table/wiki), see respective links for information on setting multicore usage
3. **Integration with R/Bioconductor**
	* upstream/VCF processing
	* exploratory data analysis, visualization

## Installation

Three methods exist to run `antigen.garnish`:

1. Docker
2. Linux
3. Amazon Web Services

### Docker

```sh
docker pull leeprichman/antigen_garnish
```

See the [wiki](https://github.com/immune-health/antigen.garnish/wiki/Docker) for instructions to run the Docker container.

### Linux

#### Requirements

- R &ge; 3.4
- python-pip
- tcsh (required for netMHC)
- `sudo` privileges (required for netMHC)

#### Installation script

The following line downloads and runs the initial [installation script](http://get.rech.io/install_antigen.garnish.sh).

```sh
$ curl -fsSL http://get.rech.io/install_antigen.garnish.sh | sudo sh
```

Next, download the netMHC suite of tools for Linux, available under an academic license:

* [netMHC](http://www.cbs.dtu.dk/cgi-bin/nph-sw_request?netMHC)
* [netMHCpan](http://www.cbs.dtu.dk/cgi-bin/nph-sw_request?netMHCpan)
* [netMHCII](http://www.cbs.dtu.dk/cgi-bin/nph-sw_request?netMHCII)
* [netMHCIIpan](http://www.cbs.dtu.dk/cgi-bin/nph-sw_request?netMHCIIpan)

After downloading the files above, move the binaries into the `antigen.garnish` data directory, first setting the `NET_MHC_DIR` and `ANTIGEN_GARNISH_DIR` environmental variables, as shown here:

```sh

NET_MHC_DIR=/path/to/folder/containing/netMHC/downloads
ANTIGEN_GARNISH_DIR=/path/to/antigen.garnish/data/directory

cd "$NET_MHC_DIR" || return 1

mkdir -p "$ANTIGEN_GARNISH_DIR/netMHC" || return 1

find . -name "netMHC*.tar.gz" -exec tar xvzf {} -C "$ANTIGEN_GARNISH_DIR/netMHC" \;

chown "$USER" "$ANTIGEN_GARNISH_DIR/netMHC"
chmod 700 -R "$ANTIGEN_GARNISH_DIR/netMHC"

```

### Amazon Web Services

See the [wiki](https://github.com/immune-health/antigen.garnish/wiki/antigen.garnish-on-AWS) for instructions to create an Amazon Web Services instance.

## Package documentation

Package documentation can be found: [website](https://neoantigens.rech.io/reference/index.html), [pdf](https://get.rech.io/antigen.garnish.pdf).

### Workflow example

  1. Prepare input for MHC affinity prediction and quality analysis:

		* VCF input - `garnish_variants`
		* Fusions from RNASeq via [JAFFA](https://github.com/Oshlack/JAFFA)- `garnish_jaffa`
		* Prepare table of direct transcript or peptide input - see manual page in R (`?garnish_affinity`)

  1. Add MHC alleles of interest - see examples below.
  1. Run ensemble prediction method and perform antigen quality analysis including proteome-wide differential agretopicity, IEDB alignment score, and dissimilarity: `garnish_affinity`.
  1. Summarize output by sample level with `garnish_summary` and `garnish_plot`, and prioritize the highest quality neoantigens per clone and sample with `garnish_antigens`.

### Function examples

#### Predict neoantigens from missense mutations, insertions, and deletions

```r
library(magrittr)
library(data.table)
library(antigen.garnish)

  # load an example VCF
	dir <- system.file(package = "antigen.garnish") %>%
		file.path(., "extdata/testdata")

	dt <- "antigen.garnish_example.vcf" %>%
	file.path(dir, .) %>%

  # extract variants
    garnish_variants %>%

  # add space separated MHC types

  # see list_mhc() for nomenclature of supported alleles

	# MHC may also be set to "all_human" or "all_mouse" to use all supported alleles

      .[, MHC := c("HLA-A*01:47 HLA-A*02:01 HLA-DRB1*14:67")] %>%

  # predict neoantigens
    garnish_affinity

  # summarize predictions
    dt %>%
      garnish_summary %T>%
        print

  # generate summary graphs
    dt %>% garnish_plot
```

#### Predict neoantigens from gene fusions

```r
library(magrittr)
library(data.table)
library(antigen.garnish)

  # load example jaffa output
	dir <- system.file(package = "antigen.garnish") %>%
		file.path(., "extdata/testdata")

	path <- "antigen.garnish_jaffa_results.csv" %>%
			file.path(dir, .)
	fasta_path <- "antigen.garnish_jaffa_results.fasta" %>%
			file.path(dir, .)

  # get predictions
    dt <- garnish_jaffa(path, db = "GRCm38", fasta_path) %>%

  # add MHC info with list_mhc() compatible names
    .[, MHC := "H-2-Kb"] %>%

  # get predictions
    garnish_affinity %>%

  # summarize predictions
    garnish_summary %T>%
    print
```

#### Get full MHC affinity output from a Microsoft Excel file of variants

```r
library(magrittr)
library(data.table)
library(antigen.garnish)

  # load example Microsoft Excel file
  dir <- system.file(package = "antigen.garnish") %>%
    file.path(., "extdata/testdata")

  path <- "antigen.garnish_test_input.xlsx" %>%
    file.path(dir, .)

  # predict neoantigens
    dt <- garnish_affinity(path = path) %T>%
      str
```

#### Directly calculate IEDB score and dissimilarity for a list of sequences

```r
library(magrittr)
library(data.table)
library(antigen.garnish)

  # generate our character vector of sequences
  v <- c("SIINFEKL", "ILAKFLHWL", "GILGFVFTL")

  # calculate IEDB score
  v %>% iedb_score(db = "human") %>% print

	# calculate dissimilarity
	v %>% garnish_dissimilarity(db = "human") %>% print
```

#### Automated testing

From ./`<Github repo>`:

```r
  devtools::test(reporter = "summary")
```

#### How are peptides generated?

```r
  library(magrittr)
  library(data.table)
  library(antigen.garnish)

  # generate a fake peptide
    dt <- data.table::data.table(
       pep_base = "Y___*___THIS_IS_________*___A_CODE_TEST!______*__X",
       mutant_index = c(5, 25, 47, 50),
       pep_type = "test",
       var_uuid = c(
                    "front_truncate",
                    "middle",
                    "back_truncate",
                    "end")) %>%
  # create nmers
    make_nmers %T>% print
```

## Plots and summary tables

* `garnish_plot` output:

![](https://get.rech.io/antigen.garnish_example_plot.png)

* `garnish_antigens` output:

![](https://get.rech.io/antigen.garnish_summary_table.png)

## Citation

> Richman LP, Vonderheide RH, and Rech AJ. "Neoantigen dissimilarity to the self-proteome predicts immunogenicity and response to immune checkpoint blockade." *Cell Systems* **9**, 375-382.E4, (2019).

## Contributing

We welcome contributions and feedback via [Github](https://github.com/immune-health/antigen.garnish/issues) or [email](mailto:leepr@upenn.edu).

## Acknowledgments

We thank the follow individuals for contributions and helpful discussion:

- [David Balli](https://www.linkedin.com/in/davidballi1)
- [Adham Bear](https://www.med.upenn.edu/apps/faculty/index.php/g20001100/p1073)
- [Katelyn Byrne](https://www.parkerici.org/person/katelyn-byrne/)
- [Danny Wells](http://dannykwells.com/)

## License

Please see [LICENSE](https://github.com/immune-health/antigen.garnish/blob/master/LICENSE).
