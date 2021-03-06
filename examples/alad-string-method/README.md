
# String Method in Collective Variables -- An Implementation in NAMD v 2.11 using CFACV

This README resides in the alad-string-method subdirectory of github.com/cameronabrams/cfacv/examples/.  The purpose of the readme is to provide an explanation of how to run a string method in collective variables (SMCV) simulation using our CFACV implementation.  The example is a solvated molecule of alanine dipeptide under the CHARMM22 force-field and TIP3P water.  

> **Note:**  It is probably a good idea to try out the earlier CFACV tutorials first:
>
> - `alad-fixed-phi-psi/`: MD with harmonic restraints on ![phi](README_images/phi.png) and ![psi](README_images/psi.png) angles
> - `alad-tamd/`: TAMD where ![z1](README_images/z1.png)
> - `alad-single-sweep/`: single-sweep reconstruction of the free-energy surface ![F1](README_images/F1.png)

It is also assumed the reader has a good working knowledge of MD simulations in general and NAMD in particular.  Because this implementation is based on NAMD's replica exchange feature, it also requires a locally compiled NAMD binary with MPI.

## Background

The current implementation of SMCV is described in Maragliano and Vanden-Eijnden, "On-the-fly string method for minimum free energy paths calculation", *Chemical Physics Letters* 2007;**446**:182-190.  

The objective of a SMCV calculation is to identify a minimum free-energy pathway (MFEP) through a collective variable (CV) space chosen by us.  The multidimensional variable ![z](README_images/z.png) represents a location in CV space, and one can map an all-atom (3![N](README_images/N.png)-dimensional, where ![N](README_images/N.png) is the number of atoms) configuration ![x](README_images/x.png) to a point in CV space using the multidimensional mapping function ![theta_of_x](README_images/theta_of_x.png).  (For alanine dipeptide, the CV space most commonly explored is the 2-D space spanned by the backbone dihedral angles ![phi](README_images/phi.png) and ![psi](README_images/phi.png).)  An MFEP connects two separated, local minima with a curve ![z_of_s](README_images/z_of_s.png) that satisfies 


 ![eqn1](README_images/eqn1.png)

That is, the MFEP is that curve ![z_of_s](README_images/z_of_s.png) whose perpendicular components of the metric-tensor-rotated free-energy gradients locally perpendicular to the curve are zero.  (The metric tensor is required because the mapping functions can in general be curvilinear functions of atomic configuration.) This curve can (again, almost) be found by first setting up an initial curve arbitrarily, and then solving the differential equation


 ![eqn2](README_images/eqn2.png)

Here, ![gamma](README_images/gamma.png) is a friction that we can choose, and the second term on the RHS represents reparameterization of the curve.  It's clear that, in general, one needs to know the tensor function ![tilde_M](README_images/tilde_M.png) and vector function ![nabla_F](README_images/nabla_F.png) in order to solve this equation, but in the context of a CV space ![z](README_images/z.png) these functions are statistical-mechanical averages over underlying configuration space ![x](README_images/x.png); specifically,

 ![eqn3](README_images/eqn3.png)

where ![m_sub_i](README_images/m_sub_i.png) is the mass of configurational variable ![i](README_images/i.png), and

 ![eqn4](README_images/eqn4.png)

Here, the ![z](README_images/z.png) subscripts on the angle brackets denote that one needs to compute these locally in CV space.  Since these are ensemble averages, we of course run MD on an atomistic system to compute them, but to keep them local, we run _restrained_ MD using a harmonic external potential ![harm_extern_pot](README_images/harm_extern_pot.png). Of course, one could _precompute_ ![tilde_M](README_images/tilde_M.png) and ![nablaF](README_images/nabla_F.png) along finite but uniformly spaced set of discrete points spanning CV space, and then find an MFEP by direct explicit integration; however, this is unpractical in most CV spaces.  

In practice, SMCV represents ![z_of_s](README_images/z_of_s.png) as a set of ![R](README_images/R.png) discrete points ![z_sup_p_full](README_images/z_sup_p_full.png), each of which is termed an "image", and which together constitute the "string" representation of the curve ![z_of_s](README_images/z_of_s.png).  In this implementation, we can choose to either use a single companion restrained all-atom MD system for each image ![z_zup_p](README_images/z_sup_p.png) to compute ![tilde_M_of_z_sup_p](README_images/tilde_M_of_z_sup_p.png) and ![nabla_F_of_z_sup_p](README_images/nabla_F_of_z_sup_p.png), or, we can use two systems for the same ![z_sup_p](README_images/z_sup_p.png), one to compute ![tilde_M_of_z_sup_p](README_images/tilde_M_of_z_sup_p.png) and the other to compute ![nabla_F_of_z_sup_p](README_images/nabla_F_of_z_sup_p.png).  The latter of these is the strictly correct way to run SMCV, since these two quantities should be strictly uncorrelated from each other.  In traditional SMCV, one should run long MD at fixed ![z_sup_p](README_images/z_sup_p.png) to compute these, and then a finite update step that solves the string update is applied to give the next set of image ![z_zup_p](README_images/z_sup_p.png) values, and the cycle repeats.  In the ``on-the-fly'' variant, the string updates are chosen to be very small (by setting ![gamma](README_images/gamma.png) to be large) so that updates of ![z_sup_p](README_images/z_sup_p.png) together with ![tilde_M](README_images/tilde_M.png) and ![nabla_F](README_images/nabla_F.png) are computed in lockstep.  (In the implementation here, the number of MD timesteps per string image update for this averaging can be set as low as 1 to implement the OTF version.)  In either case, the MFEP is located when the image updates no longer appreciably move the string.  So, think of a typical SMCV simulation as something like replica-exchange, except that as the replicas are evolving, their "mapping points" in CV-space are moving as a string toward an MFEP.

## Instructions

> **The short version:**
> 1. (one time only) download the CFACV repository and compile `cfacv.so`; note that CFACV has been updated with this release of SMCV, so you should just recompile it.
> 2. For a given SMCV simulation:
>    1. define CV space and set up initial images;
>    2. make the output directory tree;
>    3. choose values of ![kappa](README_images/kappa.png) and ![gamma](README_images/gamma.png), and the number of timesteps between string updates (`steps_per_run`), and edit the appropriate input files;
>    4. create output directory structure, launch SMCV, and monitor for convergence.
>    5. if desired, run a set of restrained MD simulations on the final string to compute the free-energy profile along the string.

### The Long Version:
This follows the basic structure of the NAMD version of replica exchange, on which I based this implementation of SMCV.  File names appear in **bold**, and all directories in `monospace` refer to subdirectories of the CFACV repository.  

#### Get the latest CFACV implementation

Clone this repository to your local machine or whatever machine you'll be running on.  Change directory to `src` and issue the command

`make all`

Reminder that you need the Gnu Scientific Library (gsl) and its headers (gsl-devel) installed.


#### Setting up and running the example SMCV calculation

Now go to the `examples/alad-string-method` directory.  The file **job0.conf** is the master NAMD config file in the command line of the NAMD invocation used to start a string method calculation.  For _this_ calculation, which is able to be run on a single workstation, I provide a bash script **runstring.sh** that shows how to invoke a SMCV calculation.  (**runstring.sh** should be easily adaptable into a PBS or SLURM batch submission script.)  The "0" indicates the first in what could be a sequential set of restarts; i.e., a **job1.conf** is the file one sends to NAMD to restart from the latest checkpoint of the simulation run from **job0.conf**.

**job0.conf** sources **alad_stringmethod.conf**, which sets parameters for the specific calculation, and then sources the general **stringmethod.tcl** (located in the `tcl/` directory of the CFACV repository).    The NAMD invocation in **runstring.sh** shows how to specify the number of images, and the total number of processors to use (which must be a multiple of the number of images).  It is envisioned that one should only need to change specifics in a system-specific **stringmethod.conf** file and **job0.conf** file to start a new SMCV calculation.

For this tutorial, **alad_stringmethod.conf** is fully commented and should be read carefully.  It identifies the base NAMD configuration file **alad_base.namd** that contains MD parameters common to all simulations, including the commands that enable the CFACV `tclforces()` module.  One of _those_ parameters is the spring constant ![kappa](README_images/kappa.png).  Also of note here are the fact that we are requesting 24 MD systems to drive a 12-image string in dual-system (i.e., true on-the-fly) mode, and that the number of steps per run is 1, meaning this is true on-the-fly SMCV. 

Based on what is in **alad_stringmethod.conf**, the input files for each image's MD simulation (including the .coor, .vel, and .xsc files) follow a filename convention of the form **go.job0.#.coor**, where "#" is the integer system number.  IMPORTANT:  We use the following convention for numbering systems.  In the one-system-per-image case, the system number and image number are identical.  In the two-systems-per-image case, an image's first system has a system number identical to the image number, and its second system is offset by the total number of images.  So, if this is a 12-image string, image 0 has system 0 and system 12, image 1 has system 1 and system 13, etc.  In this tutorial, we'll work with 12 images with two systems each, so the "#" is replaced with integers 0...23.  It is important to realize that, in this tutorial, these image-specific input files are generated by running a set of sequential restrained MD simulations.  (In general, there may be many ways to generate an initial string; this is merely one way.)  The shell script **mkinitimages.sh** is provided here to show how these image-specific inputs are generated, beginning from the files **alad_wb.psf** and **alad_wb.pdb**, along with inputs **go_md.coor**, **go_md.vel**, and **go_md.xsc** taken from a long, unrestrained MD simulation.  **mkinitimages.sh** is also well-commented and should be read carefully.  It is what allows one to choose the placement of the initial string in ![parens_phi_psi](README_images/parens_phi_psi.png)-space and how many images are desired.  In this case, you'll see that we have chosen to begin the inital string (i.e., place systems 0 and 12) at ![z_sup_0_of_0](README_images/z_sup_0_of_0.png) = (1.8, -0.5) radians, and to end the initial string (i.e., place systems 11 and 23) at ![z_sup_1_of_0](README_images/z_sup_1_of_0.png) = (-0.25, -1.3) radians.  The interior images are equally spaced along the line connecting these two points.  One can generate a new set of initial image inputs simply by issuing the command:

`./mkinitimages.sh > init.log`

This command can be monitored (as you might suspect) by

`tail -f init.log`

**mkinitimages.sh** runs a bunch of sequential NAMD simulations to set up the images.  Note that here is the first place you encounter the input files **cv.inp** and **label.pdb**.  These are **very, very important files**, as you should have learned in the very first CFACV tutorial.  Briefly, **label.pdb** is a full PDB file congruent to the system that merely serves to label atoms as belonging either to _no_ group (![beta](README_images/beta.png) = 0) or to a specified group (![beta](README_images/beta.png) = 1,2,...).  The center of mass of group ![n](README_images/litn.png) as defined by **label.pdb** is referred to as "center" ![parens_n_1](README_images/parens_n-1.png) in **cv.inp**.  In this tutorial, we define the two CV's as the ![phi](README_images/phi.png) and ![psi](README_images/psi.png) dihedrals; if you examine **label.pdb** and **cv.inp** carefully, you should be able to confirm this for yourself.  Finally, **mkinitimages.sh** generates a separate file for each image with the name **restr.job0.#-1.inp**, where the **#**  is replaced by the image number from 0 to 23.  These files contain the value of the point in CV space each image is initially tethered to.  These are also read in by the SMCV calculation at the beginning.

**alad_stringmethod.conf** also stipulates that image simulation outputs are organized into directories of naming format `output/#/`, where again the `#` is the image number.  To easily generate an empty set of output directories, use the NAMD-provided script **make_output_directories.sh** via

`../../sbin/make_output_directories.sh output 24`

where the first argument is the base subdirectory name for the output and the second is the number of subdirectories of _that_ directory to make, one for each image.

One more thing in the **stringmethod.conf** file is the stipulation of the variables `steps_per_run`, `num_runs`, and `runs_per_frame`.  A "run" in this context is a length of sequential MD timesteps between updates of the string image locations; in true "on-the-fly" SMCV, this is ONE.  The number of runs (`num_runs`) is just how many string method updates one wants to perform.  The set of runs is broken up arbitrarily into sequential sets called "frames" of size `runs_per_frame` runs, and by default, information about current positions, gradients, and values of the metric tensor are output at the end of each frame.  Restart sets are also generated every few frames, as stipulated by the `frames_per_restart` variable.

#### Running the SMCV simulation

Taking stock:  You are ready to launch the SMCV simulation for a system of ![R](README_images/R.png) images provided you have a **stringmethod.conf** file (here, this is **alad_stringmethod.conf**), a **job0.conf** master config file, the **go.job0.0.coor**, **go.job0.1.coor**, **go.job0.2.coor**, etc. (same for **.vel** And **.xsc**) input files, the **restr.job0.#-1.inp** input files that give the anchor positions of the initial images in CV space, a **base.namd** base config file (here, **alad_base.namd**), any parameter files your system requires, and the output directory structure.  Issue the command

`./runstring.sh > run.log &`

You can monitor the progress by 

`tail -f output/0/alad_sm.job0.0.history`

This history file for replica `0` contains three types of output:  gradients, metric tensors, and current image locations in CV space.  It generates one line of output of each type for each image once per frame.   If there are no errors, the simulation will run for a number of MD timesteps given by the product of `steps_per_run` and `num_runs` in the **alad_stringmethod.conf** input file.  So, say for example you have a system of 24 images with 1 step per run and 20,000 runs and 50 runs per frame, you would expect 400 * 3 * 24 = 28,800 lines of output in `output/0/alad_sm.job0.0.history`.  

#### Animating the string evolution

The **mov/** directory contains a shell script and a helper gnuplot script that can be used to make an mpeg animation of the string evolution (note this is only OK to do in a 2-D CV space!).  It uses the free-energy surface for alanine dipeptide computed in the single-sweep example, so make sure you've done that first.  It also uses ffmpeg, so have fun getting that working.  To use this script, simply cd to `output/0` and invoke `../../mov.tcsh`.  This is a still image from this animation:

 ![image1](README_images/string1.png)

#### Monitoring for Convergence

The script **measurermsd.sh** in the `sbin/` directory provides the capability of computing the average "distance" of the string from a reference string, according to 

 ![eqn5](README_images/eqn5.png)

where ![z_sup_i_of_t](README_images/z_sup_i_of_t.png) is the instaneous location of image ![i](README_images/i.png) in CV space at time ![t](README_images/t.png), ![z_sup_i_of_0](README_images/z_sup_i_of_0.png) is a reference position of image ![i](README_images/i.png) (which here is the position after the **latest** update), and as before ![R](README_images/R.png) is the number of images.  Anytime during the run, invoke this as

`../../sbin/measurermsd.sh`

This will assume that the name of the string method config file is **alad_stringmethod.conf** unless you specify another one using the "-smc" switch.  It will also assume the history file is **output/0/alad_sm.job0.0.history** unless you specify another one with the "-smh" switch.  It will also read the **cv.inp** file to determine the dimensionality of CV space.  It will generate an output file **rmsd.dat** in which each line has the frame number and the RMSD at that point in time, relative to the latest string.
The bash script **rmsd.sh** will produce a plot of the rmsd vs iteration number that looks like this:

 ![rmsd](README_images/rmsd-crop.png)

Notice how it ends at zero RMSD by construction, since it is measuring RMSD relative to the latest string position; invoking this after the run has completed simply uses the final interation position.  This data shows that the string is converged to the MFEP well before even 10,000 iterations is reached; the 0.1 RMSD represents thermal fluctuations.

#### Computing the free-energy profile along the string

Computing the free energy profile along the string requires running several restrained MD simulations and tallying the free energy gradients and metric tensors.
This is handled by the script **measurefep.sh** in the `sbin/` directory.
Note that in order to run this script, you need to compile two helper C programs: `src/ForcesFromLog.c` and `src/FEPFromForces.c`.  
If you issued a `make all` in the `src/` directory, these should have been compiled and their executables should exist in the `bin/` directory.   **measurefep.sh** is extensively commented and should be read carefully.
The default values for the variables it needs conform to the alanine dipeptide tutorial, so just issuing it as `../../sbin/measurefep.sh` should launch the MD simulations in series, resulting finally in the file **fep.sh**.  The file **fep.sh** is a bash script that will plot the result, which will look like this:

 ![fep](README_images/fep-crop.png)

This curve is a measure of the free energy along the string shown in the figure above.  Looks like it matches pretty close to what single-sweep says the free energy should be!  

## Conclusion

That is the end of the tutorial -- but look for more updates in the future.  The next update will feature the ability to restart a SMCV calculation from an intermediate iteration.

