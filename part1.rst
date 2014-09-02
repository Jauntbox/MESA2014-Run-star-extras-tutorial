Lab 1 - Custom stopping/output criteria
=======================================

.. 
	In this part we'll learn how to define more complex stopping/saving criteria using ``run_star_extras.f``

Intro
-----
In the MESA Tutorial, we learned how to add various stopping criteria to MESA. Typically these are of the form "stop when a certain varibale passes a certain threshold". Often this is enough, but sometimes you want more complicated stopping criteria - or maybe you'll want to save a bunch of profiles/models at 100 different times without chaining together 100 different inlists.

Setup
-----
Make a local copy of the work directory, you can name it whatever you want. Go to the location you want to place the copied work directory and type

.. code-block:: bash

	cp -r $MESA_DIR/star/work work_kevin_part1
	
where ``work_kevin_part1`` can be replaced with any name you want to use. You'll then want to check that everything worked fine by compiling and running things with the normal commands

.. code-block:: bash

	./mk
	./rn

*Any time we change the code of the files in the local src directory, we must recompile the code!*


Exercise 1 - Profile output criteria in inlists
-----------------------------------------------
Recall that MESA provides two output file types by default as it runs:

* History files (eg. ``work/LOGS/history.data``) give the global properties of the star as a function of time. This is the file you'd look in if you wanted to make an HR diagram or a luminosity vs. time plot.

* Profiles (eg. ``work/LOGS/profile37.data``) give a spatial slice of the star at a fixed time. This is the type of file you'd look in if you wanted to make a plot of temperature vs. pressure or abundances vs. mass coordinate.

If you want to save a profile through inlist controls (see eg. the ``&controls`` documentation in ``mesa/star/defaults/controls.defaults``), you can save profiles at a given timestep interval, through, eg.

.. code-block:: fortran

	profile_interval = 50

and/or save one on termination with (in the ``&star_job`` section)

.. code-block:: fortran

	write_profile_when_terminate = .true.
	filename_for_profile_when_terminate = 'my_profile_name.data'

If you have some special criteria you want to save profiles on, then you'd need to use a separate inlist for each criterion and save a profile on termination. While this works, it's kind of a pain if you want to run a grid of models or change some parameters of your run since you need to modify all the inlists. If you want to have these same output criteria for a grid of stars, each with slightly different input parameters, then you can put the stopping criteria in ``run_star_extras.f``.

.. 
	This is shown in ``work_kevin_part1.tar``, which contains three inlists which run a :math:`20\ M_\odot` star through the main sequence, saving profiles & models at various core composition cutoffs. 

MESA always reads its parameters from a file named ``inlist``. In the default ``work`` directory we copied, this inlist file just reads from another file named ``inlist_project``. This allows you to have several different inlists in your directory, only needing to change the file that ``inlist`` itself reads from through the lines

.. code-block:: fortran

	extra_star_job_inlist1_name = 'inlist_project'
	extra_controls_inlist1_name = 'inlist_project'
	extra_pgstar_inlist1_name = 'inlist_pgstar'

We're going to run a :math:`20\ M_\odot` star through the main sequence here, and want to save profiles and models at three different abundance criteria: :math:`X_{\rm center} = 0.69`, :math:`X_{\rm center} = 0.40`, and :math:`X_{\rm center} = 0.01`. You can just take the ``inlist_project`` file and copy it into three new inlist files that we'll slightly modify. What all needs to be changed?

.. 
	inlist_project is short - read it!

* stellar mass - now :math:`20\ M_\odot`
* profile output criteria - see above
* stopping criteria - see below

The stopping criteria in the inlists go in the ``&controls`` section and are as follows. You should name your inlists something descriptive to help you keep track of them - the names and stopping criteria I used are as follows:

First inlist (``inlist_to_h_ignition``):

.. code-block:: fortran

	xa_central_lower_limit_species(1) = 'h1'
	xa_central_lower_limit(1) = 0.69

Second inlist (``inlist_to_h_burn``):

.. code-block:: fortran

	xa_central_lower_limit_species(1) = 'h1'
	xa_central_lower_limit(1) = 0.40
	
Third inlist (``inlist_to_h_depletion``):

.. code-block:: fortran

	xa_central_lower_limit_species(1) = 'h1'
	xa_central_lower_limit(1) = 0.01
	
To save time, you should save models at the end of the first and second inlists and load them into the sedond and third inlist, repsectively. To save a model, you can add to the ``&star_job`` section of your inlist (for example)

.. code-block:: fortran

   save_model_when_terminate = .true.
   save_model_filename = 'h_ignition.model'

and to load a model, you can add (also to the ``&star_job`` section)

.. code-block:: fortran

   load_saved_model = .true.
   saved_model_name = 'h_ignition.model'
	
Here, your only task is to verify that these inlists work as intended (i.e. you get three profiles out with the different names you gave them). Enjoy watching the PGSTAR plots!

*Note: There are example shell scripts on how to automate chaining multiple inlists together so that you only need to execute one command to run the whole batch. See, for example, mesa/star/test_suite/make_planets. You still have the inconvenience of having to edit multiple inlists to change parameters.*

.. 
	We're using a slightly modified version of the default work directory that's based on some of the ``test_suite`` examples such as ``make_planets``. The actual executable that gets created when you compile the code is called ``rn1``, which the script that you run is called ``rn``. The ``rn`` script copies each of the inlists in our run sequence into the ``inlist`` file and then runs it.

Exercise 2 - Data output criteria in ``run_star_extras.f``
-------------------------------------------------------------

Profiles
^^^^^^^^
Your next task is to express the profile output criteria in the inlists from the previous section in ``run_star_extras.f``. This way you only need to modify one inlist which makes things much more convenient when changing things such as resolution.

Recall from the ``run_star_extras.f`` tutorial (`see here <http://mesa.sourceforge.net/run_star_extras.html>`_) that you need to replace the include statement in ``src/run_star_extras.f``

.. code-block:: fortran

	include 'standard_run_star_extras.inc'
	
with a copy/paste of the contents of that file (located in ``mesa/star/job``). Check that this works by compiling and running again. There's no need to run to completion - you should just verify that the code still compiles/runs before making further changes. *This is good advice for starting any modification of MESA. As with voting, you should recompile early and often - it will help prevent you having to look through a bunch of changes that all of a sudden aren't compiling!*

Look through the documentation of the provided procedures you copy/pasted into ``run_star_extras.f`` and find which one you should put output/stopping criteria in (when in the time step do you want to check these criteria?).

The recommended option is (highlight below to reveal):

.. raw::  html

	<font color="FFFFFF">extras_finish_step()</font>
	
You can express your output criteria using the ``star_info`` data structure which contains all the info MESA knows about your star. Look in ``mesa/star/public/star_def.f`` for the definition of this data structure. Lots of the variables are included from other files so they're not all listed explicitly in ``star_def.f`` (in particular, most of the variables are actually defined in ``mesa/star/public/star_data.inc``). You need to find the names of the composition variables (there are many options that will work) in order to write output criteria. Throughout MESA's code, you'll see this stucture refered to as ``s``, so if you see something like ``s% mass(k)`` then that just means look inside the structure ``s`` for the array ``mass`` and give me the kth entry.

If you're having trouble finding the right variable to use, try (highlight below to reveal)

.. raw::  html

	<font color="FFFFFF">s% center_h1</font>
	
Once you can express your criteria in if/then statements, you need the subroutines for outputting models/profiles. These are listed in ``mesa/star/public/star_lib.f``, so search there for the right ones. You can call all of these subroutines from ``run_star_extras.f`` because they are already included using the line at the top

.. code-block:: fortran

	use star_lib

If you can't find a suitable subroutine there, try looking at (highlight below to reveal)

.. raw::  html

	<font color="FFFFFF">star_write_profile_info()</font> <br>
	
Note that while the call signatures of these subroutines require you to pass several things to them (including other subroutines!), most of these have the same names in ``run_star_extras.f`` so you shouldn't need to track down additional arguments. The main difference between using this method versus setting ``s% need_to_save_profiles_now = .true.`` is that calling the subroutine allows us to specify the profile's name, while ``s% need_to_save_profiles_now = .true.`` just tells MESA to output a profile using its standard naming conventions (so you'll get a profile named ``profileXX.data`` with whatever profile number you're on).
	
Finally, you need to slightly modify the inlists from before. I'd suggest making a new one (eg. ``inlist_full``) that will run the star through the entire main sequence and output the three profiles at the right points. What's the inlist stopping criteria now? Do you need any profile output statements?

Check that you again produce the same results as you did with the multiple inlists.
	
Models
^^^^^^
You can also save model files from ``run_star_extras.f`` for use as restart points using a similar subroutine. Take a look through ``mesa/star/public/star_lib.f`` to find it.

	
Stopping criteria in ``run_star_extras.f``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Specifying a stopping criterion is the same as what we did in the previous part, except instead of calling subroutines to output profiles or model files, we send MESA/star a command that tells it to stop evolution. Notice at the beginning of the function ``extras_finish_step()``, it is set to return the value

.. code-block:: fortran

	extras_finish_step = keep_going
	
which tells MESA/star to keep calm and carry on evolving the star. If you want to terminate the evolution for some reason, then you can instead set the function to return the value

.. code-block:: fortran

	extras_finish_step = terminate
	
among other possible integer stop codes listed in the ``extras_check_model()`` function located in ``run_star_extras.f``. You can also return termination codes from that function.  Documentation is provided on how to output custom messages for different stopping criteria in that function as well.

Brief overview of PGSTAR
------------------------
Frank's lab will cover making custom PGSTAR plots in more detail, but you'll want some graphical output to stare at while your stars are evolving. There are two things you need to put in your inlist to make sure you have graphical output. In the ``&star_job`` section, you need to enable PGSTAR with the line

.. code-block:: fortran

	pgstar_flag = .true.
	
and in the ``&pgstar`` section, you need to specify what plots you want to see. The grid windows (several plots at once) are the best for starting out since they give you a dense set of info. Try turning on one of these windows with 

.. code-block:: fortran

	Grid1_win_flag = .true.
	
You should also tell PGSTAR not to close the plots as soon as the run is over so you can still see what your star looked like at the end. You can do this with the line (also in ``&pgstar``)

.. code-block:: fortran

	pause = .true.
	
MESA reads the ``&pgstar`` section of the inlist at each time step, so you can add and modify plots virtually in real time. Try it!