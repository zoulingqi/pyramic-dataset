Pyramic Dataset : Multichannel Anechoic Audio Recordings
========================================================

The Pyramic Dataset contains recordings done using the
[Pyramic](https://github.com/LCAV/Pyramic) 48 channels microphone array in an
anechoic chambers. The recordings consist of 8 different samples (2x sweeps, 1x
noise, 5x speech) repeated at 180 angles (every 2 degrees) and from 3 different
heights. The audio samples recorded are

* Linear and exponential sweeps
* Noise sequence
* 2x male and 3x female speech

This dataset allows to evaluate the performance of array processing algorithms
on real-life recordings done using MEMS microphones similar to those used in
mobile phones with all the non-idealities involved.  By subsampling the 48
microphones, a large number of array configurations can be tested.  Example of
algorithms are:

* Direction of arrival (DOA) estimation
* Beamforming
* Source separation
* Array calibration

Another application is the generation of realistic room impulse by combining
the impulse responses of microphones from sources at multiple angles with a
variant of the image source model.

In addition to the raw ([compressed](compressed) or not) and segmented
recordings, the impule responses of all the microphones for every source
locations were recovered from the exponential sweep measurements and are
distributed along the dataset. The initial manual measurement of loudspeakers
and microphones locations was improved upon using a blind calibration method as
described [later](calibration).

### Dataset

The dataset and code is available on Zenodo split in two parts due to its size:

* Raw only: 
* Compressed and processed: 10.5281/zenodo.1209563

A zipped version of this documentation and code is stored along the data
on Zenodo but is also available on github for convenience.

### Download the dataset

The dataset is split in four different archives

* The raw recordings in wav format (38GB)
* The raw recordings compressed to [tta](compression) format (18GB)
* The segmented recorded samples in wav format (22GB) (**<- probably what you need**)
* The impulse responses (280MB)

The following will get you started
    
    git clone https://github.com/fakufaku/pyramic-dataset
    cd pyramic-dataset

    # get the segmented measurements (probably what you want)
    wget -qO- <url> | tar xzv

    # OR Get the raw measurements
    wget -qO- https://zenodo.org/record/1209005/files/pyramic_raw_recordings.tar.gz | tar xzv

    # OR get the compressed raw measurements
    wget -qO- <url> | tar xv

    # OR get the impulse responses
    wget -qO- <url> | tar xzv

#### Checksum the Raw Data

The `sha256` checksums of the wav files is available in `checksums.txt`. The checksums
were obtained by running

    sha256sum recordings/ > checksums.txt

and can be verified against the version downloaded (assuming the recordings wav
files are in `recordings` folder)

    cd recordings
    sha256sum -c checksums.txt

#### File Naming

The raw recording files are named according to the following pattern:

    recordings/pyramic_spkrX_all_samples_Y.[wav|tta]

where `X` is the loudspeaker index and `Y` is the angle (in degrees) by which **the array was
rotated in the trigonometric direction** (i.e. _counterclockwise_).  

The segmented files are saved in the `<output_dir>` folder following the naming convention

    segemented/<sample_name>/<sample_name>_spkrX_angleZ.wav
    
where this time `Z` is the angle (in degrees) **the loudspeaker was rotated in the
trigonometric direction** with respect to the array. The name `<sample_name>`
is one of the following:

* `silence` : a segment of silence, this can be used as a reference for the noise in the recordings
* `sweep_lin` : linear sweep
* `sweep_exp` : exponential sweep
* `noise` : the noise sequence
* `fq_sample0` : female speech
* `fq_sample1` : female speech
* `fq_sample2` : male speech
* `fq_sample3` : female speech
* `fq_sample4` : male speech

__Note__:
Since the array rotates counterclockwise, the rotation of the loudspeaker
relative to the array is _clockwise_ (or negative trigonometric direction).

### Post-Processing

### Code Dependencies

Python 3.5 with the following packages installed.

    numpy, scipy, ipyparalle, samplerate, pyroomacoustics

We recommend using Anaconda as it is the simplest way to quickly get numpy and
scipy running. The rest can be installed via pip.

No effort was made to make this code Python 2.7 compatible, but it is not
impossible that large parts of it are nonetheless.

#### Compression

To reduce the file size efficiently, they were compressed using the [True
Audio](https://en.wikipedia.org/wiki/TTA_(codec)) (TTA) lossless compression format.
This format is supported for compression and decompression by
[FFMPEG](https://ffmpeg.org).  Decompression of the files can be done using
older versions of ffmpeg (e.g. 2.8), but compression (normally not needed for
using this data) requires a newer version like 3.3.

The files were compressed by running

    ./code/compress.sh recordings compressed_recordings

and then wrapped in a tar archive by

    tar cfv compressed_recordings.tar compressed_recordings/

The files can be uncompressed to wav directly from the tar
archive by running

    python ./code/uncompress.py /path/to/compressed_archive.tar /path/to/output/dir

This command will read the TTA compressed file from the tar archive, uncompress
to wav and save them to the output directory.

Alternatively, the TTA files can be extracted from the tar archive (`tar xfv
compressed_recordings.tar`) and read directly from python (via ffmpeg)

    import sys
    sys.path.append('./code/')
    import ffmpeg_audio

    r,data = ffmpeg_audio.read('./compressed_recordings/pyramic_spkr0_all_samples_0.wav')

#### Segmentation

Since all the sound samples were recorded together in the same file, a
segmentation step was necessary to recover the recordings of individual
samples. This was done automatically using the script `code/segment.py`. The
python package `ipyparallel` was used to accelerate the process.  Run the
following.

    python code/segment.py -a -q -o <output_dir> recordings

The result of the segmentation was manually verified for all recordings by visual
inspection of the spectrogram image produced by the `-q` option above.

#### Calibration

The locations of the microphones on the array are fairly rigid and were
manually measured in advance of the experiment. The locations of sources with
respect to the array were also manually measured at the time of the experiment
and recorded in `protocol.json`.  Nevertheless, using the recorded signals, it
is possible to infer relative time-of-flights between the microphones and use
this data to run a blind calibration algorithm (possibly aided by the manual
measurements). Because the sources are in the far field with respect to the
array, we chose the method from Sebastian Thrun [[2]](references). The file
`code/calibration.py`

We start by computing the time difference of arrivals with reference to
microphone 0 using GCC-PHAT [[1]](references) on the noise sequence samples.
This results in a matrix with as many rows as the number of microphones minux
one (47), and as many columns as there are sources (3 loundspeakers x 180
angles = 540). We can convert the TDOA to distances by multiplying by the speed
of sound (adjusted for the temperature and humidity). When inspecting the
distances obtained, we noticed a few outliers where the relative distance
obtained is larger than 3x the size of the array itself.  There were five of
these measurements so rather than fixing them manually, we replaced them with
the value we obtain from the manually calibrated locations. Since only 5 /
25380 measurements are missing, we hope that the blind calibration will take
care of them. Finally, we run Thrun's method on the cleaned relative distance
matrix to obtain the calibrated microphone and sources locations. 

Because this algorithm doesn't preserve the global rotation, we use a
[Procrustes](https://en.wikipedia.org/wiki/Orthogonal_Procrustes_problem)
transformation to rotate all the locations so that the distance between the
manually and automatically calibrated microphones locations is minimized.  The
calibrated locations are stored in `calibration/calibrated_locations.json` with
the following format.

* `speakers_numbering` : contains the mapping between `middle`/`low`/`high` to the index used in the file names
* `microphones` : an array of triplets containing the microphones locations
* `sources` : the location of sources in polar coordinates
  * `high`/`middle`/`low` : the loudspeaker
    * `gain`/`azimuth`/`colatitude` : here `gain` adjustement found by the blind calibration algorithm, this can probably be generally ignored as it is usually between 0.99 and 1.01
      * `"A" : B` : pairs where `"A"` is the degree value of the azimuth rotation of the loudspeaker with respect to the array that was set for the experiment, and `B` is the actual value of `gain`/`azimuth`/`colatitude` found by the blind calibration

The calibration procedure is automated and can be run like this.

    # compute all the TDOA (fairly long)
    python ./code/compute_tdoa.py -o calibration/pyramic_tdoa.json -r 0 -p protocol.json
    # calibrate the locations
    python code/run_calibration.py -f calibration/pyramic_tdoa.json -m svd -s calibration/calibrated_locations.json -p

#### Impulse Responses

The response of every microphone to every angle was obtained from the exponential
sweep by [Wiener deconvolution](https://en.wikipedia.org/wiki/Wiener_deconvolution).
The `silence` recordings are used to get the power spectral density of the microphone
self-noise used in the Wiener deconvolution. Then the length of the RIR was empirically
determined to be less than 1500 samples (at 48 kHz) and truncated at that length.

All the recordings were processed by running the following.

    python code/run_deconvolution.py samples/sweep_exp.wav segmented/ pyramic_ir -l 1500 -q

Here, the `-q` option produced quality control plots of the impulse responses
that were used to visually inspect them for unexpected defects or problems.

### Experimental Protocol

The experimental protocol followed for the collection of this dataset is described
in details in `Protocol.md`. A machine readable (JSON) version of the document that
contains all the most important quantities needed to process the data is also provided.

### References

[1] C. Knapp and G. C. Carter, __The generalized correlation method for estimation of time delay__, IEEE Trans. Acoust., Speech, Signal Process., vol. 24, no. 4, pp. 320–327, 1976.

[2] Sebastian Thrun, __[Affine Structure from Sound](https://papers.nips.cc/paper/2770-affine-structure-from-sound.pdf)__, NIPS, 2007

