# Matching outbreak reports to OSHA complaints. 

These scripts identify establishments with both an OSHA complaint, and a state-level outbreak report (such as those collected by state departments of health).

### Description:
This pipeline takes as input:
1. A dataset of **OSHA complaints**, such as [these](https://www.osha.gov/foia/archived-covid-19-data). 
2. A dataset of **workplace outbreak reports** for a specific state (e.g., [these for Colorado](https://covid19.colorado.gov/covid19-outbreak-data)).

It **matches** one dataset to the other in a two-step process:
1. **Fuzzy matching**:  provides a best-guess match based on establishment names and addresses.
2. **Verification**:  verifies each match by querying the Google Maps Geocoding API w/ the business name +
      address from the state dataset, and then the OSHA dataset.  If they resolve to the
      same GMaps listing, then the match is good.

**Caveat:**  The result is usually quite accurate, but there may be a few false positives / negatives. To find these, we recommend manually inspecting the matches.  To facilitate this, the first four columns of the output data are the establishment name and address from each dataset, followed by a "human verified?" column.  Additionally, the matches which failed step two (verification) are sorted from strongest to weakest, so most false negatives will be near the top.

**Caveat:**  Matching does not take into account the **date/time** of the outbreaks/complaints.  So, if `business_x` has an OSHA complaint from April 2020, and a state outbreak report from August 2020, that's still a match.

## Usage

### Setup and installation
**NOTE**:  You will need a key for the GMaps Geocoding API.  This must be placed in a file called `config.yml`, with contents `gmaps_api_key: your_key_here`.

1.  Install environment:  `conda env create osha_env.yaml`

### Running the pipeline

Each state has its own file, `[XX]DataProcessor`, which extends `StateDataProcessor`.  To run matching for, say, the Colorado data:
1. Specify the filepath of the source datasets, and the matching direction (see FAQ), in the `__main__` method of `CODataProcessor.py`.
2. Run `python CODataProcessor.py`

The **matching dataset** will appear in `./output/`.  Its filename will be of the form:

```[two letter state abbreviation]_[timestamp of creation]__all_rows__[left dataset]__[right_dataset].csv```

### Format of the output dataset.

The columns are ordered as follows:
- establishment names and addresses for both datasets.
- Column _"computer: is it a match?"_: the result of GMaps verification.
- [all of the **left** dataset's columns]
- [all of the **right** dataset's columns]

## Writing your own state data processor

To implement matching for a new state, you'll need to make a new subclass of `StateDataProcessor`.  `StateDataProcessor` was designed to make this as simple as possible.  Your derived class need only specify which columns in the outbreak dataset are used for establishment names and addresses, as well as any preprocessing necessary for matching.

`StateDataProcessor` splits the matching process into three steps:
1. `preprocess()`: do anything needed to prepare the data for matching.
2. `process()`: do the fuzzy matching + GMaps API verification.
3. `postprocess()`: do any formatting / etc. needed before matches are saved.

`StateDataProcessor`'s subclasses **must** override `preprocess()`, and can optionally override the other functions as well.
 - _For a simple example, see `ARDataProcessor.py`.  For a more complex example, see `NMDataProcessor.py`_

## FAQ

_**Q:** What is the `raw_matches` dataset I get in addition to the matching dataset?_  
- **A:** It's the output of the fuzzy matching step, before GMaps verification.  It also includes extra columns, which can be useful for debugging.

_**Q:** What is "matching direction"?   What are the "left/right datasets"?_
- **A:**  This relates to fuzzy matching.  For every entry in the **left** dataset, the fuzzy matcher identifies the best possible match in the **right** dataset.  **Repeats of the right datset are allowed!**  You can choose which dataset (OSHA or state) is "left" or "right", depending on your needs.
