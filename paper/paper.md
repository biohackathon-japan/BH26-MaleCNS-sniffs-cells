---
title: 'DBCLS BioHackathon 2026 report: A fly connectome as a novelty detector for single-cell transcriptomes'
title_short: 'BioHackJP26: MaleCNS sniffs cells'
tags:
  - Connectome
  - Single-cell RNA-seq
  - Spiking neural network
  - Cancer
authors:
  - name: Jiandong Chen
    affiliation: 1
    role: Conceptualization, Development, Writing - original draft
  - name: Masao Nagasaki
    affiliation: 1
    role: Reviewing
affiliations:
  - name: Division of Biomedical Information Analysis, Medical Institute of Bioregulation, Kyushu University
    index: 1
date: 18 September 2026
cito-bibliography: paper.bib
event: BH26JP
biohackathon_name: "DBCLS BioHackathon 2026"
biohackathon_url:   "https://2026.biohackathon.org/"
biohackathon_location: "Matsuyama, Japan, 2026"
group: MaleCNS-sniffs-cells
# URL to project git repo --- should contain the actual paper.md:
git_url: https://github.com/biohackathon-japan/BH26-MaleCNS-sniffs-cells
# This is the short authors description that is used at the
# bottom of the generated paper (typically the first two authors):
authors_short: Jiandong C \emph{et al.}
---


# Introduction

MaleCNS v1.0 is the synapse-level wiring diagram of the complete central nervous system of an adult male
Drosophila melanogaster. It consists of 166,700 neurons and 124 million synaptic contacts, with cell-type names,
neurotransmitter predictions and soma positions for every neuron [@citesAsDataSource:MaleCNS]. Together with
whole-brain leaky integrate-and-fire (LIF) models that reproduce sensorimotor behaviour from the wiring alone
[@usesMethodIn:Shiu2024], it makes it possible to run a fly brain as a program.

At the DBCLS BioHackathon 2026 we asked whether such a brain can do useful work on genomics data. The fly's
mushroom body turns an odor into a sparse code over ~2,000 Kenyon cells (KCs), which is a locality-sensitive hash
[@citesAsAuthority:Dasgupta2017], and its α'3 compartment implements a novelty detector, in which the output neurons
MBON-α'3 respond strongly to a new odor and are suppressed within a few repetitions by dopamine from PPL1-α'3,
odor-specifically, recovering over an hour [@citesAsEvidence:Hattori2017]. We present single-cell RNA-seq
profiles to the simulated brain as odors and read the α'3 output as a novelty score. Cells resembling those
the brain has already smelled habituate, while cells unlike them stay novel.

Here we report three stages of the experiments. On PBMC 3k, we verified that the circuit habituates to recurring cell types. 
On the lung adenocarcinoma dataset GSE131907, the brain learns normal lung epithelium and then scores tumour and metastases.
Using a Smart-seq2 lung cancer dataset with per-cell DNA evidence of driver mutations, we compared the novelty method with
inferCNV against a mutation-based truth. The source code is available on https://github.com/stahiga/fly-sniff-scRNA.

# Model

## Connectome and dynamics

From the MaleCNS v1.0 flat-connectome tables we keep every body with an assigned
superclass except glia (166,700 neurons, 25.6 million directed connections). Every neuron is an identical LIF unit
(rest −52 mV, threshold −45 mV, τ~m~ 20 ms, τ~syn~ 5 ms, delay 1.8 ms, refractory 2.2 ms, 0.275 mV per synaptic
contact) as described in Shiu et al. [@usesMethodIn:Shiu2024], and the sign of each connection follows the presynaptic neuron's
consensus transmitter. The LIF kernel was adapted from the open-source stonkfly project [@usesMethodIn:stonkfly]. Two
corrections to the tables were needed to obtain a silent network after odor offset. Cells with an unresolved
transmitter (2,999) are treated as inhibitory, and the 30 antennal-lobe local neurons `lLN1_bc`, predicted
cholinergic but placing 38% of their output synapses on each other, are set to GABA following Chou et al.
[@citesAsAuthority:Chou2010], since as predicted cholinergic they saturated at 340 Hz. Feedback inhibition from the APL neuron onto
KCs is scaled by 0.25 so that one odor activates 8% of KCs (5 to 10% in vivo [@citesAsEvidence:Honegger2011]).

## Input

A cell's expression profile are encoded into a 53-dimensional vector in [0, 1], corresponding with the number of antennal-lobe glomerulus.
The signal is injected as a constant current (25 mV × value) into all olfactory receptor neurons of that glomerulus for 1 s. Since the
assignment of features to glomeruli does not have specific biological meaning, five runs with random shuffling of assignment rules are applied.

## Circuit and readout

Figure \ref{fig:brain} shows the odor-mushroom body circuit. The plastic synapses are the 1,132 connections from the 695 α'β' KCs onto the four α'3
output neurons (MBON16, MBON17 in MaleCNS nomenclature). The dopamine neuron is PPL104 (PPL1-α'3), which receives
a 7 mV tonic depolarisation so that it is silent without odor and responds to any odor, as observed in vivo.

The settings for learning follows Hattori et al., that when a Kenyon cell and PPL104 are active at the same time, the synapses from that Kenyon cell onto the MBONs are depressed by Δw = −η · KC trace · PPL104 rate. The learning rate η = 0.0005 reproduces the habituation curve in Fig. 1C of Hattori et al. The metric is the total synaptic drive that the MBONs receive from the α'β' Kenyon cells in mV. This is the quantity that Hattori et al. measured as calcium. For each cell, we compute this drive twice from the same spikes, once with the learned weights and another with the naive weights to compute the novelty score, which is the first value divided by the second.

![Where the novelty circuit sits in the MaleCNS brain. Soma positions of all brain neurons in dorsal view (grey), with the
neurons of the olfactory pathway coloured: olfactory receptor neurons and projection neurons in the antennal lobe,
the Kenyon cells of the mushroom body with the α'β' subset in purple, the APL feedback neuron, and the four α'3
output neurons (MBON-α'3) with the dopamine neuron PPL1-α'3 beside them. Only the α'β' Kenyon cell to MBON-α'3
synapses are plastic. The ventral nerve cord is simulated but lies outside the frame.
\label{fig:brain}](./brain_map.png)

# Habituation to recurring cell types (PBMC 3k)

## Data and encoding

The 2,638 cells of the 10x PBMC 3k dataset with scanpy's louvain labels (8 cell types).
Each cell is encoded by 53 marker genes (top genes of the 8 clusters, round-robin), each scaled to [0, 1] by its
99th percentile.

## The Kenyon-cell code keeps most of the cell-type information

With learning off, each cell activates 91 KCs on average (2.2%). To test whether this sparse code retains cell-type information, we predicted each cell's louvain label from its binary Kenyon-cell code by leave-one-out kNN with k = 15, obtaining accuracy of 0.884.The same classifier reaches 0.926 on the 53-gene input and 0.953 on the first 50 principal components, and the majority-class baseline is 0.43. The loss is largest for rare types, and for dendritic cells the accuracy falls from 0.59 on the gene input to 0.35 on the Kenyon-cell code.

## Every cell type habituates, and the spread between encodings comes from the arbitrary glomerulus assignment

Starting from naive weights, we present all cells of one type once each in random order, in five rounds of N/5 cells. Table 1 gives the mean novelty in the first and the last round, and Figure \ref{fig:pbmc} gives the full curves. Over the five rounds, the mean novelty falls from 0.31–0.74 in the first round to 0.14–0.35 in the fifth, depending on the type. The assignment of genes to glomeruli has no biological meaning, and we observed that a particular assignment can leave PPL104 undriven for a whole cell type. We therefore shuffle the assignment and report the average over five shuffles.

| Type | n | Round 1 | Round 5 |
|---|---|---|---|
| B | 342 | 0.31 ± 0.13 | 0.14 ± 0.03 |
| CD14+ monocyte | 480 | 0.40 ± 0.25 | 0.22 ± 0.20 |
| CD4 T | 1144 | 0.43 ± 0.07 | 0.19 ± 0.03 |
| CD8 T | 316 | 0.54 ± 0.24 | 0.26 ± 0.13 |
| NK | 154 | 0.38 ± 0.24 | 0.20 ± 0.12 |
| FCGR3A+ monocyte | 150 | 0.57 ± 0.23 | 0.30 ± 0.17 |
| Dendritic | 37 | 0.67 ± 0.24 | 0.35 ± 0.24 |
| Megakaryocyte | 15 | 0.74 ± 0.24 | 0.26 ± 0.17 |

Table: Each immune cell type habituates within its own sequence. For each type, the table gives the mean novelty of the cells presented in the first round and in the fifth round when that type alone is presented from naive weights. Values are mean ± sd over the five gene-to-glomerulus permutations.

![Novelty falls within the first tens of cells of a type and then plateaus. For each of the eight louvain cell types, all cells of the type are presented once each in random order from naive weights, and the novelty of each cell is plotted against the number of cells of that type presented before it. Thin lines are the five gene-to-glomerulus permutations and the thick line is their mean. Both are smoothed with a running mean over one fifteenth of the cells. For every type, novelty falls within the first few tens of cells and then settles between 0.1 and 0.3. \label{fig:pbmc}](./pbmc_habituation.png)

## Viewer

An interactive page shows the spiking activity of the whole CNS while a cell is presented, a fly body
driven by the motor and descending neurons of each frame, and the α'3 readout
(https://res.stahiga.cc/pub/CyberKOBAE_sniff_to_find_cells_PBMC/).

# Recognising tumour cells by learning normal cells (lung adenocarcinoma, GSE131907)

## Data and encoding

Kim et al. [@citesAsDataSource:Kim2020] profiled 208,506 cells from 44 lung adenocarcinoma patients with 10x Chromium. We use the 36,467 epithelial cells. By tissue of origin they are normal lung (3,703), primary tumour (7,270), advanced or bronchial tumour (6,582), lymph-node metastasis (3,053), brain metastasis (15,463) and pleural effusion (396). To encode a cell, we normalise and log-transform the counts, select the 2,000 most variable genes over all epithelial cells, and fit a non-negative matrix factorisation (NMF) with 53 components. Each component is scaled to [0, 1] by its 99th percentile across cells, and the 53 scaled loadings of a cell are its glomerular values.

## The brain responds with higher novelty score to tumour cells compared to normal lung

For each of five permutations, 600 randomly chosen normal-lung cells (from 11 patients) are
presented once each with learning on, after which the weights are frozen and all 36,467 cells are presented once. In a
second condition the brain learns 600 primary-tumour cells instead. Table 2 and Figure \ref{fig:luad} give the novelty by origin,
excluding the cells that were used for learning.

| Learned on | Normal lung | Primary | Bronchial | Lymph node | Effusion | Brain |
|---|---|---|---|---|---|---|
| 600 normal lung | 0.11 | 0.20 | 0.23 | 0.22 | 0.23 | 0.32 |
| 600 primary tumour | 0.14 | 0.11 | 0.18 | 0.16 | 0.20 | 0.27 |

Table: Tumour cells of every origin are more novel than the lung tissue the brain has learned. Each row gives the median novelty of the epithelial cells of each tissue of origin after the brain has learned 600 cells of the origin named in the first column. The cells used for learning are excluded, and the medians are averaged over the five permutations.

![Tumour cells remain novel to a brain that has learned normal lung, while normal lung is only slightly novel to a brain that has learned the primary tumour. Each violin shows the distribution of the novelty score over all epithelial cells of one tissue of origin after the brain has learned 600 normal-lung cells (left) or 600 primary-tumour cells (right), averaged over five permutations. Grey marks the origin that was learned, and the cells used for learning are excluded. The horizontal bar is the median, the vertical bar is the interquartile range, and the number beside each bar is the median. \label{fig:luad}](./luad_novelty.png)

After learning normal lung, the brain scores every tumour origin above normal lung, and it scores the brain metastases highest. The two conditions are not symmetric. A brain that has learned the primary tumour finds normal lung only slightly novel (0.14), whereas a brain that has learned normal lung finds the primary tumour clearly novel (0.20). This is what one expects if tumour cells span a wider region of expression space than normal epithelium.

## Normal epithelium learned from other patients transfers to a held-out patient

To test whether what the brain learns from normal epithelium transfers between patients, we held out one patient at a time. The brain learned 600 normal-lung cells drawn from the other patients, and the held-out patient's cells were then scored. For each of the 11 patients with normal lung, the held-out normal cells had a median novelty between 0.10 and 0.12, while the same patient's primary tumour cells scored between 0.11 and 0.33 (Table 3).

| Held-out patient | 01 | 06 | 08 | 09 | 18 | 19 | 20 | 28 | 30 | 31 | 34 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Normal lung | 0.11 | 0.11 | 0.11 | 0.11 | 0.11 | 0.10 | 0.11 | 0.11 | 0.12 | 0.11 | 0.11 |
| Primary tumour | n/a | 0.13 | 0.33 | 0.11 | 0.26 | 0.13 | 0.15 | 0.33 | 0.19 | 0.33 | 0.20 |

Table: Normal lung learned from other patients is not novel to a held-out patient, while that patient's tumour is. Each column gives, for one held-out patient, the median novelty of the patient's normal-lung cells and of the patient's primary-tumour cells after the brain has learned normal lung from the other ten patients.

## The novelty score alone separates tumour samples from normal lung

We fitted a class-balanced logistic regression on the novelty score alone, evaluated with patients held out, which reaches an AUROC of 0.975 with a decision threshold of 0.16. It calls 3.8% of the normal-lung cells of held-out patients tumour, and 88.5% of the cells from tumour samples tumour (primary 72%, bronchial 89%, lymph node 83%, effusion 82%, brain 97%). On the same cells we ran inferCNV [@usesMethodIn:infercnvpy] with all normal-lung cells as reference, a window of 100 genes, a step of 10, and the mean absolute CNV per cell as its score. It reaches an AUROC of 0.882. At 95% specificity on normal lung, inferCNV detects 48% of the tumour-sample cells and the novelty score 91%. The two scores have a Spearman correlation of 0.52.


# Validation on driver mutation-carrying tumour cells (lung cancer, Maynard et al. 2020)

## Motivation

In GSE131907 a cell is labelled by the sample it came from, and the tumour samples contain normal epithelium in unknown proportion. Neither the novelty score nor inferCNV can therefore be evaluated on single cells. A cell-level truth needs genomic evidence in the same cell. Maynard et al. [@citesAsDataSource:Maynard2020] sequenced 23,420 cells from 49 biopsies of 30 patients with advanced non-small-cell lung cancer using Smart-seq2. Its full-length reads can cover the driver mutation that was identified clinically in each patient, so a cell in which the mutant allele is read is a tumour cell by DNA rather than by inference.

## Data

We took the counts and cell-type annotations of the 4,387 epithelial cells of this dataset from the LuCA lung cancer atlas [@citesAsDataSource:Salcher2022], where each cell is identified by its SRA run accession. Of these, 3,338 cells come from patients whose driver is a point mutation or a small deletion (EGFR L858R, EGFR exon-19 deletions, EGFR T790M, KRAS G12C/G12D/G12V/G13D, BRAF V600E). For these cells we downloaded the raw reads from ENA (PRJNA591860), aligned them with minimap2 to the three genes, and counted the reads at the patient's driver locus. We call a cell with at least one mutant read a DNA-proven tumour cell. Among the 2,759 cells processed, 102 are DNA-proven tumour cells, of which LuCA annotates 94 as tumour cells and 8 as normal epithelium.

The study also includes two normal-adjacent-tissue biopsies (TH238, TH179) that are not part of LuCA. We located their cells in ENA through the biopsy identifier and quantified 492 of the 1,029 cells by alignment to GENCODE v38 protein-coding transcripts. Of these, 84 are epithelial by marker genes (EPCAM, KRT8/18/19, SFTPC, SFTPB, NAPSA, AGER, SCGB1A1, FOXJ1 against PTPRC, CD3E, CD68, LYZ, COL1A1, PECAM1, VWF, MS4A1, CD79A, NKG7). These 84 cells are the normal epithelium defined by tissue of origin.

## Method

We encoded the Maynard epithelial cells as in the previous section, with an NMF of 53 components fitted on their 2,000 most variable genes, and projected the normal-adjacent cells onto the same components. We split the 84 normal cells at random into two halves of 42. The brain learned one half, with one presentation of each cell and five permutations, and its weights were then frozen. We scored the 102 DNA-proven tumour cells and the other 42 normal cells. We ran inferCNV on the same cells with the same 42 normal cells as reference. Both methods were evaluated as tumour versus held-out normal, with the decision threshold set at 95% specificity on the held-out normal cells. The split was repeated five times.

## The novelty score recovers 89% of the DNA-proven tumour cells and inferCNV recovers all of them

Table 4 and Figure \ref{fig:roc} give the outcome. The novelty score separates the DNA-proven tumour cells from the held-out normal epithelium with an AUROC of 0.98 and recalls 89% of them at 95% specificity. inferCNV separates the two groups completely.

| | AUROC | Precision | Recall |
|---|---|---|---|
| Novelty score | 0.982 ± 0.009 | 0.97 | 0.89 ± 0.04 |
| inferCNV | 1.000 ± 0.000 | 0.97 | 1.00 ± 0.00 |

Table: Both methods separate DNA-proven tumour cells from held-out normal epithelium. The 102 DNA-proven tumour cells are scored against 42 held-out normal-adjacent epithelial cells. The other 42 normal cells serve as the brain's training set and as inferCNV's reference. Precision and recall are taken at 95% specificity, and all values are mean ± sd over five random splits.

![ROC curves of the novelty score and of inferCNV on the same test. The 102 DNA-proven tumour cells are the
positives and the 42 held-out normal-adjacent epithelial cells the negatives. Thin lines are the five random
splits and thick lines their mean. The dotted line marks 95% specificity, where the precision and recall in
Table 4 are taken. \label{fig:roc}](./maynard_roc.png){ width=70% }

The normal cells come from two biopsies and the tumour cells from other biopsies. Differences between biopsies therefore contribute to the separation obtained by both methods.

# Discussion

The recent release of MaleCNS v1.0 made a complete adult fly nervous system available as a program. We took this BioHackathon as the occasion to try it on the recognition of tumour cells in single-cell transcriptomes which still remains a task lacking a broadly applicable gold standard. The premise was that a behavioural response of the fly can be put to use in data analysis when the setting and the task are chosen to match it. The response we used is the loss of response of the mushroom body's α'3 compartment to an odor that recurs. In this task the compartment acts as a state machine with a non-linear decay. It loses its response to what it has smelled before and keeps its response to what it has not. It is encouraging that a brain of this complexity, in which the signal is carried by analogue quantities such as sub-threshold synaptic drive, reaches a high performance on the task.

In a connectome composed of 124 million connections, only 1,132 synapses change in this model. The neuron model is the same for every cell, and the two constants that we set, the learning rate and the scale of APL feedback, were calibrated against fly physiology. The same brain habituates to immune cell types, to normal lung and to primary tumour.

The novelty score complements the most widely used method inferCNV. The inferCNV detects copy-number changes from expression shifts along the genome. The novelty score compares the whole expression profile with what the brain has learned and makes no use of genomic position. On GSE131907 the two scores correlate at 0.5, and the novelty score demonstrated a higher sensitivity than inferCNV, while on the DNA-verified cells inferCNV separated tumour from normal better than the novelty score. A tumour without copy-number changes should be detected by the novelty score and missed by inferCNV. The training set largely defines what the brain can recognise, and a training set that contains tumour cells lowers the recall.

The MaleCNS offers mechanisms that could be used for multiple purposes. For example, the recovery of the α'3 synapses over an hour is a forgetting mechanism. It could play the part that weight resets play in artificial networks and keep the brain from committing too strongly to what it has already seen. Addtionally, the escape response to a looming stimulus is a switch that fires on the rate of change of a signal, and it could detect state transitions in biological time series. Another example is the optomotor response which integrates weak and noisy motion cues across the visual field into a single direction, and it could extract a consistent direction from many weak pieces of evidence. We expect that developing such kind of biological brain into a toolkit that matches to a class of biological questions, will give researchers a new and well-characterised source of computational tools for their data.


## Acknowledgements

This work was carried out at the DBCLS BioHackathon 2026 in Matsuyama, and we deeply thank the organizers for holding the
event. We thank Dr. Deepa Kasaragod of Hiroshima University for her interest in this project and her valuable feedback. 
We thank the FlyEM project team for releasing MaleCNS v1.0 under CC-BY, and the authors of the *stonkfly* and *Neural
Canvas* projects for open-source LIF kernel, plasticity code and NeuroMechFly body.
Single-cell data were provided by 10x Genomics (PBMC 3k), Kim et al. (GSE131907), Maynard et al. (PRJNA591860)
and the LuCA atlas (Salcher et al.).　

# References
