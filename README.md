# neb_udi_demux

Custom [Dorado](https://github.com/nanoporetech/dorado) barcode-kit definitions for
demultiplexing **UDI-indexed libraries sequenced on Oxford Nanopore flow cells**.

When ONT LSK (ligation sequencing kit) adapters are ligated directly onto a finished
short-read library, the P5/P7 adapters — and the unique dual indexes (UDIs) inside them —
are still present in the insert. The pool can therefore be demultiplexed on its **UDI
barcodes** rather than on native ONT barcodes. Dorado supports this through custom barcode
arrangements (`--barcode-arrangement` / `--barcode-sequences`); this repository provides
the arrangement files for the index sets below.

## NEB kits suported

| Kit name | Indexes | Index length | Adapter flanks | Index IDs |
|---|---|---|---|---|
| `E6441` | 96 UDI pairs | 8 bp | NEBNext | `7-001` … `7-098` |
| `E6443` | 96 UDI pairs | 8 bp | NEBNext | `7-099` … `7-194` |
| `E6445` | 96 UDI pairs | 8 bp | NEBNext | `7-197` … `7-292` |
| `E6447` | 96 UDI pairs | 8 bp | NEBNext | `7-297` … `7-393` |
| `E6449` | 96 UDI pairs | 8 bp | NEBNext | `7-396` … `7-491` |

The kit name is exactly the file basename — that is the string you pass to Dorado's
`--kit-name`.

### Files per kit

```
config/
  <KIT>.toml               # dorado barcode arrangement: flanking masks + scoring
  <KIT>_sequences.fasta    # 96 i7 + 96 i5 sequences, named <KIT>-i7-%03i / <KIT>-i5-%03i
  <KIT>_positions.json     # provenance: index ID (e.g. "7-062", "7-UDP0001") -> kit position
```

`<KIT>_positions.json` is **not read by Dorado**. It records which plate well / catalog
index ID each numbered kit position came from, so an arrangement can be regenerated or
audited. Only the `.toml` and `_sequences.fasta` are consumed at runtime.

### What the flanking masks are

Dorado locates a barcode by anchoring on the constant sequence around it. `mask1` is the
i7 (P7) side, `mask2` is the i5 (P5) side:

| | `mask1_front` | i7 | `mask1_rear` | | `mask2_front` | i5 | `mask2_rear` |
|---|---|---|---|---|---|---|---|
| **E644x** | `CTCCAGTCAC` | 8 bp | `ATCTCGTATG` | | `AGATCTACAC` | 8 bp | `ACACTCTTTC` |

The masks are the NEBNext adapter flanks and are identical across all five kits — only the
index sequences and IDs differ (see [NEB kits suported](#neb-kits-suported)). The `[scoring]` block
is likewise identical across all five and is tuned for the higher per-base error rate of
ONT reads (`max_barcode_penalty = 5`, `front/rear_barcode_window = 175`).

### Index pairing is strict

These are **unique dual indexes**: kit position *N* is the pair (i7-*N*, i5-*N*). A library
whose i7 resolves to one position and whose i5 resolves to another is not a valid member of
the kit and should be treated as an error, not silently assigned.

The i5 sequences in each FASTA are stored in one fixed orientation. Sample sheets report i5
in either orientation depending on the instrument that produced them (forward vs.
reverse-complement workflow), so if every i7 matches and no i5 does, reverse-complement the
`index2` column before comparing.

---

## Route 1: POD5 → basecall + demux

The cheapest option: Dorado classifies barcodes during basecalling, so reads come out of
`dorado basecaller` already tagged and `dorado demux` only has to split them.

```bash
KIT=E6441
KITS=/path/to/neb_udi_demux/config

dorado basecaller sup /path/to/run/pod5 \
    --kit-name            "$KIT" \
    --barcode-arrangement "$KITS/$KIT.toml" \
    --barcode-sequences   "$KITS/${KIT}_sequences.fasta" \
    --barcode-both-ends \
    > calls.bam

# --no-classify: reuse the classification already stored in calls.bam
dorado demux --no-classify --output-dir demux_bams calls.bam
```

`--barcode-both-ends` requires a barcode to be found at *both* ends of a read before that
read is assigned. For a UDI pool this is what you want — it is the property that makes the
dual index actually unique, and it sends index-hopped and chimeric reads to `unclassified`
instead of mis-assigning them. Omit it only if yield matters more than purity and you
accept the cross-talk.

---

## Route 2: FASTQ → demux

Use this when basecalling already happened without the custom kit — live basecalling on the
sequencer, a previous unbarcoded run, or data received from elsewhere. Classification now
has to happen at the demux step, so `--no-classify` is **omitted** and the kit flags move
onto `dorado demux`.

### 2a. Straight from FASTQ

```bash
KIT=E6441
KITS=/path/to/neb_udi_demux/config

dorado demux \
    --kit-name            "$KIT" \
    --barcode-arrangement "$KITS/$KIT.toml" \
    --barcode-sequences   "$KITS/${KIT}_sequences.fasta" \
    --barcode-both-ends \
    --emit-fastq \
    --output-dir demux_fastq \
    /path/to/run/fastq_pass
```

Dorado accepts a directory or individual FASTQ/BAM files as the final argument. Drop
`--emit-fastq` to get BAM out instead — recommended, since BAM keeps the `BC`/`RG` tags
recording the classification.

### 2b. FASTQ → uBAM → demux

Converting to unaligned BAM first preserves read tags and per-read metadata through the
whole chain:

```bash
samtools import -O BAM -0 reads.fastq.gz -o reads.ubam

dorado demux \
    --kit-name            "$KIT" \
    --barcode-arrangement "$KITS/$KIT.toml" \
    --barcode-sequences   "$KITS/${KIT}_sequences.fasta" \
    --barcode-both-ends \
    --output-dir demux_bams \
    reads.ubam
```

> **Note on already-split FASTQ.** If reads arrive already split into
> `fastq_pass/barcodeNN/`, those are *ONT native* barcodes, not UDIs. Merging them and
> running the steps above is the correct move; the existing `barcodeNN` directories carry
> no information about the UDI.

---

## Output layout

Dorado 2.x writes nested output:

```
demux_bams/{experiment}/{run}/bam_pass/{barcode}/{file}.bam
```

older versions write flat `demux_bams/{barcode}.bam`. `{barcode}` encodes the **kit
position**, zero-padded, alongside an `unclassified` bin for reads that failed the penalty
threshold or the both-ends requirement. Check the emitted directory names once per Dorado
version rather than assuming the padding width — the number in the name is the position,
and that is what you join on.
