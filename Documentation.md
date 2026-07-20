# VeloceDR Correction Pipeline Documentation

# Correction Pipeline Documentation
Our script is written in `Python`. It contains several functions, each designed to perform a specific task. The details of each function, including its purpose and the arguments it requires, are provided below.

## Initiate the Script
The script includes two functions, `main()` and `process_file()`, which handle user input commands and direct the script to the appropriate functions to carry out the requested operations.

- **`main()`**  
  This function serves as the entry point for processing *Veloce* Flexible Image Transport System (FITS) files. It parses command-line arguments provided by the user and initiates the process of inter-quadrant cross-talk correction. All available arguments are listed in the table below.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `-p` | Flag | `False` | Enable diagnostic plotting and printing. |
| `-l FILE` | String | `None` | Path to a text file listing filenames to process. |
| `-v` | Flag | `False` | Enable verbose logging. |
| `-cal` | Flag | `False` | Calibrate to create new correction coefficients. |
| `-cal_t1_corr` | String | `None` | Use SimTh frames to correct type I cross-talk. Provide a list of SimTh files. |
| `--retrain-model` | Flag | `False` | Force retraining of the mask classifier. |
| `--mask1_threshold` | Integer | `6000` | Threshold for mask from Q1. |
| `--mask2_threshold` | Integer | `4000` | Threshold for mask from Q2. |
| `--q4_q1_cfs` | String | `"[0,None,None,None,None]"` | Correction coefficients for Q4 from Q1. `None` can be changed to any number. |
| `--q4_q2_cfs` | String | `"[0,None,None,None,None]"` | Correction coefficients for Q4 from Q2. |
| `--q3_q2_cfs` | String | `"[0,None,None,None,None]"` | Correction coefficients for Q3 from Q2. |
| `--q3_q1_cfs` | String | `"[0,None,None,None,None]"` | Correction coefficients for Q3 from Q1. |
| `--q2_q1_cfs` | String | `"[0,None,None,None,None]"` | Correction coefficients for Q2 from Q1. |
| `--q1_q2_cfs` | String | `"[0,None,None,None,None]"` | Correction coefficients for Q1 from Q2. |
| `--skip` | String | `None` | Comma‑separated list of corrections to skip, e.g., `q1_q2,q2_q1`. |
| `files` | List | `[]` | Directories, glob patterns, or filenames to process. |

- **`process_file(plot_flag, l, v, cal, mask1_threshold, mask2_threshold, q4_q1_cfs, q4_q2_cfs, q3_q2_cfs, q3_q1_cfs, q2_q1_cfs, q1_q2_cfs, skip, image_file, simth_list=None, retrain_model=False, log_dir=None)`**  
  This function serves as the initial stage of preprocessing. It handles tasks such as reading the FITS file, dividing it into separate quadrants, and interpreting the user’s input to determine which operations to perform. Based on the command, it then routes each image quadrant to the appropriate functions—for example, to generate a source mask, create a calibration, or apply default correction coefficients for cross-talk correction. Finally, it saves the corrected image to the specified path. All relevant commands are listed in the table above.

## Logging
Logging is important so that once the code is deployed into a larger data reduction pipeline, we always have a record of what has been done to each file.

- **`get_timestamped_log_name()`**  
  This function generates a log file name with a timestamp.

  **Returns:**  
  - A log file name in the format ``vcrosst_DDMMYYYY_HHMMSS.log``.

- **`setup_logger(log_file)`**  
  This function sets up a logger that writes output to both a file and the console. The logger is uniquely identified by the current process ID, making it suitable for use in multiprocessing environments. For clarity, the log format excludes the process name. It takes `log_file` as a string argument—specifying the path to the log file—and returns a configured logger instance.

  **Args:**  
  - `log_file (str)`: The path to the log file to write to.

  **Returns:**  
  - `logging.Logger`: Configured logger instance.

## File I/O, Parallel & Batch Processing
Before running any correction modules, it is important to ensure that the image data is provided in the correct format. To reduce overall processing time on a multi-core system, parallelizing the program across small batches of images is beneficial. Once a correction is applied, the output file must be saved under a different name (preserving the original file). These operations are carried out by the following functions:

- **`get_files_from_input(input_paths, l, logger)`**  
  This function extracts a list of valid FITS files matching `*o.fits` from various input formats: directories (by collecting all matching `*o.fits` files), text files containing lists of `*o.fits` files, individual FITS files, and lists specified using brace expansion syntax.

  **Args:**  
  - `input_paths (list[str])`: List of file paths, directories, or list files.  
  - `l (bool)`: Interpret `.txt` files as lists of relative file names.  
  - `logger (logging.Logger)`: Logger for reporting info and warnings.

  **Returns:**  
  - `list[str]`: Sorted list of valid FITS file paths ending in `*o.fits`.

- **`get_optimal_max_files()`**  
  Determines the optimal number of files to process in parallel by evaluating the system’s available cores and memory. Starts with half the total number of cores to balance performance and overhead. Reduces parallelism if memory < 8 GB, increases if memory > 16 GB.

  **Returns:**  
  - `int`: Recommended maximum number of files to process concurrently.

- **`get_optimal_workers()`**  
  Estimates the optimal number of worker processes for parallel execution based on the system’s CPU and memory resources. Caps at 8 or total cores, reduces if memory < 8 GB, ensures at least 2 workers.

  **Returns:**  
  - `int`: Recommended number of worker processes.

- **`process_files_in_batches(files, args, log_dir=None)`**  
  Processes a list of FITS files in parallel by dividing them into batches and executing each batch concurrently using a process pool. Sends batches to `process_file()` for correction. Creates `log_dir` if it doesn’t exist.

  **Args:**  
  - `files (list[str])`: List of FITS file paths to process.  
  - `args (Namespace)`: Parsed command-line arguments containing processing parameters.  
  - `log_dir (str, optional)`: Directory to save temporary log files.

  **Returns:**  
  - `str`: Path to the temporary log directory.

- **`load_fits_image(file_path, logger)`**  
  Loads image data from a FITS file as a `NumPy` n-dimensional array. Reads the primary HDU (extension 0). Logs and raises an error if the file cannot be read.

  **Args:**  
  - `file_path (str)`: Path to the FITS file.  
  - `logger (logging.Logger)`: Logger instance for errors.

  **Returns:**  
  - `numpy.ndarray`: The image data from the FITS file.

- **`write_to_csv(fits_file, loc, data, suffixes, csv_file, headers, logger)`**  
  Creates, appends to, or updates a CSV file with data from a FITS file, ensuring column order matches headers. Updates existing entries by `UTMJD` and adds new entries if missing.

  **Args:**  
  - `fits_file (str)`: Path to the FITS file for metadata extraction.  
  - `loc (str)`: Location string used to prefix suffixes for column names.  
  - `data (dict)`: Dictionary of data values to write into the CSV.  
  - `suffixes (list[str])`: Suffixes appended to the location for column keys.  
  - `csv_file (str)`: Path to the CSV file.  
  - `headers (list[str])`: List of all expected CSV column headers.  
  - `logger (logging.Logger)`: Logger instance.

- **`save_multi_extension_fits(corrected_data, mask_data, save_path, logger)`**  
  Saves two arrays into a multi-extension FITS file. The primary HDU contains the corrected image; the secondary HDU stores the mask of corrected pixels (0 = unaltered, 1 = Type I, 2 = Type II/III).

  **Args:**  
  - `corrected_data (numpy.ndarray)`: Primary image data.  
  - `mask_data (numpy.ndarray)`: Secondary image data for mask.  
  - `save_path (str)`: Path where the FITS file will be written.  
  - `logger (logging.Logger)`: Logger instance for errors.

## Data Extraction
It is essential to apply corrections at the appropriate locations using the correct correction coefficients. Based on the user input along with the data read from the file header, the script can know which quadrant is being corrected, the source quadrant of the cross-talk, and which correction factor should be applied. If the user chooses to create a new calibration, the script must also use two additional calibration files: the simultaneous thorium and laser comb masks. In this case, the script checks for the presence of laser comb lines along with thorium lines to avoid superimposed cross-talk on those features before generating the calibration. These tasks are carried out by various functions described below.

- **`amp_mode(CFs, mjd_row, logger)`**  
  Reads the amplifier mode used during data acquisition (2-amp or 4-amp) from the second column of the correction coefficients (CFs) array. Determines whether certain quadrant corrections should be skipped.

  **Args:**  
  - `CFs (np.ndarray)`: ND array of correction coefficients.  
  - `mjd_row (int)`: Row index to read.  

  **Returns:**  
  - `int`: Amplifier mode value (2 or 4) for the specified row.

- **`convert_loc_to_q_format(loc)`**  
  Converts a quadrant-pair location string, e.g., `'4_1'`, into the format `'Q4_Q1'`. Used when preparing CSV headers.

  **Args:**  
  - `loc (str)`: Location string in the format `'X_Y'`, e.g., `'4_1'`.  

  **Returns:**  
  - `str`: Converted location string, e.g., `'Q4_Q1'`.

- **`extract_fits_headers(fits_file, logger)`**  
  Extracts the `UTMJD` (epoch) and `FILEORIG` (filename) from a FITS file header. Determines which row of the CFs array corresponds to the file’s MJD.

  **Args:**  
  - `fits_file (str)`: Path to the FITS file.  
  - `logger (logging.Logger)`: Logger instance.  

  **Returns:**  
  - `utmjd (float)`: Modified Julian Date from the FITS header.  
  - `fileorig (str)`: Filename stored in FITS metadata.

- **`check_lfc_lines(fits_file, logger=None)`**  
  Checks whether a FITS file contains laser frequency comb (LFC) lines based on header flags (`VIDSLCR` for red arm, `VIDSLCB` for blue arm).

  **Args:**  
  - `fits_file (str)`: Path to the FITS file.  
  - `logger (logging.Logger, optional)`: Logger for diagnostics.  

  **Returns:**  
  - `bool`: True if LFC lines are present, otherwise False.

- **`get_data_for_location(CFs, mjd_row, loc, logger)`**  
  Retrieves the calibration data for a specific location and row from the CFs array. Extracts calibration factors (`m` and `c` values) for the given location using a predefined mapping.

  **Args:**  
  - `CFs (np.ndarray)`: 2D array of calibration factors (rows = MJDs, columns = calibration parameters).  
  - `mjd_row (int)`: Row index corresponding to the MJD.  
  - `loc (str)`: Location identifier, e.g., `"4_1"`, `"3_2"`.  
  - `logger (logging.Logger)`: Logger for error messages.  

  **Returns:**  
  - `tuple`: Calibration values for the specified location and row:  
    (`m_2, m_2e, c_2, c_2e, m_3, m_3e, c_3, c_3e`), where:  
    - `m_2`, `m_3`: Calibration slopes for Type II and III pixels  
    - `m_2e`, `m_3e`: Uncertainties of slopes  
    - `c_2`, `c_3`: Calibration intercepts  
    - `c_2e`, `c_3e`: Uncertainties of intercepts
   
- **`load_and_mean_fits(input_source, logger)`**
  Loads one or more FITS images and returns either the input image or their pixel-wise mean. The input may be a single FITS file, a comma-separated list of FITS files, or a text file containing one FITS filename per line. If multiple images are provided, they are averaged after verifying that all have identical dimensions.

  **Args:**
  
  - `input_source (str)`: Input specification. Can be a FITS filename, a comma-separated list of FITS filenames, or a text file containing FITS filenames.
  - `logger (logging.Logger)`: Logger used to record processing information.
  
  **Returns:**
  
  - `np.ndarray`: Loaded FITS image if a single file is supplied; otherwise, the pixel-wise mean of all loaded FITS images.

## Pre-processing
After extracting the data and corresponding image for a given file, some preprocessing steps are required—such as dividing the image into four quadrants and generating a mask from the two source quadrants, Q1 and Q2. Based on user inputs, the script then decides whether to use the default correction coefficients or to derive a new calibration. These tasks are performed by the following functions:

- **`get_source_mask(array, lower_lim, sat_lim=60000, size_lim=50)`**  
  Creates a mask from a source quadrant. Applies a saturation limit to identify Type I pixels, then expands to include Type II and III pixels. Small blobs below the size limit are discarded.

  **Args:**  
  - `array (np.ndarray)`: 2D image data of a source quadrant (Q1 or Q2).  
  - `lower_lim (float)`: Pixel intensity threshold for region growth.  
  - `sat_lim (float, optional)`: Saturation threshold; default 60000.  
  - `size_lim (int, optional)`: Minimum pixel count for retained regions; default 50.

  **Returns:**  
  - `np.ndarray`: Boolean mask of the same shape as input, with `True` for detected source pixels.

- **`simth_t1_corr(image_file, ct, simth_file, type_1_mask, full_mask, th_mask, lc_mask, lc_on, logger, loc, plot_flag, axis=None)`**  
  Performs calibration-based Type I cross-talk correction using a SimTh frame.

  The function estimates the Type I cross-talk signal by removing the background from a mean-combined SimTh image. The extracted Type I signal is then subtracted from the cross-talk quadrant only at pixels identified as Type I.

  **Args:**  
  - `image_file (str)`: Path to the FITS image being processed.  
  - `ct (np.ndarray)`: Cross-talk quadrant to be corrected.  
  - `simth_file (np.ndarray)`: Mean-combined SimTh image used to estimate the Type I cross-talk signal.  
  - `type_1_mask (np.ndarray)`: Binary mask identifying Type I cross-talk pixels.  
  - `full_mask (np.ndarray)`: Complete source mask used to identify cross-talk regions.  
  - `th_mask (np.ndarray)`: Threshold-based source mask.  
  - `lc_mask (np.ndarray)`: Machine-learning source mask.  
  - `lc_on (bool)`: If `True`, both the machine-learning and threshold masks are used to identify background pixels. Otherwise, only the threshold mask is used.  
  - `logger (logging.Logger)`: Logger used to record processing information and errors.  
  - `loc (str)`: Identifier of the quadrant pair being corrected.  
  - `plot_flag (bool)`: If `True`, generates diagnostic images of the background estimation and extracted Type I signal.  
  - `axis (int, optional)`: Axis used when flipping the source mask to align it with the cross-talk quadrant.

  **Returns:**  
  - `np.ndarray`: Cross-talk quadrant after applying the calibration-based Type I correction.

- **`correction_using_known_cfs(image_file, mjd_input, CFs, mjd_row, final_mask, v, plot_flag, source, ct, logger, th_mask, lc_mask, lc_on, simth_file, shifter_value, loc, axis=None)`**  
  Applies inter-quadrant cross-talk correction using pre-determined calibration coefficients.

  The function corrects cross-talk pixels by applying the appropriate correction model for the specified quadrant pair. Type I pixels are corrected using either the default trend-based method or a calibration-based method when a SimTh frame is provided. Type II and Type III pixels are corrected using the supplied calibration coefficients.

  **Args:**  
  - `image_file (str)`: Path to the FITS image being processed.  
  - `mjd_input (int or float)`: Modified Julian Date (MJD) of the observation.  
  - `CFs (np.ndarray)`: Array containing the calibration coefficients for all supported MJD ranges.  
  - `mjd_row (int)`: Index of the row in `CFs` corresponding to the selected MJD.  
  - `final_mask (np.ndarray)`: Pixel classification mask identifying the cross-talk pixel types.  
  - `v (bool)`: Enables verbose logging.  
  - `plot_flag (bool)`: Enables diagnostic plots.  
  - `source (np.ndarray)`: Source quadrant contributing the cross-talk signal.  
  - `ct (np.ndarray)`: Cross-talk quadrant to be corrected.  
  - `logger (logging.Logger)`: Logger used to record processing information.  
  - `th_mask (np.ndarray)`: Threshold-based mask identifying saturated source pixels.  
  - `lc_mask (np.ndarray)`: Machine-learning source mask.  
  - `lc_on (bool)`: If `True`, uses the machine-learning mask; otherwise, only the threshold mask is used.  
  - `simth_file (np.ndarray or None)`: Mean-combined SimTh image used for calibration-based Type I correction. If `None`, the trend-based correction is applied.  
  - `shifter_value (float)`: Offset applied to the correction model where required.  
  - `loc (str)`: Identifier of the quadrant pair being corrected (e.g. `"q4_q1"`).  
  - `axis (int, optional)`: Axis along which the arrays are flipped before or after correction, depending on the quadrant orientation.

  **Returns:**  
  - `np.ndarray`: Corrected cross-talk quadrant.

- **`get_correction(image_file, final_mask, shifter_value, m2, c2, m3, c3, v, plot_flag, source, ct, logger, th_mask, lc_mask, lc_on, simth_file, loc, axis=None)`**  
  Applies Type I, Type II, and Type III cross-talk corrections to a single quadrant using the supplied calibration coefficients.

  The function first estimates and removes the local background from the cross-talk quadrant before applying linear correction models to Type II and Type III pixels. Type I pixels are then corrected using either the default trend-based method or the calibration-based SimTh method, depending on whether a SimTh frame is provided.

  **Args:**  
  - `image_file (str)`: Path to the FITS image being processed.  
  - `final_mask (np.ndarray)`: Pixel classification mask containing the identified cross-talk pixel types.  
  - `shifter_value (float)`: Offset used by the trend-based Type I correction.  
  - `m2 (float)`: Initial slope for the Type II correction model.  
  - `c2 (float)`: Initial intercept for the Type II correction model.  
  - `m3 (float)`: Initial slope for the Type III correction model.  
  - `c3 (float)`: Initial intercept for the Type III correction model.  
  - `v (bool)`: Enables verbose logging.  
  - `plot_flag (bool)`: Enables generation of diagnostic plots.  
  - `source (np.ndarray)`: Source quadrant responsible for the cross-talk signal.  
  - `ct (np.ndarray)`: Cross-talk quadrant to be corrected.  
  - `logger (logging.Logger)`: Logger used to record processing information.  
  - `th_mask (np.ndarray)`: Threshold-based source mask.  
  - `lc_mask (np.ndarray)`: Machine-learning source mask.  
  - `lc_on (bool)`: If `True`, the machine-learning mask is used together with the threshold mask when estimating the background.  
  - `simth_file (np.ndarray or None)`: Mean-combined SimTh image used for calibration-based Type I correction. If `None`, the default trend-based correction is applied.  
  - `loc (str)`: Identifier of the quadrant pair being corrected.  
  - `axis (int, optional)`: Axis used when flipping arrays to align the source and cross-talk quadrants.

  **Returns:**  
  - `np.ndarray`: Cross-talk quadrant after applying all applicable Type I, Type II, and Type III corrections.  
  - `np.ndarray`: Pixel classification mask aligned with the returned corrected quadrant.

- **`linear_fit(x, y, m=None, c=None)`**  
  Performs linear regression with optional fixed slope (`m`) and/or intercept (`c`). Uses `sklearn` and `scipy` for fitting and uncertainty estimation.

  **Args:**  
  - `x (1D array)`: Independent variable.  
  - `y (1D array)`: Dependent variable.  
  - `m (float, optional)`: Fixed slope; estimated if `None`.  
  - `c (float, optional)`: Fixed intercept; estimated if `None`.

  **Returns:**  
  - `dict`: Dictionary containing slope (`m`), intercept (`c`), and uncertainties.

- **`run_linear_fits(X_t2, Y_t2, X_t3, Y_t3, m2=None, c2=None, m3=None, c3=None)`**  
  Performs separate linear fits for Type II and III pixels. Optional fixed slopes/intercepts; returns fitted slopes, intercepts, uncertainties, and predicted values.

  **Args:**  
  - `X_t2, Y_t2 (1D array)`: Independent/dependent data for Type II pixels.  
  - `X_t3, Y_t3 (1D array)`: Independent/dependent data for Type III pixels.  
  - `m2, c2, m3, c3 (float, optional)`: Slopes/intercepts for fitting; calculated if `None`.

  **Returns:**  
  - `dict`: Fitting results for both Type II and III data.

- **`apply_random_forest_with_bounding_boxes(loc_data, ct, shifter_value, bounding_boxes, threshold=160)`**  
  Applies a random forest model to detrend data within specified locations, smooths predictions using Lowess smoothing, and adjusts values based on a threshold to match the background level.

  **Args:**  
  - `loc_data (np.ndarray)`: 2D array of locations to be corrected.  
  - `ct (np.ndarray)`: 2D array to apply corrections to.  
  - `shifter_value (float)`: Value added to detrended data to match background.  
  - `bounding_boxes (List[Tuple[Tuple[int, int], Tuple[int, int]]])`: List of bounding boxes enclosing cross-talk regions, each defined by `(row_start, row_end)` and `(col_start, col_end)`.  
  - `threshold (float, optional)`: Ignore pixels above this value; default `160`.

  **Returns:**  
  - `np.ndarray`: Quadrant with corrected Type I cross-talk.

## Random Forest Classifier

- **`extract_features(img, coords)`**
  Extracts feature vectors for image pixels to be used by the machine-learning source mask classifier. For each pixel, the extracted features include the pixel intensity, local intensity gradients, local 3×3 standard deviation, and the normalised pixel coordinates.

  **Args:**

  - `img (np.ndarray)`: 2D detector image.
  - `coords (np.ndarray)`: Array of pixel coordinates with shape (N, 2), where each row contains the (row, column) index of a pixel.

  **Returns:**

  - `np.ndarray`: Feature matrix of shape (N, M), where each row contains the extracted features for a single pixel.
  
- **`type_mask(img, mask, model)`**
  Classifies candidate source pixels into cross-talk types using a trained machine-learning model. Pixels above the detector saturation threshold are automatically classified as Type I sources. Remaining candidate pixels within the input mask are classified using the supplied machine-learning model.

  **Args:**

  - `img (np.ndarray):` 2D detector image.
  - `mask (np.ndarray):` Binary mask identifying candidate source pixels to classify. Only non-zero pixels are processed by the model.
  - `model (sklearn.base.BaseEstimator):` Trained machine-learning classifier used to distinguish source pixel types.
  
  **Returns:**
  
  - `np.ndarray`: Integer mask with the same shape as img, containing the predicted source classifications.
