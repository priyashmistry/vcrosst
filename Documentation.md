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
| `--mask1_threshold` | Integer | `6000` | Threshold for mask from Q1. |
| `--mask2_threshold` | Integer | `4000` | Threshold for mask from Q2. |
| `--q4_q1_cfs` | String | `"[0,None,None,None,None]"` | Correction coefficients for Q4 from Q1. `None` can be changed to any number. |
| `--q4_q2_cfs` | String | `"[0,None,None,None,None]"` | Correction coefficients for Q4 from Q2. |
| `--q3_q2_cfs` | String | `"[0,None,None,None,None]"` | Correction coefficients for Q3 from Q2. |
| `--q3_q1_cfs` | String | `"[0,None,None,None,None]"` | Correction coefficients for Q3 from Q1. |
| `--q2_q1_cfs` | String | `"[0,None,None,None,None]"` | Correction coefficients for Q2 from Q1. |
| `--q1_q2_cfs` | String | `"[0,None,None,None,None]"` | Correction coefficients for Q1 from Q2. |
| `--skip` | String | `""` | Comma‑separated list of corrections to skip, e.g., `q1_q2,q2_q1`. |
| `files` | List | `[]` | Directories, glob patterns, or filenames to process. |

- **`process_file(plot_flag, l, v, cal, mask1_threshold, mask2_threshold, q4_q1_cfs, q4_q2_cfs, q3_q2_cfs, q3_q1_cfs, q2_q1_cfs, q1_q2_cfs, skip, image_file, log_dir=None)`**  
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

- **`correction_using_known_cfs(image_file, mjd_input, CFs, mjd_row, final_mask, type1_pixels, v, plot_flag, source, ct, logger, shifter_value, loc, axis=None)`**  
  Applies pre-calculated correction coefficients to an image using a Gaussian Mixture Model for Type II/III pixels and a random forest model for Type I pixels.

  **Args:**  
  - `image_file (str)`: Path of the image.  
  - `mjd_input (int)`: MJD corresponding to calibration row.  
  - `CFs (np.ndarray)`: 2D array of calibration factors.  
  - `mjd_row (int)`: Row index of calibration.  
  - `final_mask (np.ndarray)`: Mask indicating areas for correction.  
  - `type1_pixels (np.ndarray)`: Mask for Type I pixels.  
  - `v (bool)`: Verbose logging flag.  
  - `plot_flag (bool)`: Flag to generate diagnostic plots.  
  - `source (np.ndarray)`: Source quadrant pixel data.  
  - `ct (np.ndarray)`: Target quadrant pixel data.  
  - `logger (logging.Logger)`: Logger instance.  
  - `shifter_value (float)`: Pixel shift to background noise level.  
  - `loc (str)`: Location key for calibration.  
  - `axis (int, optional)`: Axis for flipping; default `None`.

  **Returns:**  
  - `ct_corrected (np.ndarray)`: Corrected image data.  
  - `combined_mask (np.ndarray)`: Array indicating pixel types 0 (unaffected), 1 (Type I), 2 (Type II/III).

- **`get_correction(image_file, final_mask, type1_pixels, source, ct, shifter_value, m2, c2, m3, c3, plot_flag, v, th_mask, lc_mask, lc_on, logger, loc, axis=None)`**  
  Calculates correction coefficients for all spectra by identifying Type II/III pixels, fitting models, and returning best-fit slopes and intercepts.

  **Args:**  
  - `image_file (str)`: Path to FITS image.  
  - `final_mask (np.ndarray)`: Mask for cross-talk-affected pixels.  
  - `type1_pixels (np.ndarray)`: Mask for Type I pixels.  
  - `source (np.ndarray)`: Source quadrant pixels.  
  - `ct (np.ndarray)`: Target quadrant pixels.  
  - `shifter_value (float)`: Additional correction shift.  
  - `m2, c2 (float)`: Initial slope/intercept for Type II.  
  - `m3, c3 (float)`: Initial slope/intercept for Type III.  
  - `plot_flag (bool)`: Flag to generate diagnostic plots.  
  - `v (bool)`: Verbose logging flag.  
  - `th_mask (np.ndarray)`: Thorium mask.  
  - `lc_mask (np.ndarray)`: Laser comb mask.  
  - `lc_on (bool)`: Whether `lc_mask` is applied.  
  - `logger (logging.Logger)`: Logger instance.  
  - `loc (str)`: Quadrant location string.  
  - `axis (int, optional)`: Axis for flipping; default `None`.

  **Returns:**  
  - `ct_corrected (np.ndarray)`: Corrected image data.  
  - `combined_mask (np.ndarray)`: Array indicating pixel types.

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

## Correction
Once all locations requiring correction have been identified and the corresponding correction values determined, the script uses the following functions—one for Type I pixels, and another for Type II and III pixels:

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

- **`make_correction(ct, loc_data, source, separation_point, m_2, c_2, m_3, c_3, logger)`**  
  Applies corrections to an array using a source array. The correction differs depending on whether pixels are Type II or III.

  **Args:**  
  - `array (np.ndarray)`: Array to apply the correction to.  
  - `loc_data (np.ndarray)`: 2D array of locations to correct.  
  - `source (np.ndarray)`: Source data array used for correction.  
  - `separation_point (float)`: Threshold separating Type II and III pixels.  
  - `m_2, c_2 (float)`: Slope and intercept for Type II correction.  
  - `m_3, c_3 (float)`: Slope and intercept for Type III correction.  
  - `logger (logging.Logger)`: Logger instance for errors.

  **Returns:**  
  - `np.ndarray`: Quadrant with corrected Type II and III pixels.
