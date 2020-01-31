
# Introduction


This is a project to synthesize large amounts of MIDI data into audio files. It is built with 
the [Lakh MIDI Dataset v0.1](https://colinraffel.com/projects/lmd/) (LMD). We call this
Synthesized Lakh, or Slakh for short. This repository contains everything you need to use
Native Instruments' Kontakt or other VSTs to synthesize large amounts of musical data.


A full dataset of synthesized data is available for download at the 
[Slakh project website](www.slakh.com/).


## Table of Contents

1. [Overview](#overview)
2. [About the Slakh Dataset](#about-the-slakh-dataset)
3. [License and Attribution](#license-and-attribution)
4. [Generating Data](#generating-data)
    <br>&emsp;a. [Step 1: Setting up RenderMan on Mac](#step-1-setting-up-renderman-on-mac)
    <br>&emsp;b. [Step 2: Install/Setup Kontakt and/or other Synthesis VSTs](#step-2-installsetup-kontakt-andor-other-synthesis-vsts)
    <br>&emsp;c. [Step 3: Set up the json files](#step-3-set-up-the-json-files)
    <br>&emsp;c. [Step 4: Running data generation script](#step-4-running-data-generation-script)<br>
5. [Flakh & using FluidSynth](#flakh--using-fluidsynth)
6. [Gotchas](#gotchas)


## Overview

This repository contains all of the code, config files, and patches used to create the original 
Slakh2100 dataset. The details regarding all of the settings used to make Slakh2100 are included in
the file MakingSlakh.md included in this repo. This guide assumes you have downloaded the Lakh MIDI
Dataset (LMD) prior to using it.

At the base of creating Slakh is a script called `render_by_instrument.py`, which looks for MIDI
files that match a user-definable "band" definition, synthesizes each instrument separately using
a VST host, and then mixes the resultant audio to have equal loudness according to the ITU-R 
BS.1770-4 loudness standard.

The VST host used is called RenderMan. RenderMan is a C++ VST host (built on JUCE) that has python
bindings. The source code is included here (in `RenderMan-master/`). The Linux and Mac versions of
RenderMan both work, but Kontakt only runs on Mac and Windows. So **this project is Mac only!!!**


## About the Slakh Dataset

The Synthesized Lakh (Slakh) Dataset is a new dataset for audio source separation that is 
synthesized from the [Lakh MIDI Dataset v0.1](https://colinraffel.com/projects/lmd/) using 
professional-grade sample-based virtual instruments.  **Slakh2100** contains 2100 automatically
mixed tracks and accompanying MIDI files synthesized using a professional-grade sampling engine.
The tracks in **Slakh2100** are split into training (1500 tracks), validation (375 tracks), and 
test (225 tracks) subsets, totaling **145 hours** of mixtures.

Slakh is brought to you by [Mitsubishi Electric Research Lab (MERL)](http://www.merl.com/) and 
the [Interactive Audio Lab at Northwestern University](http://music.cs.northwestern.edu/). For 
more info, please visit [the Slakh website](www.slakh.com/).


## License and Attribution
  
If you use Slakh2100 or generate data using the same method we ask that you cite it using the 
following bibtex entry:

```
@inproceedings{manilow2019cutting,
  title={Cutting Music Source Separation Some {Slakh}: A Dataset to Study the Impact of Training Data Quality and Quantity},
  author={Manilow, Ethan and Wichern, Gordon and Seetharaman, Prem and Le Roux, Jonathan},
  booktitle={Proc. IEEE Workshop on Applications of Signal Processing to Audio and Acoustics (WASPAA)},
  year={2019},
  organization={IEEE}
}

```

Slakh2100 and Flakh2100 are licensed under a 
<a rel="license" href="http://creativecommons.org/licenses/by/4.0/">Creative Commons Attribution 
4.0 International License</a>.

This code is licensed under an MIT license.


# Generating Data


## Step 1: Setting up RenderMan on Mac


RenderMan originally comes from [here](https://github.com/fedden/RenderMan), but the version used
here is based on a forked version that allows reading of entire MIDI files at once. That's from
[here](https://github.com/cannoneyed/RenderMan). This modified version is included in this repo
(at `RenderMan-master/`).


- Open the RenderMan XCode project (`RenderMan-master/Builds/MacOSX/RenderMan.xcodeproj`),
it should be configured correctly to build. 

- One common issue we ran into is not having Boost python installed or set up correctly (bindings
between C++ and python). The easiest way to install it is with `homebrew`; make sure you get the
python2 version (not sure if RenderMan has been tested with python3). The command is just 
`brew install boost-python`.

- Then you need to make sure the XCode build settings are correct. The most common issue we 
encountered was a linker issue--making sure that boost-python was able to find and build against
the system's python dynamic libraries. To fix this, check your build settings in XCode. Double
check that the build target is a dynamic library, and under the 'Target' build settings look for
the 'Linker' heading. Make sure 'Other Linker Flags' has the following setting: 
`-shared -lpython2.7 -lboost_python27`.

- Press `[cmd]`-`[b]` to build.

- If it builds successfully, you should have a file called `librenderman.so.dylib` 
(in `RenderMan-master/Builds/MacOSX/build/Debug/` or `Release`). Rename that file to
`librenderman.so`. And now RenderMan is set up!


## Step 2: Install/Setup Kontakt and/or other Synthesis VSTs

See Native Instruments' website for details on installing Kontakt. I don't believe there are any
restrictions on which Kontakt build version to use, because they all conform to the VST API.

We used Kontakt Komplete 12, but any version of Kontakt with some instrument packs will work.
You will need to know the absolute path to the `Kontakt.component`, and the path to any other
`.component` files for every VST that you use.  (Usually found in 
`/Library/Audio/Plug-Ins/Components/`)

### Step 2a: Make Kontakt patches/presets

The process for making Kontakt patches is a little involved so I'll outline it here. This
doesn't apply for other VSTs, which might have their own method for storing and recalling
presents.

Using Kontakt Player 6:
1. Open Kontakt, find a sampler/patch that you like.
2. Click the floppy disk icon in the top bar.
3. Select "Save Multi as...". Note this will save all of the .nki instruments and settings 
currently loaded.
4. Save. Keep track of this location.


**NOTE: We used a Kontakt Komplete 12 (2018), which may have patches (`.nkm` files) that might
not be present on your system. Make sure to check prior to running.** 



## Step 3: Set up the json files

### Step 3a: Instrument Definition Metadata File

There are two json files that you'll need to set up to run the script. 

The first is the *Instrument Definition Metadata File* (the `"defs_metadata_file"` in the config 
json file). This file defines how MIDI Instrument program numbers get mapped to Kontakt or VST
patches.


Annotated:
```
{
  "{{Instrument Name}}" : {
    "class": "{{Instrument MIDI Class}}",
    "program_numbers" : [0, 1, 2, 3, 4], # MIDI Instrument program numbers
    "defs": [    # List of nkm files or paths to .component files
      "{{Kontakt patch}}.nkm",
      "{{another Kontakt patch}}.nkm",
      "{{absoulte path to a VST}}.component",
      "{{another absoulte path to a VST}}.component"
    ]
  },
  ...
}
```

`"program_numbers`: A list of MIDI Instrument program numbers for this patch. See 
`general_midi_inst_*based.txt` for a readable layout of MIDI Instrument program numbers. Any time
a number from this list is seen, a random VST from the `"defs"` list will be selected for
synthesis. 

`"defs"`: A list containing either ".nkm" filenames (in the `user_defs_dir` specified in
`config.json`), or absolute paths to VST `.component` files.



Example from `classes_strict.json`:
```json
{
  "Acoustic Piano" : {
    "class": "Piano",
    "program_numbers" : [0, 1, 2, 4, 7],
    "defs": [
      "ragtime_piano.nkm",
      "the_giant_hard_and_tough.nkm",
      "the_giant_modern_studio.nkm",
      "the_giant_vibrant.nkm",
      "alicias_keys.nkm",
      "the_grandeur.nkm",
      "the_gentleman.nkm",
      "grand_piano.nkm",
      "upright_piano.nkm",
      "harpsichord.nkm",
      "concert_grand.nkm",
      "august_foerster_grand.nkm"
    ]
  }
}
```

All entries in `"defs": []` that end with `.nkm` will be loaded with Kontakt, and everything else
will be loaded as a regular VST. Loading presets with other VSTs is highly dependent on how those 
VSTs work. RenderMan does provide an interface for changing VST parameters programmatically, but
the interface is highly specific to each VST. For generating Slakh, we saved presets for Kontakt
(the `.nkm` files) and loaded those. For more information setting VST parameters see the RenderMan
functions `get_plugin_parameters_description()` and `set_parameter()` or reach out to me.


### Step 3b: Band Definition file and MIDI file lists

The way this script selects which MIDI files to render happens in two ways, by providing
a Band Definition file and/or a MIDI file list.

A Band definition file is a json file that determines if a MIDI file is okay to synthesize. If a 
candidate MIDI file has at least one of each instrument defined in this file then will be selected
to be synthesized. Here's an example from `band_defs/rock_band.json`:


```json
{ "key": "class",
  "band_def":
      [
        "Piano",
        "Guitar",
        "Bass",
        "Drums"
      ]
}
```

The first line `"key": "class"` is fixed (for now, possibly extensible in the future). The list
`"band_def` determines which MIDI instrument classes a MIDI file must have to be selected to
synthesize.


### Step 3c: `config.json`

The second json file is `config.json`. This handles all of the parameters that get sent to the main
script. Here is an annotated list of all of the variables:

`lmd_base_dir`: Absolute path to base directory of the Lakh MIDI Dataset. (`str`)  
`kontakt_path`: Absolute path to the `.component` for Kontakt. 
Usually `"/Library/Audio/Plug-Ins/Components/Kontakt.component"`. (`str`)   
`kontakt_defs_dir`: Absolute path to the default Kontakt patch location. 
Usually `"/Users/{{username}}/Library/Application Support/Native Instruments/Kontakt/default"`. (`str`)     
`user_defs_dir`: Path to directory containing user defined patches (
`.nkm` files created in step 2a, above). (`str`)  
`instrument_classes_file`: Path to json file that defines MIDI program number and classes. 
Default `general_midi_inst_0based.json`. (`str`)  
`defs_metadata_file`: Path to your Instrument Definition Metadata File as defined above. (`str`)  
`output_dir`: Absolute path to a base directory where audio will be output to. (`str`)  
`renderman_sr`: Sample rate that the audio will be synthesized at. (44.1kHz) (`int`)  
`renderman_buf`: Buffer size for the hosted VSTs. (`int`)  
`renderman_sleep`: Sleep time (in seconds) after a VST is loaded. Kontakt needs time to load 
samples into memory. (`float`)  
`renderman_restart_lim`: The RenderMan engine can be glitchy (see below). This number is the of 
stem files RenderMan will make before restarting. (`int`)  
`random_seed`: Random seed for selecting MIDI files from the LMD. (`int`)  
`max_num_files`: Total number of audio mixtures to generate. (`int`)  
`separate_drums`: If true and if a MIDI file has multiple tracks for drums this will render them 
all separately, else files like these get skipped. (`bool`)  
`mix_normalization_factor`: Value (in dB) that each track is normalized to according to 
ITU-R BS.1770-4. (`float`)  
`mix_target_peak`: Value (in dB) that the mix is normalized to once all the stems are summed 
together. (`float`)  
`render_pgm0_as_piano`: If true, MIDI program number 0 is interpreted as piano. Useful when using 
1-based MIDI. (`bool`)  
`rerender_existing`: If true, will overwrite existing audio files that have been synthesized. 
Else, will skip files if they've been seen before. Useful for restarting from a previously 
crashed session. (`str`)  
`band_definition_file`: 
`midi_file_list`: rock_band_files_min_overall.txt
`zero_based_midi`: If true, will 

## Step 4: Running data generation script


### Step 4a: Install python script requirements

Before you can run the script you need to install all of the required python libraries. As usual,
I recommend you make a new conda environment to run this. You can install of the requirements
by running `pip install -r requirements.txt` in the command line, where `requirements.txt` is
included in this repo.


### Step 4b: The data generation script

The data generation script is `render_by_instrument.py`. Call it like so:
 
```bash
python render_by_instrument.py --config path/to/config.json
``` 

All of the configurations for generation are defined in the json file, as formatted above.

**Note:** This hasn't been tested with python 3 yet, so check python 2.

 
At a high level, this script works in three stages. 

1. First it reads the MIDI files (either from the LMD or from a provided list of MIDI file paths)
and splits each multi-track MIDI file into separate MIDI files for each track, with each resultant
MIDI file containing only one instrument. At this stage, it randomly assigns a patch to each track
based on the mapping defined in your Instrument Definition Metadata File (as described above).

2. In the second stage, the script loads up a single instrument patch and generates audio for every 
MIDI track that has been assigned to that patch, regardless of which original MIDI file it came 
from. This is done such that only one VST is loaded at a time.

3. The final stage is the mixing stage. Each track has a gain applied such that they all have equal
loudness according to ITU-R BS.1770 (the loudness, in dB, is set in `config.json`). Then each track
is summed to make an instantaneous mixture. The peak of that mixture is calculated and if it is 
above `target_peak` (in dB, from `config.json`), the gain of the mixture and each track are lowered
to match `target_peak`.

Each of these three stages is a giant function, these functions talk via dicts that have the 
required info about 


## Flakh & using FluidSynth

Currently, I am not releasing the code to create Flakh, which is the accompanying dataset to Slakh 
that is the same MIDI files generated with FluidSynth. This is simple to do, but is a separate
process entirely. If you need to generate Flakh, please contact me and I will provide scripts.


## Gotchas

**1-based vs 0-based**  
The official MIDI specification is 
[1-based](https://www.midi.org/specifications-old/item/gm-level-1-sound-set)
but many *but not all* MIDI files in the LMD are 0-based. There is a switch in `config.json` to
swap between parsing 1-based and 0-based MIDI files. 0-based is a good setting for most use cases.

**Parallelization**  
I recommend **NOT** parallelizing this script as loading many patches at once is quite resource
intensive. **If you are planning on synthesizing many hours of audio give yourself time!** That 
being said, I did try a few ways to speed this up and the three stage process outlined above seems 
best if you do figure out a way to 
efficiently parallelize this type of data generation,
please let me know!

**Restarting the Engine**  
The RenderMan engine is finicky. It can sometimes fail silently in that it does not report any
issues, but it outputs waveforms of all `0`'s without notice. There are some checks for this in the
code, but there is also a hard restart built into the code. I.e., the script will restart a new
RenderMan engine after it synthesizes every *n* tracks (as defined by `renderman_restart_lim` 
above). I don't know if this is a problem with RenderMan or Kontakt/other VSTs. I doubt the Native
Instruments engineers planned for the type of massive processing of MIDI files that occurs in this
repo.


**Loading the Engine**


