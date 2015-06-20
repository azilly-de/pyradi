Staring Array Module (rystare)
****************************
.. include global.rst

Overview
--------

This module provides a high level model for CCD and CMOS staring array 
signal chain modelling.  The model accepts an input image in photon rate irradiance units 
and then proceeds to calculate the various noise components and 
signal components along the signal flow chain.

The code in this module serves as an example of implementation of a high-level 
CCD/CMOS photosensor signal chain model. The model is described in the article 
'High-level numerical simulations of noise in solid-state photosensors:  
review and tutorial' by Mikhail Konnik and James Welsh. 
The code was originally written in Matlab and used for the Adaptive Optics 
simulations and study of noise propagation in wavefront sensors, but can be 
used for many other applications involving light registration on CCD/CMOS
photosensors.  The original files are available at:

- Paper: http://arxiv.org/pdf/1412.4031.pdf
- Matlab code: https://bitbucket.org/aorta/highlevelsensorsim

The original Matlab code was ported to Python and extended
in a number of ways.  The core of the model remains the original Konnik model
as implemented in the Matlab code.  The  Python code was validated 
against results obtained with the Matlab code, up to a point 
and then substantially reworked and refactored.  During the refactoring
due diligence was applied with regression testing, checking the new
results against the previous results.

The documentation in the code was copied from Konnik's Matlab code, so 
he deserves all credit for the very detailed documentation.  His documentation 
was extracted from the paper quoted above.

The sample code in the repository models two different cases (from Konnik's code)

- a simple model: which is completely linear (no non-linearities), 
  where all noise are basically Gaussian, and without 
  source follower noise, 
- an advanced model: which has V/V and V/e non-linearities, 
  Wald or lognormal noise, source follower and sense node noise 
  sources and even ADC non-linearities.

The code supports enabling/disabling of key components by using flags.

In the documentation for the Matlab code Konnik expressed the hope "that this 
model will be useful for somebody, or at least save someone's time.
The model can be (and should be) criticized."  Indeed it has, thanks Mikhail!
Konnik quotes George E. P. Box, the famous statistician, and who said that 
"essentially, all models are wrong, but some are useful".


Signal Flow
------------

The process from incident photons to the digital numbers appearing in the 
image is outlined in the picture below. 
First the input image is provided in photon rate irradiance, 
with photon noise already present in the image.  The count of photons 
captured in the detector is determined from the irradiance by accounting 
for the detector area and integration time.
Then, the code models the process of conversion from photons to 
electrons and subsequently to signal voltage. Various noise sources 
are modelled to derive at a realistic image model.
Finally, the ADC converts the voltage signal into digital numbers. 
The whole process is depicted in the figure below.
 
.. image:: _images/camerascheme_horiz.png
    :width: 812px
    :align: center
    :height: 244px
    :alt: camerascheme_horiz.png
    :scale: 100 %

Many noise sources contribute to the resulting noise image that is produced by
the sensor. Noise sources can be broadly classified as either
*fixed-pattern (time-invariant)* or *temporal (time-variant)*
noise. Fixed-pattern noise refers to any spatial pattern that does not change
significantly from frame to frame. Temporal noise, on the other hand, changes
from one frame to the next.  All these noise sources are modelled in the code.
For more details see Konnik's original paper or the docstrings present in the code.

Changes to Matlab code
--------------------------

1. Renamed many, if not all, variables to be more descriptive.

2. Created a number of new functions by splitting up the Matlab functions for increased modularity.

3. Store (almost) all input and output variables in an HDF5 file for full record keeping.

4. Precalculate the image data input as HDF5 files with linear detector parameters embedded 
   in the file.  This was done to support future image size calculations.  The idea is to 
   embed the target frequency in the data file to relate observed performance with the 
   frequency on the focal plane.

5. Moved sourcefollower calcs out from under dark signal flag. sourcefollower noise is 
   now always calculated irrespective of whether dark noise is selected or not.

6. Input image now photon rate irradiance q/(m2.s), image should already include photon noise 
   in input.  Removed from ccd library: irradiance from radiant to photon units, adding 
   photon shot noise.  This functionality has been added to the image generation code.
 
7. Both CCD and CMOS now have fill factors, the user can set CCD fill factor differently 
   from CMOS fill factor.  The fill factor value is used as-in in the rest of the code, 
   without checking for CCD or CMOS.  This is done because CCD fill factor is 1.0 for 
   full frame sensors but can be less than 1.0 for other types of CCD.

8. Now uses SciPy's CODATA constants where these are available.

9. Put all of the code into a single file rystare.py in the pyradi repository.

10. Minor changes to Konnik's excellent documentation to be Sphinx compatible.  
    Documentation is now generated as part of the pyradi documentation.


Example Code
-------------

The two examples provided by Konnik are merged into a single code, with flags to 
select between the two options.  The code is found at the end of the module file
in the `__main__` part of the module file.  Set `doTest = 'Simple'` or `doTest = 'Advanced'`
depending on which model. 
Either example will run the `photosensor` function 
(all functions are thoroughly documented in the Python code, thanks Mikhail!).

The two prepared image files are both 256x256 in size.  New images can be generated
following the example shown  in the `__main__` part of the rystare.py module file 
(use the function `create_HDF5_image` as a starting point to develop your own).

The easiest way to run the code is to open a command window in the installation directory 
and run the `run_example` function in the module code.  This will load the module and 
execute the example code function. Running the example code function will create files 
with names similar to `PSOutput.hdf5` and `PSOutput.txt`.  To run the example, create
a python file with the following contents and run it at the command line prompt:

.. code-block:: python

    import pyradi.rystare as rystare
    rystare.run_example('Advanced','Output', doPlots=True, doHisto=True, doImages=True)
    rystare.run_example('Simple','Output', doPlots=True, doHisto=True, doImages=True)

By setting all the flags to True the example code will print a number of images to file.
Plotting the results to file takes a while.  Execution is much faster with all flags set to False.

Study the text file using a normal text editor and study the HDF5 file by using the viewer
available from https://www.hdfgroup.org/products/java/hdfview/.

Some time in future an IPython notebook will be released on 
https://github.com/NelisW/ComputationalRadiometry.

The full code for the example file is as follows:

.. code-block:: python

    #prepare so long for Python 3
    from __future__ import division
    from __future__ import print_function
    from __future__ import unicode_literals

    import numpy as np
    import re
    import os.path
    from matplotlib import cm as mcm
    import matplotlib.mlab as mlab

    import pyradi.rystare as rystare
    import pyradi.ryplot as ryplot
    import pyradi.ryfiles as ryfiles
    import pyradi.ryutils as ryutils

    """This file provides examples of use of the CcdCmosSim models for a CMOS/CCD photosensor.

    Two models are provided 'simple' and 'advanced'

    Author: Mikhail V. Konnik, revised/ported by CJ Willers

    Original source: http://arxiv.org/pdf/1412.4031.pdf
    """

    #set up the parameters for this run
    doPlots=True
    doHisto=True
    doImages=True
    outfilename = 'Output'
    pathtoimage = 'W:/MyApps/pyradi/pyradi/data/image-Disk-256-256.hdf5'

    doTest = 'Simple'
    doTest = 'Advanced'

    if doTest in ['Simple']:
        prefix = 'PS'
    elif  doTest in ['Advanced']:
        prefix = 'PA'
    else:
        exit('Undefined test')

    [m, cm, mm, mum, nm, rad, mrad] = rystare.define_metrics()

    #open the file to create data structure and store the results, remove if exists
    hdffilename = '{}{}.hdf5'.format(prefix, outfilename)
    if os.path.isfile(hdffilename):
        os.remove(hdffilename)
    strh5 = ryfiles.open_HDF(hdffilename)

    #sensor parameters
    strh5['rystare/SensorType'] = 'CCD' # must be in capitals
    #strh5['rystare/SensorType'] = 'CMOS' # must be in capitals

    # full-frame CCD sensors has 100% fil factor (Janesick: 'Scientific Charge-Coupled Devices')
    if strh5['rystare/SensorType'].value in ['CMOS']:
        strh5['rystare/FillFactor'] = 0.5 # Pixel Fill Factor for CMOS photo sensors.
    else:
        strh5['rystare/FillFactor'] = 1.0 # Pixel Fill Factor for full-frame CCD photo sensors.

    strh5['rystare/IntegrationTime'] = 0.01 # Exposure/Integration time, [sec].
    strh5['rystare/ExternalQuantumEff'] = 0.8  # external quantum efficiency, fraction not reflected.
    strh5['rystare/QuantumYield'] = 1. # number of electrons absorbed per one photon into material bulk
    strh5['rystare/FullWellElectrons'] = 2e4 # full well of the pixel (how many electrons can be stored in one pixel), [e]
    strh5['rystare/SenseResetVref'] = 3.1 # Reference voltage to reset the sense node. [V] typically 3-10 V.

    #sensor noise
    strh5['rystare/SenseNodeGain'] = 5e-6 # Sense node gain, A_SN [V/e]

    #source follower
    strh5['rystare/SourceFollowerGain'] = 1. # Source follower gain, [V/V], lower means amplify the noise.

    # Correlated Double Sampling (CDS)
    strh5['rystare/CDS-Gain'] = 1. # CDS gain, [V/V], lower means amplify the noise.

    # Analogue-to-Digital Converter (ADC)
    strh5['rystare/ADC-Num-bits'] = 12. # noise is more apparent on high Bits
    strh5['rystare/ADC-Offset'] = 0. # Offset of the ADC, in DN

    # Light Noise parameters
    strh5['rystare/flag/photonshotnoise'] = True #photon shot noise.
    # photo response non-uniformity noise (PRNU), or also called light Fixed Pattern Noise (light FPN)
    strh5['rystare/flag/PRNU'] = True
    strh5['rystare/noise/PRNU/seed'] = 362436069
    strh5['rystare/noise/PRNU/model'] = 'Janesick-Gaussian' 
    strh5['rystare/noise/PRNU/parameters'] = [] # see matlab filter or scipy lfilter functions for details
    strh5['rystare/noise/PRNU/factor'] = 0.01 # PRNU factor in percent [typically about 1\% for CCD and up to 5% for CMOS];

    # Dark Current Noise parameters
    strh5['rystare/flag/darkcurrent'] = True
    strh5['rystare/OperatingTemperature'] = 300. # operating temperature, [K]
    strh5['rystare/DarkFigureMerit'] = 1. # dark current figure of merit, [nA/cm2].  For very poor sensors, add DFM
    #  Increasing the DFM more than 10 results to (with the same exposure time of 10^-6):
    #  Hence the DFM increases the standard deviation and does not affect the mean value.
    strh5['rystare/DarkCurrentElecrons'] = 0.  #to be computed

    # dark current shot noise
    strh5['rystare/flag/DarkCurrent-DShot'] = True

    #dark current Fixed Pattern Noise 
    strh5['rystare/flag/DarkCurrentDarkFPN-Pixel'] = True
    # Janesick's book: dark current FPN quality factor is typically between 10\% and 40\% for CCD and CMOS sensors
    strh5['rystare/noise/darkFPN/DN'] = 0.3 
    strh5['rystare/noise/darkFPN/seed'] = 362436128
    strh5['rystare/noise/darkFPN/limitnegative'] = True # only used with 'Janesick-Gaussian' 

    if doTest in ['Simple']:
        strh5['rystare/noise/darkFPN/model'] = 'Janesick-Gaussian' 
        strh5['rystare/noise/darkFPN/parameters'] = [];   # see matlab filter or scipy lfilter functions for details
    elif  doTest in ['Advanced']:
        strh5['rystare/noise/darkFPN/model'] = 'LogNormal' #suitable for long exposures
        strh5['rystare/noise/darkFPN/parameters'] = [0., 0.4] #first is lognorm_mu; second is lognorm_sigma.
    else:
        pass

    # #alternative model
    # strh5['rystare/noise/darkFPN/model']  = 'Wald'
    # strh5['rystare/noise/darkFPN/parameters']  = 2. #small parameters (w<1) produces extremely narrow distribution, large parameters (w>10) produces distribution with large tail.

    # #alternative model
    # strh5['rystare/noise/darkFPN/model']  = 'AR-ElGamal'
    # strh5['rystare/noise/darkFPN/parameters']  = [1., 0.5] # see matlab filter or scipy lfilter functions for details

    #dark current Offset Fixed Pattern Noise 
    strh5['rystare/flag/darkcurrent_offsetFPN'] = True
    strh5['rystare/noise/darkFPN_offset/model'] = 'Janesick-Gaussian'
    strh5['rystare/noise/darkFPN_offset/parameters'] = [] # see matlab filter or scipy lfilter functions for details
    strh5['rystare/noise/darkFPN_offset/DNcolumn'] = 0.0005 # percentage of (V_REF - V_SN)

    # Source Follower VV non-linearity
    strh5['rystare/flag/VVnonlinearity'] = False

    #ADC
    strh5['rystare/flag/ADCnonlinearity'] = 0 
    strh5['rystare/ADC-Gain'] = 0.

    if doTest in ['Simple']:
        strh5['rystare/flag/sourcefollowernoise'] = False
    elif  doTest in ['Advanced']:
    #source follower noise.
        strh5['rystare/flag/sourcefollowernoise'] = True
        strh5['rystare/noise/sf/CDS-SampleToSamplingTime'] = 1e-6 #CDS sample-to-sampling time [sec].
        strh5['rystare/noise/sf/flickerCornerHz'] = 1e6 #flicker noise corner frequency $f_c$ in [Hz], where power spectrum of white and flicker noise are equal [Hz].
        strh5['rystare/noise/sf/dataClockSpeed'] = 20e6 #MHz data rate clocking speed.
        strh5['rystare/noise/sf/WhiteNoiseDensity'] = 15e-9 #thermal white noise [\f$V/Hz^{1/2}\f$, typically \f$15 nV/Hz^{1/2}\f$ ]
        strh5['rystare/noise/sf/DeltaIModulation'] = 1e-8 #[A] source follower current modulation induced by RTS [CMOS ONLY]
        strh5['rystare/noise/sf/FreqSamplingDelta'] = 10000. #sampling spacing for the frequencies (e.g., sample every 10kHz);
    else:
        pass

    #charge to voltage
    strh5['rystare/flag/Venonlinearity'] = False

    #sense node reset noise.
    strh5['rystare/sn/V-FW'] = 0.
    strh5['rystare/sn/V-min'] = 0.
    strh5['rystare/sn/C-SN'] = 0.
    strh5['rystare/flag/SenseNodeResetNoise'] = True
    strh5['rystare/noise/sn/ResetKTC-Sigma'] = 0.
    strh5['rystare/noise/sn/ResetFactor'] = 0.8 # the compensation factor of the Sense Node Reset Noise: 
                                           # 1 - no compensation from CDS for Sense node reset noise.
                                           # 0 - fully compensated SN reset noise by CDS.

    #Sensor noises and signal visualisation
    strh5['rystare/flag/plots/doPlots'] = False
    strh5['rystare/flag/plots/plotLogs'] = False
    # strh5['rystare/flag/plots/irradiance'] = True
    # strh5['rystare/flag/plots/electrons'] = True
    # strh5['rystare/flag/plots/volts'] = True
    # strh5['rystare/flag/plots/DN'] = True
    # strh5['rystare/flag/plots/SignalLight'] = True
    # strh5['rystare/flag/plots/SignalDark'] = True

    #For testing and measurements only:
    strh5['rystare/flag/darkframe'] = False # True if no signal, only dark

    #=============================================================================

    if strh5['rystare/flag/darkframe'].value:  # we have zero light illumination    
        hdffilename = 'data/image-Zero-256-256.hdf5'
    else:   # load an image, nonzero illumination
        hdffilename = 'data/image-Disk-256-256.hdf5'

    if pathtoimage is None:
        pathtoimage = os.path.dirname(__file__) + '/' + hdffilename

    imghd5 = ryfiles.open_HDF(pathtoimage)

    #images must be in photon rate irradiance units q/(m2.s)
    strh5['rystare/SignalPhotonRateIrradiance'] = imghd5['image/PhotonRateIrradiance'].value

    strh5['rystare/pixelPitch'] = imghd5['image/pixelPitch'].value
    strh5['rystare/imageName'] = imghd5['image/imageName'].value
    strh5['rystare/imageSizePixels'] = imghd5['image/imageSizePixels'].value

    #calculate the noise and final images
    strh5 = rystare.photosensor(strh5) # here the Photon-to-electron conversion occurred.

    with open('{}{}.txt'.format(prefix,outfilename), 'wt') as fo: 
        fo.write('{:25}, {:.5e}, {:.5e}\n'.format('SignalPhotonRateIrradiance',np.mean(strh5['rystare/SignalPhotonRateIrradiance'].value), np.var(strh5['rystare/SignalPhotonRateIrradiance'].value)))
        fo.write('{:25}, {:.5e}, {:.5e}\n'.format('signalLight',np.mean(strh5['rystare/signalLight'].value), np.var(strh5['rystare/signalLight'].value)))
        fo.write('{:25}, {:.5e}, {:.5e}\n'.format('signalDark',np.mean(strh5['rystare/signalDark'].value), np.var(strh5['rystare/signalDark'].value)))
        fo.write('{:25}, {:.5e}, {:.5e}\n'.format('source_follower_noise',np.mean(strh5['rystare/noise/sf/source_follower_noise'].value), np.var(strh5['rystare/noise/sf/source_follower_noise'].value)))
        fo.write('{:25}, {:.5e}, {:.5e}\n'.format('SignalPhotons',np.mean(strh5['rystare/SignalPhotons'].value), np.var(strh5['rystare/SignalPhotons'].value)))
        fo.write('{:25}, {:.5e}, {:.5e}\n'.format('SignalElectrons',np.mean(strh5['rystare/SignalElectrons'].value), np.var(strh5['rystare/SignalElectrons'].value)))
        fo.write('{:25}, {:.5e}, {:.5e}\n'.format('SignalVoltage',np.mean(strh5['rystare/SignalVoltage'].value), np.var(strh5['rystare/SignalVoltage'].value)))
        fo.write('{:25}, {:.5e}, {:.5e}\n'.format('SignalDN',np.mean(strh5['rystare/SignalDN'].value), np.var(strh5['rystare/SignalDN'].value)))

    if doPlots:
        lstimgs = ['rystare/SignalPhotonRateIrradiance','rystare/SignalPhotons','rystare/SignalElectrons','rystare/SignalVoltage',
                   'rystare/SignalDN','rystare/signalLight','rystare/signalDark', 'rystare/noise/PRNU/nonuniformity',
                   'rystare/noise/darkFPN/nonuniformity']
        # ryfiles.plotHDF5Images(strh5, prefix=prefix, colormap=mcm.gray,  lstimgs=lstimgs, logscale=strh5['rystare/flag/plots/plotLogs'].value) 
        ryfiles.plotHDF5Images(strh5, prefix=prefix, colormap=mcm.jet,  lstimgs=lstimgs, logscale=strh5['rystare/flag/plots/plotLogs'].value) 

    if doHisto:
        lstimgs = ['rystare/SignalPhotonRateIrradiance','rystare/SignalPhotons','rystare/SignalElectrons','rystare/SignalVoltage',
                   'rystare/SignalDN','rystare/signalLight','rystare/signalDark',
                   'rystare/noise/PRNU/nonuniformity','rystare/noise/darkFPN/nonuniformity']
        ryfiles.plotHDF5Histograms(strh5, prefix, bins=100, lstimgs=lstimgs)

    if doImages:
        lstimgs = ['rystare/SignalPhotonRateIrradiance','rystare/SignalPhotonRate', 'rystare/SignalPhotons','rystare/SignalElectrons','rystare/SignalVoltage',
                    'rystare/SignalDN','rystare/signalLight','rystare/signalDark', 'rystare/noise/sn_reset/noisematrix','rystare/noise/sf/source_follower_noise',
                    'rystare/noise/PRNU/nonuniformity','rystare/noise/darkFPN/nonuniformity']
        ryfiles.plotHDF5Bitmaps(strh5, prefix, format='png', lstimgs=lstimgs)

    strh5.flush()
    strh5.close()





HDF5 File
---------

The Python implementation of the model uses an HDF5 file to capture the
input and output data for record keeping or subsequent analysis. 
HDF5 files provide for hierarchical data structures and easy read/save to disk. 
See the file `hdf5-as-data-format.md` for more detail.

Input images are written to and read from HDF5 files as well.  These files store the
image as well as the images' dimensional scaling in the focal plane.  
The intent is to later create test targets with specific spatial 
frequencies in these files.


Code Overview
---------------
.. automodule:: pyradi.rystare


Module functions
------------------

.. autofunction:: pyradi.rystare.photosensor	

.. autofunction:: pyradi.rystare.source_follower	

.. autofunction:: pyradi.rystare.cds

.. autofunction:: pyradi.rystare.adc

.. autofunction:: pyradi.rystare.sense_node_chargetovoltage

.. autofunction:: pyradi.rystare.sense_node_reset_noise

.. autofunction:: pyradi.rystare.dark_current_and_dark_noises

.. autofunction:: pyradi.rystare.source_follower_noise

.. autofunction:: pyradi.rystare.set_photosensor_constants

.. autofunction:: pyradi.rystare.create_data_arrays

.. autofunction:: pyradi.rystare.image_irradiance_to_flux

.. autofunction:: pyradi.rystare.convert_to_electrons

.. autofunction:: pyradi.rystare.shotnoise

.. autofunction:: pyradi.rystare.responsivity_FPN_light

.. autofunction:: pyradi.rystare.responsivity_FPN_dark

.. autofunction:: pyradi.rystare.FPN_models

.. autofunction:: pyradi.rystare.create_HDF5_image

.. autofunction:: pyradi.rystare.define_metrics

.. autofunction:: pyradi.rystare.limitzero

.. autofunction:: pyradi.rystare.distribution_exp

.. autofunction:: pyradi.rystare.distribution_lognormal

.. autofunction:: pyradi.rystare.distribution_inversegauss

.. autofunction:: pyradi.rystare.distribution_logistic

.. autofunction:: pyradi.rystare.distribution_wald

.. autofunction:: pyradi.rystare.distributions_generator

.. autofunction:: pyradi.rystare.validateParam

.. autofunction:: pyradi.rystare.checkParamsNum

.. autofunction:: pyradi.rystare.run_example


