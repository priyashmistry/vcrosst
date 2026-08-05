# vcrosst
Inter-quadrant Cross-talk Correction

Documentation  
https://github.com/priyashmistry/vcrosst/blob/main/Documentation.md

# Random Forest Classifier Models

The trained Random Forest classifier models, along with the files used to train them, can be downloaded from the following SharePoint folder:

**Training Models and Data:**
https://unsw-my.sharepoint.com/:f:/g/personal/z5517274_ad_unsw_edu_au/IgBQx-ykQw22TrWx9BS7u8bvAbQzK8heIA741i7Sh2cByNg?e=8cJAIy

## Installation

Download the following model files:

```text
model_Q1.pkl
model_Q2.pkl
```

Copy both files into:

```text
VeloceDR/cal_files/
```

## Directory Structure

Ensure that the `VeloceDR` directory is located in the same parent directory as the `vcrosst` executable/script.

```text
parent_directory/
├── vcrosst
└── VeloceDR/
    └── cal_files/
        ├── model_Q1.pkl
        └── model_Q2.pkl
```

## Notes

* Both `model_Q1.pkl` and `model_Q2.pkl` are required for the cross-talk correction pipeline.
* The models must be placed in `VeloceDR/cal_files/` before running `vcrosst`.
* If the models are not found in the expected location, the classifier-based pixel classification step will fail.

# Installation

## Mac Installation
### Install Script
`#!/usr/bin/env python3`  
or if using Anaconda  
`#!/Users/username/anaconda3/bin/python3`

### Setup Local bin Directory and Update PATH
`mkdir -p ~/bin`  
`chmod +x ~/bin/vcrosst`  
`grep -qxF 'export PATH="$HOME/bin:$PATH"' ~/.zshrc || echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc`  
`export PATH="$HOME/bin:$PATH"`  

### Setup Calibration Files
`export VELOCEDR_CAL_FILES="/Users/username/bin/VeloceDR/cal_files"`

## Linux Installation
### Install Script
`#!/usr/bin/env python3`  
or if using Anaconda  
`#!/home/username/anaconda3/bin/python3`

### Setup Local bin Directory and Update PATH
`mkdir -p ~/bin`  
`chmod +x ~/bin/vcrosst`  
`grep -qxF 'export PATH="$HOME/bin:$PATH"' ~/.bashrc || echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc`  
`export PATH="$HOME/bin:$PATH"`  

### Setup Calibration Files
`export VELOCEDR_CAL_FILES="/home/username/bin/VeloceDR/cal_files"`

## Windows Installation (Using PowerShell)
### Install Script
Make sure Python 3 is installed and added to PATH.  
Windows doesn't use shebang, so ensure .py files are associated with python3  

### Setup Local bin Directory and Update PATH
Create a bin directory in the user profile  
`New-Item -ItemType Directory -Force -Path "$HOME\bin"`

### Copy your script there
`Copy-Item .\vcrosst.py "$HOME\bin\vcrosst.py"`

### Add bin to PATH permanently for PowerShell
`[Environment]::SetEnvironmentVariable("PATH", "$env:USERPROFILE\bin;$env:PATH", "User")`

### Setup Calibration Files
`$env:VELOCEDR_CAL_FILES = "$HOME\bin\VeloceDR\cal_files"`
