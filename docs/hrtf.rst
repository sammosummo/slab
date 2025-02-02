.. _Hrtfs:

HRTFs
=====

A head-related transfer function (HRTF) describes the impact of the listeners ears, head and torso on incoming sound
for every position in space. Knowing the listeners HRTF, you can simulate a sound source at any position by filtering
it with the transfer function corresponding to that position. The :class:`HRTF` class provides methods for
manipulating, plotting, and applying head-related transfer functions.

Reading HRTF data
-----------------
Typically the :class:`HRTF` class is instantiated by loading a file. The canonical format for HRTF-data is called
sofa (Spatially Oriented Format for Acoustics). To read sofa files, you need to install the h5netcdf module:
`pip install h5netcdf`. The slab includes a set of standard HRTF recordings from KEMAR (a mannequin for acoustic
recordings) in the folder indicated by the :func:`data_path` function. For convenience, you can load this dataset
by calling :func:`slab.HRTF.kemar()`. Print the resulting object to obtain information about the structure of the
HRTF data ::

    hrtf = slab.HRTF.kemar()
    print(hrtf)
    # <class 'hrtf.HRTF'> sources 710, elevations 14, samples 710, samplerate 44100.0

Libraries of many other recordings can be found on the `website of the sofa file format <https://www.sofaconventions.org/>`_.
You can read any sofa file downloaded from these repositories by calling the :class:`HRTF` class with the name of
the sofa file as an argument.

.. note:: The class is at the moment geared towards plotting and analysis of HRTF files in the `sofa format <https://www.sofaconventions.org/>`_, because we needed that functionality for grant applications. The functionality will grow as we start to record and manipulate HRTFs more often.

.. note:: When we started to writing this code, there was no python module for reading and writing sofa files.
Now that `pysofaconventions <https://github.com/andresperezlopez/pysofaconventions>`_ is available, we will at some
point switch internally to using that module as backend for reading sofa files, instead of our own limited implementation.

Plotting sources
--------------------
The HRTF is a set of many transfer functions, each belonging to a certain sound source position (for example,
there are 710 sources in the KEMAR recordings). You can plot the source positions in 3D with the :meth:`.plot_sources`
to get an impression of the density of the recordings. The red dot indicates the position of the listener and the red
arrow indicates the lister's gaze direction. Optionally, you can supply a list of source indices which will be
highlighted in red. This can be useful when you are selecting source locations for an experiment and want to confirm
that you chose correctly. In the example below, we select sources using the methods :meth:`.elevation_sources`, which
selects sources along a horizontal sphere slice at a given elevation and :meth:`.cone_sources`, which selects sources
along a vertical slice through the source sphere in front of the listener a given angular distance away from the
midline:

.. plot::
    :include-source:
    :context:

    # cone_sources and elevation_sources return lists of indices which are concatenated by adding:
    hrtf = slab.HRTF.kemar()
    sourceidx = hrtf.cone_sources(0) + hrtf.elevation_sources(0)
    hrtf.plot_sources(sourceidx) # plot the sources in 3D, highlighting the selected sources

Try a few angles for the :meth:`.elevation_sources` and :meth:`.cone_sources` methods to understand how selecting
the sources works!

Plotting transfer functions
---------------------------
As mentioned before, a HRTF is collection of transfer functions. Each single transfer function is an instance of the
:class:`slab.Filter` with two channels - one for each ear. The transfer functions are located in the :attr:`data`
list and the coordinates of the corresponding sources can be found in the :attr:`sources.cartesian` list of the hrtf
object. In the example below, we select a source, print it's coordinates and plot the corresponding transfer function.

.. plot::
    :include-source:
    :context: close-figs

    from matplotlib import pyplot as plt
    hrtf = slab.HRTF.kemar()
    filt = hrtf.data[10] # choose a filter
    source = hrtf.sources.vertical_polar[10]
    print(f'azimuth: {round(source[0])}, elevation: {source[1]}')
    filt.tf()

The :class:`HRTF` class also has a :meth:`.plot_tf` method to plot transfer functions as either `waterfall`
(as is Wightman and Kistler, 1989), `image` plot (as in Hofman 1998). The function takes a list of source indices as an
argument which will be included in the plot. The function below shows how to generate a `waterfall` and `image` plot
for the sources along the central cone. Before plotting, we apply a diffuse field equalization to remove non-spatial
components of the HRTF, which makes the features of the HRTF that change with direction easier to see:

.. plot::
    :include-source:
    :context: close-figs

    hrtf = slab.HRTF.kemar()
    fig, ax = plt.subplots(2)
    dtf = hrtf.diffuse_field_equalization()
    sourceidx = hrtf.cone_sources(0)
    ax[0].set_title("waterfall plot")
    ax[1].set_title("image plot")
    hrtf.plot_tf(sourceidx, ear='left', axis=ax[0], show=False, kind="waterfall")
    hrtf.plot_tf(sourceidx, ear='left', axis=ax[1], show=False, kind="image")
    plt.tight_layout()
    plt.show()


As you can see the HRTF changes systematically with the elevation of the sound source, especially for frequencies above
6 kHz. Individual HRTFs vary in the amount of spectral change across elevations, mostly due to differences in the
shape of the ears. You can compute a measure of the HRTFs spectral dissimilarity the vertical axis, called vertical
spatial information (VSI, `Trapeau and Schönwiesner, 2016 <https://pubmed.ncbi.nlm.nih.gov/27586720/>`_).
The VSI relates to behavioral localization accuracy in the vertical dimension: listeners with acoustically more
informative spectral cues tend to localize sounds more accurately in the vertical axis. Identical filters give a VSI
of zero, highly dissimilar filters give a VSI closer to one. The hrtf has to be diffuse-field equalized for this
measure to be sensible, and the :meth:`.vsi` method will apply the equalization. The KEMAR mannequin have a VSI
of about 0.73::

    hrtf.vsi()
    # .73328

The :meth:`.vsi` method accepts arbitrary lists of source indices for the dissimilarity computation.
We can for instance check how the VSI changes when sources further off the midline are used. There are some reports
in the literature that listeners can perceive the elevation of a sound source better if it is a few degrees to the
side. We can check whether this is due to more dissimilar filters at different angles (we'll reuse the `dtf` from above
to avoid recalculation of the diffuse-field equalization in each iteration)::

    for cone in range(0,51,10):
        sources = dtf.cone_sources(cone)
        vsi = dtf.vsi(sources=sources, equalize=False)
        print(f'{cone}˚: {vsi:.2f}')
        # 0˚: 0.73
        # 10˚: 0.63
        # 20˚: 0.69
        # 30˚: 0.74
        # 40˚: 0.76
        # 50˚: 0.73

The effect seems to be weak for KEMAR, (VSI falls off for directions slightly off the midline and then increases again
at around 30-40˚).


Applying an HRTF to a monaural sound
------------------------------------
The HRTF describes the directional filtering of incoming sounds by the listeners ears, head and torso. Since this is the
basis for localizing sounds in three dimensions, we can apply the HRTF to a sound to evoke the impression of it coming
from a certain direction in space when listening through headphones. The :meth:`HRTF.apply` method returns an instance of
the :class:`slab.Binaural`. This method differs from :meth:`~Filter.apply` and conserves ITDs. In the example below we
apply the transfer functions corresponding to three sound sources at different elevations along the vertical midline to white noise.

.. plot::
    :include-source:
    :context: close-figs

    hrtf = slab.HRTF.kemar()
    sound = slab.Sound.pinknoise(samplerate=hrtf.samplerate)  # the sound to be spatialized
    fig, ax = plt.subplots(3)
    sourceidx = [0, 260, 536]  # sources at elevations -40, 0 and 40
    spatial_sounds = []
    for i, index in enumerate(sourceidx):
        spatial_sounds.append(hrtf.apply(index, sound))
        spatial_sounds[i].spectrum(axis=ax[i], low_cutoff=5000, show=False)
    plt.show()

You can use the :meth:`~Sound.play` method of the sounds to listen to them - see if you can identify the virtual sound
source position. Your ability to do so depends on how similar your own HRTF is to that of the the KEMAR artificial head.
Your auditory system can get used to new HRTFs, so if you listen to the KEMAR recordings long enough they will eventually
produce virtual sound sources at the correct locations.

Binaural filters from the KEMAR HRTF will impose the correct spectral profile, but no ITD. After applying an HRTF filter
corresponding to an off-center direction, you should also apply an ITD corresponding to the direction using the
:meth:`Binaural.azimuth_to_itd` and :meth:`Binaural.itd` methods.

Finally, the HRTF filters are recorded only at certain locations (710, in case of KEMAR - plot the source locations to
inspect them). You can interpolate a filter for any location covered by these sources with the :meth:`HRTF.interpolate`
method. It triangulates the source locations and finds three sources that form a triangle around the requested location
and interpolate a filter with a (barycentric) weighted average in the spectral domain. The resulting filter may not have
the same overall gain, so remember to set the level of your stimulus after having applied the interpolated HRTF.
