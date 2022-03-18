K-Band Tutorial
***************

The documentation below describes the steps to analyze images taken with a k-band filter. To analyze science images taken with the j-band filter or the helium filter, please navigate to the :doc:`jband-tutorial` page or the :doc:`helium-tutorial` page respectively.

To begin utilizing all functions within the library, create five flags corresponding to the five core functions of the library:

.. code-block:: Python

  remake_darks_and_flats = True
  remake_bkg = True
  calibrate_data = True
  photometric_extraction = True
  fit_for_eclipse = True

To run a step, the boolean flag in the python script must be set to True. To turn off any step that you do not wish to run, set the boolean flag to False.

.. code-block:: Python

  remake_darks_and_flats = False

Specify the full paths to the raw image data as well as the output directory. Also, give a prefix for the output test files and specify a naming style. For example:

.. code-block:: Python

  data_dir = "absolute path to the data directory/"
  output_dir = "absolute path to the output directory/"
  test_name = 'test_name'
  naming_style = 'image'

Note that the output directory must be manually created before running the code.

Indicate the starting and ending indices of the science images and dark sequences to be analyzed:

.. code-block:: Python

  science_seqs = [(291, 529)] 
  dark_seqs = [(530, 540)]

If multiple discontinuous series of science sequences or dark sequences are to be used, then include the seperate sequences as tuple pairs of the starting and ending index. For example:

.. code-block:: Python

  science_seqs = [(280, 477), (510, 693)] 

Similarly, include the starting and ending indices for the flat image sequences as well as for the dark for flat sequences:

.. code-block:: Python

  flat_seq = (66, 86)
  dark_for_flat_seq = (1, 21)

Specifically for images with background mode set to global, include the background sequnece:

.. code-block:: Python

  bkg_seq = (285, 289)

Set the lower and upper sigma thresholds for clipping data from the images. A higher upper sigma threshold ensures that the highest outliers are still clipped.

.. code-block:: Python
  bkg_sigma_lower = 5
  bkg_sigma_upper = 1000
  background_mode = 'global'

Covariates are quantities to be used for systematic noise correction. Add the covariates whose data you would like to be included in the final fitting procedure in the covariate_names list. For example:

.. code-block:: Python

  covariate_names = ['d_from_med', 'airmass', 'background']

A full list of covariates that may be selected include:

  |   'd_from_med',
  |   ‘airmass',
  |   'background',
  |   'psf_width',
  |   'x_cent',
  |   'y_cent',
  |   'd_from_med’,
  |   ‘water_proxy’
  
Provide the estimated pixel coordinate of the target source in the science image:

.. code-block:: Python

  source_coords = [359, 449]

A cluster of pixels may be identified as a star if its point spread function (PSF) has a full-width-half-max above a threshold value. Optionally set an estiamte of this value in the variable finding_fwhm. If finding_fwhm is not set, the value is defaulted to 15.

.. code-block:: Python

  finding_fwhm = 20.

Provide a list of aperature radii sizes to test for photometric extraction. If a list for extraction_rads is not provided, the value of the raddi list is defaulted to [20.].

.. code-block:: Python

  extraction_rads = range(25, 50)

A tuple of the inner and outer pixel radii of the annulus ring that surrounds the target star may  optionally be specified for performing the local background subtraction. If there is no specification of ann_rads, then the default radii values of the tuple is (20, 50).

.. code-block:: Python

  ann_rads = (25, 50)

Estimate a sigma threshhold for detecting stars in the images (this is the sigma threshold above the background for identifying the bright pixels that correspond to stars). The default source_detection_sigma value is 50.

.. code-block:: Python

  source_detection_sigma = 600.

The source_detection_sigma value may be readjusted after running the photometric analysis. To determine whether to lower or to raise the source_detection_sigma value, navigate to the output dump directory and search for image file source_plot.png generated from the photometry step. If you find that the source star is not circled (not detected) because it is too faint, the threshold should be lowered.

Set a maximum number of comparison stars to use in the photometry process. If the max_num_compars is not specified, it is defaulted to 10. However, note that the usable number is often smaller than 10 in sparse fields.

.. code-block:: Python

  max_num_compars = 12

Define parameters for the fitting of the expected eclipse shape. In this case, and typically for K-band observations, we are observing the secondary eclipse of an exoplanet. Therefore we set the phase to 'secondary', and include a prior on the planet to star flux ratio, fpfs_prior. The other fitting parameters are the same ones used for fitting transit profiles.

.. code-block:: Python

  phase = 'secondary'
  texp = 15*0.92/60/1440.
  r_star_prior = ('normal', 1.7, 0.07)
  period_prior = ('normal', 0.6724613, 0.0000019)
  t0_prior = ('normal', 2458997.16653, 0.00016)
  a_rs_prior = ('normal', 3.148, 0.034)
  b_prior = ('normal', 0.137, 0.029)
  ldc_val = [0., 0.]
  fpfs_prior = ('uniform', 0., 0.05)
  jitter_prior = ('uniform', 1e-6, 1e-2)

Define the parameters for how many steps to run the exoplanet PyMC3 posterior sampler:

.. code-block:: Python

  tune = 1000
  draws = 1500
  target_accept = 0.99 

Now begins the code segment of the sample k-band script:

.. code-block:: Python

  if __name__ == '__main__':

First, initialize the output directories for storing the output of the calibrations and analyses:

.. code-block:: Python

  calib_dir, dump_dir, img_dir = 
    iu.init_output_direcs(output_dir, test_name)

The calib_dir stores the calibrated image data that are later used for photometric analysis. The dump_dir stores the diagnostic information about the images that were generated by running the functions, which will later be used in the photometric analysis and fitting, along with the results of the fit. The img_dir stores the scientific analysis plots.

Construct a background image by using the make_calibrated_bkg_image function with all the following parameters:

.. code-block:: Python

  with warnings.catch_warnings():
      warnings.simplefilter("ignore")
      bkg = cu.make_calibrated_bkg_image(
        data_dir, 
        calib_dir,	
        bkg_seq,
        dark_seqs, 
        dark_for_flat_seq, 
        flat_seq,
        naming_style = naming_style,
        nonlinearity_fname = nonlinearity_fname,
        sigma_lower = bkg_sigma_lower, 
        sigma_upper = bkg_sigma_upper, 
        remake_darks_and_flats = remake_darks_and_flats,
        remake_bkg = remake_bkg)

Note that scripts set to 'median' background mode do not need the above calibration step.

Calibrate the science images if the calibrate_data flag is turned on by passing in the science sequence images, the dark images, the flat images, and the dark for flat images into the calibrate_all() function along with the three directories and other optional parameters:

.. code-block:: Python

    if calibrate_data:
      with warnings.catch_warnings():
        warnings.simplefilter("ignore")
        cu.calibrate_all(
          data_dir, 
          calib_dir, 
          dump_dir,
          science_seqs, 
          dark_seqs, 
          dark_for_flat_seq,
          flat_seq, 
          style = naming_style, 
          background_mode = background_mode,
          remake_darks_and_flats = remake_darks_and_flats)

With the science images all calibrated and the noise removed, they are now ready for photometric analysis. Perform photometry by calling the perform_photometry function if the photometric_extraction flag is turned on, and pass in all the necessary parameters:

.. code-block:: Python

  if photometric_extraction:
      with warnings.catch_warnings():
        warnings.simplefilter("ignore")
        pu.perform_photometry(
          calib_dir, 
          dump_dir, 
          img_dir,
          science_seqs, 
          source_coords,
          style = naming_style,
          finding_fwhm = finding_fwhm, 
          extraction_rads = extraction_rads,
          background_mode = background_mode,
          ann_rads = ann_rads,
          source_detection_sigma = source_detection_sigma,
          max_num_compars = max_num_compars,
          bkg_fname = bkg)

Finally, fit the extracted photometry for the secondary eclipse profile by calling the fit_lightcurve function with all necessary parameters:

.. code-block:: Python

    if fit_for_eclipse:
      with warnings.catch_warnings():
        warnings.simplefilter("ignore")
          best_ap = fu.quick_aperture_optimize(
            dump_dir,
            img_dir,
            extraction_rads,
            flux_cutoff = 0.9)
          fu.fit_lightcurve(
            dump_dir,
            img_dir,
            best_ap,
            background_mode,
            covariate_names,
            texp,
            r_star_prior,
            t0_prior,
            period_prior,
            a_rs_prior,
            b_prior,
            jitter_prior,
            phase = phase,
            ldc_val = ldc_val,
            fpfs_prior = fpfs_prior,
            tune = tune,
            draws = draws, 
            target_accept = target_accept,
            flux_cutoff = 0.9)


This concludes the k-band tutorial.
