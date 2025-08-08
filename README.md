# Run automatic self-calibration on NRAO cluster 
All credit to Patrick Sheehan (https://github.com/psheehan/auto_selfcal). This code is designed to streamline running github code auto_selfcal on clusters we have access to at UO and NRAO. 

(TODO) NOTE: this code can be tweaked to run on Talapas with the following changes with respect to filesystem and SLURM scheduler options.

## Best uses for auto_selfcal
I have found auto_selfcal to be good at...
1. Cleaning signal from off-center bright sources in FOV
2. Improving point-source shapes to look more beam-like
3. Producing clean noise pattern free of patches near source
4. Achieving low RMS. Ex: the following RMS values are achieved from 2hr SEDs with strong source signal: 15-40 uJy in L-band, 10-35 uJy in S-band, 20-40 uJy in C-band, and 20-35 uJy in X-band.

If your image is already decent, it will not do dramatically change the flux values and RMS levels you'll report, though it may visually improve the image. However, if you have a wonky SED, it's worth trying out.

## How auto_selfcal works and why my scripts use it impractically:
Patrick Sheehan's auto_selfcal (https://github.com/psheehan/auto_selfcal) solves for a range of calibration solutions using different solution intervals and applies the best-fit ones to the measurement set. It then creates a final image for you using the self-cal measurement set. It has capability to do a multi-frequency .ms for an SED but it would realistically require running in parallel using mpicasa, which I have found to error on both Talapas and NRAO. Also, I don't believe you can create a split-band image if you self-cal the entire SED unless you then apply the calibrations to create the _final.ms and then image the _final.ms yourself. That is probably a good thing to work out at some point, but it seemed like extra steps. Here's the workaround that I've adapted: split your .ms into the half-bands and then submit separate batch jobs for each. This in theory solves two issues: a) you get automatic _final.image.tt0 images from the auto_selfcal script without having to re-image from the _final.ms and b) you can submit several of these split-band auto_selfcal jobs at the same time, effectively running an entire SED worth in 8-12 parallel jobs. It gets the job done.

## Other notes
* This takes up a LOT of storage space! Running auto_selfcal on an A-config SED will produce about 2 TB of data over the course of the job. Some of this is from tclean's temp lattices that are deleted afterwards, but you still will have to clean up your directory on NRAO afterwards, and you will not be able to run on Talapas if you only have the default storage quota. I have a script to clean things up for you.
* Low-frequencies at A-config take forever to run (sometimes timing out after a week) and produce large (~2GB) _final.* files. I'm yet to establish the trend exactly. Consider this before running auto_selfcal on SEDs taken at A config.

## Getting started with GitHub
To obtain a local copy of this code + its supporting materials, clone the repository in Terminal via


    git clone https://github.com/jlynch2195/auto-image-VLA.git

You only have to do this once. To update your local copy to match the newest version on GitHub, make sure you're in the repository in your filesystem, then run

    git pull

You may get error/warning messages about needing to commit changes before pulling, meaning you've made edits to the existing files and git doesn't want to override those. If you want to keep your version, you can rename yours to avoid overwriting them. There's probably better practice; it's worth a Google.



## Repository contents:
1. prep-ms-for-auto-selfcal.py: Script to prep your measurement set in the current working directory for auto_selfcal. It first splits the main.ms file into main_target.ms. Then it splits the main_target.ms by spectral windows based on the user-specified split_bands command. Finally, it writes batch files to perform auto_selfcal on each of these split measurement sets and saves the batch file paths to a text file. No auto_selfcal will be done by running this script: it just sets up batch scripts.
2. submit_batch_of_batch_jobs.py: Simple script to submit all the batch jobs that prep-ms-for-auto-selfcal.py wrote.
3. analyze-final-images.py: Script to fit for a point source at target location and create a snapshot FITS file. Operates on a list of files but will work with just one. Returns a .csv file of image statistics and a 128x128 snapshot zoomed in at target location.
4. vla-configuration-schedule.csv: table from https://science.nrao.edu/facilities/vla/proposing/configpropdeadlines
5. vla-resolution.csv: table from https://science.nrao.edu/facilities/vla/docs/manuals/oss/performance/resolution


## Get familiar with Talapas
Login with

    ssh user@login.talapas.uoregon.edu

You're automatically in your user directory. You have 500 GB of storage here:

    pwd

Find the Cendes lab project, called nova:

    cd /projects/nova/

See what's in there. Some group members have a folder and there are some github repos as well.

    ls

Make your own directory:
    
    mkdir user 

## Get familiar with nano
The text editor "nano" is how you're going to edit code on Talapas, since there's no graphical user interface like Jupyter: you only have command line access. To try it out, move to your user directory and make a file named test.txt. This will bring up a blank file with lots of keyboard actions on the bottom

    cd user
    nano test.txt

Type whatever you want and save with CTRL X, then y, then ENTER. You can check to see if your changes went through: this will print the contents of test.txt to the terminal:

    cat test.txt

## Get your measurement set to Talapas
Open up a new terminal window on your local machine. It's not required (I don't think), but rename your .ms to be this form: project_code.source_name.obs_date_as_YYYY-MM-DD.ms (ex: 24A-322.ASASSN-14ae.2024-09-17.ms). If anything, it helps make sure you don't overwrite other measurement sets of the same source. Now copy your measurement set to project directory on Talapas

    scp -r ~/path/to/measurement_set.ms user@login.talapas.uoregon.edu:/projects/nova/user/

This should take about 30mins for a 1hr C-band ms file. Once that's done copying, login to Talapas:

    ssh user@login.talapas.uoregon.edu

Check to make sure it make it there

    cd /projects/nova/user
    ls

Check to make sure everything got copied over: I've had issues with the table.dat and co. files not making it and that breaks everything. If you're missing table.dat, you can try to continue, but know that you may get an error. If so, the path of least resistance is to delete the ms on Talapas and re-upload it. If you're also missing table.dat from your local copy, you'll need to re-download the ms from the NRAO archive.

    cd measurement_set.ms
    ls

It's probably not required, but you want the file system to be /projects/nova/user/source_name/main.ms, so set that up via

    cd /projects/nova/user/
    mkdir source_name
    mv measurement_set.ms source_name/measurement_set.ms

## Set up auto-selfcal script
Copy the files over that you need for auto_selfcal, which live in Jimmy's folder under need_for_selfcal:

    cp -r /projects/nova/jimmy/need_for_selfcal/* source_name/

Move into your source_name directory and list the files there with ls. You should see:
measurement_set.ms  prep-ms-for-auto-selfcal.py  submit_batch_of_batch_jobs.py  vla-configuration-schedule.csv  vla-resolution.csv

The only file you need to edit is prep-ms-for-auto-selfcal.py. Copy the name of measurement_set.ms before running and make sure you're in your source_name directory

    nano prep-ms-for-auto-selfcal.py

Now edit the required fields after the imports directly in the file: I removed the config.yaml file because it was too much to keep track of. One argument is "split_band" which can be "whole" to use all the spws for the band to make 1 image, "halves" to use half of the spws to make 2 images, or "both" to do both. For example, for C-band, "whole" makes an image for 6 GHz, "halves" makes 5 and 7 GHz, and "both" makes 5, 6, and 7 GHz. For a single C-band measurement set, I usually just keep it as "whole" to get a 6 GHz image. Arguments use_single_band and single_band should be set if using a single frequency measurement set.
measurement_set = "25A-060.AT2019qiz.2025-06-01.ms"
source_name = "AT2019qiz"
split_band = "whole"
use_single_band = True
single_band = "EVLA_C"

Once done editing, remember how to save: CTRL X, then y, then ENTER. You can check to make sure changes took place if you want:

    cat prep-ms-for-auto-selfcal.py

## Run auto self-cal
Important: before running any code, you need to check out an interactive node. It's bad practice to run code on the login nodes. Do so via

    srun --account=nova --pty --partition=compute --mem=32G bash

You should now see the IP address on the left of your terminal cursor change from [user@login1] to [user@XXXX] which means you're on a compute node.

Load and open up casa

    module load casa/6.7.0
    casa

Side quest: If this is your first time running CASA on Talapas, you may get an error from CASA like

    data_update: path must exist as a directory and it must be owned by the user, path = /home/user/.casa/data

You can fix it via

    mkdir ~/.casa/data

Now run the prep-ms script:

    execfile("prep-ms-for-auto-selfcal.py")

This outputs a batch script that you can then submit by running

    execfile("submit_batch_of_batch_jobs.py")

## More on batch jobs
A batch job is basically an untouchable compute task that will run for some amount of time before either crashing or finishing. I am not an expert on them and there are better practices for how I've set this up. The batch job that you are now running is performing automatic self-calibration on your measurement set. It takes about 5 hours per band. Check the status of your batch job with the below command. It should tell you if it's completed, running, interrupted, etc. You basically pray that it's still running.

    sacct -u user --format=JobID,JobName%50,State,Start,End

If your job fails, you can check the error log and the output log to see if there's any evidence of what went wrong

    cd /projects/nova/user/project/source_name
    cat job_name.err
    cat job_name.out

If it's working okay, auto_selfcal will spit out a bunch of files in your source_name/frequency directory. You can see how far along it is by listing the files in that directory. Once it's completed, the main file we want is .final.image.tt0, which is the final self-cal image. It's helpful to copy the full path to the final image, cuz you'll need it to copy it over to your laptop. I think it makes sense to apply the final calibrations and move the final .selfcal.ms file to external drives, but still need to think on that.

## Get the results back locally
Back on your local machine, open up a terminal window and copy the file over

    scp -r user@login.talapas.uoregon.edu:/projects/nova/user/project/source_name/frequency/something-final.image.tt0 ~/Desktop/or/wherever_you_want_it

Open it up in CARTA and see how it looks. I like to append the non-self_cal image to the left for comparision. 

To get flux measurements and a cropped 128x128 fits image centered at your source location, edit and then run the "analyze-final-images.py" file in this repository via

    cd ~/Desktop/place_where_analyze-final-images.py_is/ 
    casa
    execfile("analyze-final-images.py")
