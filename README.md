## Checklist of setup things
- create a Python environment or use your existing one: `python3 -m venv .env`, `source .env/bin/activate`, `pip install -r requirements.txt`
- download `tracks/` dataset from [Google Drive](https://drive.google.com/file/d/11zWYTmj4jxXbsz6bnfiSyVFxZQA4ZVma/view?usp=drive_link), or create your own music dataset with our [yt_dlp wrapper script](https://github.com/dennisfarmer/musicdl)
- Copy these files directly into your shared subgroup project repos. 

> The files to copy are `grid_search.py`, `parameters.py`, `grid_search.ipynb`, and `audio_samples/cafe_ambience.flac`. You might also have to `pip install pydub` if it's not installed. 

> Make sure to copy the cafe ambience file into your `audio_samples/` directory, instead of the parent directory with your Python files.

## Overview of grid search code

- `parameters.py`: globally set parameters in a json file for access by algorithm

- `grid_search.py`: grid search given parameter space subset, SNR ratios,..
- `grid_search.py:run_grid_search()`: perform grid search, returns GridViewer

- `grid_search.py:GridViewer`: object that grid search results are stored in
- `grid_search.py:GridViewer.filename`: path to SQLite database of results

![GridViewer schema](sql/gridviewer_schema.png)

- `grid_search.ipynb`: write your code here

## Example

```py
# write in grid_search.ipynb
import grid_search as gs

parameter_space_subset = {
    "candidates_per_band": [6, 2]
}

signal_to_noise_ratios = [3, 6]

grid_viewer = gs.run_grid_search(
    parameter_space_subset,
    signal_to_noise_ratios,
    n_songs=4
)

```

## View results of grid search using SQLite

- using command line interface:
    - `sqlite3 sql/gridviewer.db`
- using Python:
```py

import sqlite3

con = sqlite3.connect(grid_viewer.filename)  # sql/gridviewer.db
cur = con.cursor()
res = cur.execute(
    "SELECT * "
    "FROM paramsets "
    "JOIN results on paramsets.paramset_idx = results.paramset_idx "
    "WHERE results.proportion_correct > 0.85 "
    "ORDER BY results.proportion_correct DESC "
    "LIMIT 10"
    ).fetchall()[0]

print(res)
```

| paramset_idx | cm_window_size | candidates_per_band | bands  | fanout_t | fanout_f | paramset_idx | snr | proportion_correct | avg_search_time_s |
|--------------:|---------------:|--------------------:|:------|---------:|---------:|-------------:|----:|-------------------:|------------------:|
| 0             | 10             | 6                   | �^D�* | 100      | 1500     | 0            | 3.0 | 0.8                | 0.00252225399017334 |

```py

# side note: bands are stored as BLOB objects (collection of bytes) in database
# convert to Python object using pickle.loads() - load pickle object string
import pickle
bands_for_paramset_0 = pickle.loads(res[3])
print(bands_for_paraset_0)

# [ 0, 10 ], [ 10, 20 ], [ 20, 40 ], [ 40, 80 ], [ 80, 160 ], [ 160, 512 ] ]

```

## Test optimal model on larger dataset, inspect audio samples with given signal-to-noise ratio

```py
from DBcontrol import init_db

n_songs = 10  # n_songs=None to use all songs
signal_to_noise_ratio = -3  # dB, negative => more noise than signal
samples = gs.sample_from_source_audios(n_songs, snr=signal_to_noise_ratio)

# initialize database with selected parameter set
# **dict notation:
#         convert {"k1": "v1", "k2: "v2"}
#               -> k1="v1", k2="v2"
gs.set_parameters(**paramset)
init_db(n_songs=n_songs)

proportion_correct, results = gs.perform_recognition_test(
    n_songs=n_songs, samples=samples,
    perform_no_index_searches=True,
    store_microphone_samples=True)

print(f"proportion correct = {proportion_correct}")
print(f"snr = {signal_to_noise_ratio}")
df = pd.DataFrame(results)
df.head()

first_audio_sample = df["microphone_audio"][0]

ipd.display(ipd.Audio(first_audio_sample, rate=11025))
```