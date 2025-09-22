---
jupytext:
  formats: md:myst
  text_representation:
    extension: .md
    format_name: myst
kernelspec:
  name: python3          # or "ir" for R
  display_name: Python 3
---
# VIIRS LST from NASA Earthdata


```{code-cell} ipython3
:tags: [hide-input]   # folded by default; click to toggle


# --- Add this block for debugging ---
import logging
import http.client as http_client

http_client.HTTPConnection.debuglevel = 1

logging.basicConfig()
logging.getLogger().setLevel(logging.DEBUG)
requests_log = logging.getLogger("requests.packages.urllib3")
requests_log.setLevel(logging.DEBUG)
requests_log.propagate = True
# --- End of debug block ---

import requests
import h5py
import numpy as np
import matplotlib.pyplot as plt
import os
import json # Import json for pretty printing

# --- Configuration ---
# This is the short name for the dataset we want to access.
DATASET_SHORT_NAME = "VNP21_NRT"
CMR_URL = "https://cmr.earthdata.nasa.gov/search/granules.json"

def search_latest_granule(dataset_id):
    """Finds the download URL for the latest granule of a given dataset."""
    print(f"Searching for the latest file in dataset: {dataset_id}...")

    # --- DEBUG: Print the exact parameters being sent to the API ---
    search_params = {
        "short_name": dataset_id,
        "sort_key": "-start_date",  # Sort by date, descending
        "page_size": 1             # We only want the latest one
    }
    print(f"DEBUG: Using CMR search URL: {CMR_URL}")
    print(f"DEBUG: Search parameters: {search_params}")
    # --- END DEBUG ---

    response = requests.get(CMR_URL, params=search_params)
    response.raise_for_status()  # Raise an exception for bad status codes

    # --- DEBUG: Inspect the raw JSON response from CMR ---
    response_json = response.json()
    results = response_json['feed']['entry']
    print(f"DEBUG: Found {len(results)} granules in the response.")
    # --- END DEBUG ---

    if not results:
        # --- DEBUG: Print the full response if no results are found ---
        print("DEBUG: Full response from CMR since no granules were found:")
        print(json.dumps(response_json, indent=2))
        # --- END DEBUG ---
        raise ValueError(f"No results found for the dataset short_name '{dataset_id}'. Please check the ID.")
        
    # --- DEBUG: Print the entire first granule entry for inspection ---
    print("DEBUG: Inspecting the first (latest) granule found:")
    # Use json.dumps for pretty-printing the dictionary
    print(json.dumps(results[0], indent=2))
    # --- END DEBUG ---

    # Extract the HTTPS download link
    download_url = None
    for link in results[0]['links']:
        if link['rel'].endswith('/data#'):
            download_url = link['href']
            break
            
    if not download_url:
        raise ValueError("Could not find a download link in the search results.")
        
    print(f"Found latest file: {os.path.basename(download_url)}")
    return download_url

def download_file(url, local_filename):
    """Downloads a file from a URL requiring Earthdata login."""
    print(f"Downloading {os.path.basename(local_filename)}...")
    try:
        # The .netrc file is automatically used by the requests library for authentication
        with requests.get(url, stream=True, timeout=60) as r:
            r.raise_for_status()
            with open(local_filename, 'wb') as f:
                for chunk in r.iter_content(chunk_size=8192):
                    f.write(chunk)
        print("Download complete.")
        return local_filename
    except requests.exceptions.RequestException as e:
        print(f"Error downloading file: {e}")
        return None

def process_and_display_vnp21(filepath):
    """Reads a VNP21 HDF5 file, processes, and displays the LST data."""
    print("Processing and displaying the image...")
    with h5py.File(filepath, 'r') as hdf_file:
        # Navigate to the Land Surface Temperature dataset
        data_path = '/VIIRS_LSTE_NRT_Group/Data_Fields/LST'
        if data_path not in hdf_file:
            raise KeyError(f"Dataset '{data_path}' not found in HDF5 file.")
            
        lst_data = hdf_file[data_path][:]
        
        # Read metadata attributes for processing
        attrs = hdf_file[data_path].attrs
        fill_value = attrs.get('_FillValue')
        scale_factor = attrs.get('scale_factor')
        
        # Apply the scale factor and handle the fill value
        if scale_factor is not None:
            lst_data = lst_data.astype(np.float32) * scale_factor
        
        if fill_value is not None:
            lst_data[lst_data == fill_value * scale_factor] = np.nan
        
        # Convert from Kelvin to Celsius for better interpretation
        lst_celsius = lst_data - 273.15
        
        # --- Plotting the Data ---
        plt.style.use('dark_background')
        fig, ax = plt.subplots(1, 1, figsize=(10, 8))
        
        # Use a colormap suitable for temperature data
        image = ax.imshow(lst_celsius, cmap='inferno')
        
        # Add a colorbar to show the temperature scale
        cbar = fig.colorbar(image, ax=ax, shrink=0.75, pad=0.02)
        cbar.set_label('Land Surface Temperature (°C)')
        
        ax.set_title(f'VIIRS Land Surface Temperature\n{os.path.basename(filepath)}', fontsize=14)
        ax.set_xlabel('Column Index')
        ax.set_ylabel('Row Index')
        ax.axis('off') # Hide axes for a cleaner look
        
        plt.tight_layout()
        plt.show()

def main():
    """Main function to run the script."""
    local_filename = None
    try:
        # 1. Find the latest file URL
        download_url = search_latest_granule(DATASET_SHORT_NAME)
        local_filename = os.path.basename(download_url)
        
        # 2. Download the file
        if not download_file(download_url, local_filename):
            raise ValueError("Failed to download the file.")
        
        # 3. Process and display the data
        process_and_display_vnp21(local_filename)
        
    except (ValueError, requests.exceptions.RequestException, KeyError) as e:
        print(f"An error occurred: {e}")
    finally:
        # 4. Clean up by deleting the downloaded file
        if local_filename and os.path.exists(local_filename):
            print(f"Cleaning up and deleting {local_filename}.")
            os.remove(local_filename)

if __name__ == '__main__':
    main()
```
