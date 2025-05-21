Okay, here's the documentation for your Python script, designed to be clear, concise, and suitable for a GitHub README.

-----

# Batch Video Trimmer for Google Colab 🎥✂️

This script automates the process of trimming multiple video clips based on timestamps provided in a CSV file. It's designed to run in a Google Colab environment, leveraging Google Drive for video storage and FFmpeg for video processing. The key feature is its ability to adjust trim times based on a user-provided reference timestamp from the first frame of each video, ensuring accuracy even if videos don't start exactly at time zero.

## 🌟 Features

  * **Batch Processing:** Trims multiple videos listed in a CSV file.
  * **Google Drive Integration:**
      * Mounts your Google Drive to access source videos.
      * Saves trimmed videos to a specified folder in your Google Drive.
  * **Dynamic Timestamp Correction:**
      * Displays the first frame of each video.
      * Prompts the user to input the timestamp visible on this first frame (`T_ref`).
      * Calculates precise FFmpeg start and duration times by offsetting the target trim times (from CSV) with `T_ref`.
  * **User-Friendly:** Provides clear prompts and feedback during processing.
  * **Customizable Output:**
      * Generates output filenames based on the original video name and a task number from the CSV.
      * Saves trimmed videos to a dedicated output directory.
  * **Efficient Trimming:** Uses `ffmpeg -c copy` for fast, lossless trimming where possible.
  * **Error Handling:** Skips problematic videos or tasks and reports issues.
  * **Dependencies Handled:** Includes a function to install FFmpeg within the Colab environment.

-----

## ⚙️ Prerequisites

  * **Google Account:** To use Google Colab and Google Drive.
  * **Source Videos:** Videos to be trimmed should be uploaded to a specific folder in your Google Drive.
  * **CSV File:** A CSV file listing the videos and their target trim times.

-----

## 📋 CSV File Format

The script expects a CSV file with at least the following columns:

1.  **`No`**: (Optional, but recommended) A unique identifier or task number for each video. Used in the output filename. If missing or empty, the script uses an internal row index.
2.  **`PIC Trimming`**: The name of the person assigned to trim the video. The script specifically filters for tasks assigned to 'Firdaus' (case-insensitive).
3.  **`Link Video Magang`**: The exact filename of the video (e.g., `my_video.mp4`) located in the specified Google Drive search path.
4.  **`Timestamp`**: The target start and end times for trimming, formatted as `HH:MM:SS sd HH:MM:SS` (e.g., `01:10:00 sd 01:12:30`). The `sd` (stands for "sampai dengan" or "until") is crucial as a delimiter.

**Example CSV:**

| No  | PIC Trimming | Link Video Magang | Timestamp           |
| --- | ------------ | ----------------- | ------------------- |
| 1   | Firdaus      | video\_lecture\_A.mp4 | 00:05:15 sd 00:08:45 |
| 2   | John         | video\_meeting\_B.mp4 | 00:30:00 sd 00:32:10 |
| 3   | Firdaus      | tutorial\_part\_1.mkv | 01:02:10 sd 01:03:00 |

*(The script will only process rows 1 and 3 in this example if filtering for 'Firdaus')*

-----

## 🚀 How to Use

The script is organized into cells in a Google Colab notebook.

1.  **Cell 1: Install FFmpeg**

      * Run this cell first. It installs FFmpeg, which is necessary for video processing. It also prints the FFmpeg version to confirm installation.

    <!-- end list -->

    ```python
    import subprocess
    # ... (install_ffmpeg function and call) ...
    result = subprocess.run(['ffmpeg', '-version'], check=True, capture_output=True, text=True)
    print(result.stdout) # Or just `result` in Colab to see structured output
    ```

2.  **Cell 9: Upload CSV**

      * Run this cell. It will prompt you to upload your prepared CSV file.

    <!-- end list -->

    ```python
    from google.colab import files
    import pandas as pd
    uploaded_csv = files.upload()
    csv_filename = list(uploaded_csv.keys())[0]
    df_check = pd.read_csv(csv_filename)
    # df_check # Optionally display the DataFrame
    ```

3.  **Cell 10: Filter Tasks**

      * This cell filters the DataFrame for tasks assigned to 'Firdaus' and meeting other criteria (valid video links and timestamps).

    <!-- end list -->

    ```python
    firdaus_tasks_df = df_check[
        (df_check['PIC Trimming'].str.lower() == 'firdaus') &
        # ... (other conditions)
    ].copy()
    # firdaus_tasks_df # Optionally display the filtered DataFrame
    ```

4.  **Cell 11: Helper Functions**

      * This cell defines utility functions:
          * `time_to_seconds(time_str)`: Converts HH:MM:SS string to total seconds.
          * `find_file_in_drive(filename_to_find, search_base_path)`: Locates files in Google Drive.
          * `extract_first_frame_from_video(video_path, output_image_name)`: Extracts the first video frame using FFmpeg.
      * Ensure this cell is run so these functions are defined.

5.  **Cell 12: Mount Google Drive**

      * Run this cell to mount your Google Drive. You'll be asked to authorize Colab to access your Drive.

    <!-- end list -->

    ```python
    from google.colab import drive
    drive.mount('/content/drive', force_remount=True)
    ```

6.  **Cell 13: Main Processing Loop** ✨

      * **Configuration:**
          * `DRIVE_SEARCH_PATH`: Modify this path to the folder in your Google Drive containing the source videos (e.g., `"/content/drive/MyDrive/MyVideos"`).
          * `TRIMMED_VIDEOS_OUTPUT_DIR`: Modify this path to the folder in your Google Drive where trimmed videos will be saved (e.g., `"/content/drive/MyDrive/TrimmedOutput"`). The script will create this folder if it doesn't exist.
      * **Run the cell.** The script will then:
          * Iterate through each task assigned to 'Firdaus'.
          * Search for the video file in `DRIVE_SEARCH_PATH`.
          * If found, extract and display the first frame of the video.
          * **User Interaction:** Prompt you to:
            ```
            Enter the (HH:MM:SS) visible on THIS first frame :
            ```
            Look at the displayed image and type the timestamp you see (e.g., `00:01:17`). This is `T_ref`.
            You can also type `skip` to ignore the current video.
          * Parse the target start and end times from the CSV's 'Timestamp' column.
          * Calculate the actual FFmpeg start time and duration (see Workflow Explanation below).
          * Trim the video and save it to `TRIMMED_VIDEOS_OUTPUT_DIR`.
          * Output logs for each step.
      * After processing all tasks, a summary of successful and failed/skipped tasks is displayed.

-----

## 🔄 Workflow Explanation

The core of the script's accuracy lies in how it calculates trim times:

1.  **Reference Time (`T_ref`)**:

      * The script shows you the very first frame of the video file.
      * Often, videos have intros, black screens, or start at a displayed time that isn't 00:00:00.
      * You input the timestamp visually observed on this first frame (e.g., `00:01:17`). This is `T_ref_seconds`.

2.  **Target Times from CSV**:

      * The CSV provides `Timestamp` like `HH:MM:SS sd HH:MM:SS`.
      * These are converted to `t_target_start_seconds` and `t_target_end_seconds` (e.g., if CSV has `00:05:00 sd 00:06:00`, `t_target_start_seconds` is 300s, `t_target_end_seconds` is 360s). These times are relative to the *displayed time overlay* in the video, not necessarily the file's timeline.

3.  **FFmpeg Cut Calculation**:

      * The actual point in the video file to start cutting (`playback_start_cut_seconds`) is:
        `playback_start_cut_seconds = t_target_start_seconds - t_ref_seconds`
      * The actual point in the video file to end cutting (`playback_end_cut_seconds`) is:
        `playback_end_cut_seconds = t_target_end_seconds - t_ref_seconds`
      * **Adjustment:** To ensure the cut doesn't miss the beginning, the script starts trimming slightly earlier:
        `playback_start_cut_seconds_adjusted = max(0, playback_start_cut_seconds - 1.5)` (starts 1.5 seconds earlier, but not before the video's beginning).
      * The **duration** for FFmpeg is then:
        `duration_seconds = playback_end_cut_seconds - playback_start_cut_seconds_adjusted`

4.  **FFmpeg Command**:
    The script constructs an FFmpeg command similar to this:

    ```bash
    ffmpeg -ss [playback_start_cut_seconds_adjusted] -i [input_video_path] -t [duration_seconds] -c copy [output_video_path] -y
    ```

      * `-ss`: Seek to the adjusted start time.
      * `-i`: Specifies the input video.
      * `-t`: Specifies the duration of the trim.
      * `-c copy`: Copies the video and audio streams without re-encoding, which is fast and preserves quality.
      * `-y`: Overwrites the output file if it exists.

5.  **Output Filename**:

      * Trimmed videos are named: `[OriginalBaseName]_TRIM_([TaskNo]).mp4`
      * Example: If original is `video_lecture_A.mp4` and Task No is `1`, output is `video_lecture_A_TRIM_(1).mp4`.
      * Invalid characters for filenames (like `/`, `\`, `:`) are replaced.

-----

## 💡 Important Notes & Troubleshooting

  * **Google Drive Path Casing:** Ensure `DRIVE_SEARCH_PATH` and video filenames in the CSV match the exact casing in your Google Drive.
  * **File Not Found:** If videos aren't found, double-check:
      * The `DRIVE_SEARCH_PATH` is correct.
      * The `Link Video Magang` filenames in the CSV are exact matches (including extensions).
      * The files actually exist in the specified Drive location.
  * **Timestamp Format:**
      * Strictly use `HH:MM:SS` for `T_ref` input.
      * Ensure the CSV 'Timestamp' column uses `HH:MM:SS sd HH:MM:SS`.
  * **FFmpeg Errors:** If FFmpeg encounters issues (e.g., corrupted video, unsupported format for `-c copy`), error messages from FFmpeg will be printed. You might need to remove `-c copy` for re-encoding, which will be slower.
  * **Colab Timeouts:** For very long processing jobs, Colab might disconnect. Process in smaller batches if this occurs.
  * **Permissions:** Ensure Colab has permission to access your Google Drive when prompted.

-----

## 🧩 Dependencies

  * **Python 3**
  * **pandas:** For CSV manipulation.
  * **gdown:** (Used in commented-out initial cells for single file download, not directly in the batch loop but good to have if adapting).
  * **FFmpeg:** External command-line tool for video processing (installed by the script).
  * **IPython:** For display utilities within Colab.

-----
