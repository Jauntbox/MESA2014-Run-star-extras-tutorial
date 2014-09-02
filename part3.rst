Lab 3 - Specifying custom physics with ``other_*``
==================================================

Intro
-----
In this part, we'll look at how to modify how MESA evolves stars (eg. the equations it solves, the micro/macrophysics it employs, etc.). There are many 'hooks' you have access to in order to change how MESA's internals function, and these are the ``other_*`` subroutines (see ``mesa/star/other`` for all of them). Each subroutine broadly lets you overwrite a subroutine that calculates some piece of physics for MESA/star.

The hook subroutine we'll use here is ``other_mlt()``, which MESA uses as a helper method when solving for the structure of a star. It's called on each cell of the star during each Newton iteration and its job is to determine the mixing type of the cell and figure out its temperature gradient, given the local conditions. Again, we'll be starting with a fresh copy of the ``work`` directory, which you can make via

.. code-block:: bash

	cp -r $MESA_DIR/star/work work_kevin_part3

Context for this code
---------------------
*This section is for your own info - not strictly necessary for the activities*

Most of the code you'll modify through these hooks is far removed from the actual evolution step, located in ``mesa/star/private/evolve.f``. The results you calculate are often passed through several intermediate wrapper routines before they're attached to actual variables in the ``star_info`` structure. The MLT module is no different, and we'll briefly trace the subroutine calls in this section. **Note: In your own research, you'll want to follow similar calls to whatever hook you're using. It's very important to know where your code gets called in relation to the overall evolve step to ensure that your modifications work as you intend. The following is presented as an example outline of how to trace the calls in MESA/star.**

To get some more context of what the MLT module does during evolution, look for where ``mlt_eval()`` is called in the private code. You should only see it called once - it's immediately put inside a wrapper called ``do1_mlt_eval()``, defined in ``mesa/star/private/mlt_info.f``. 

So what does ``do1_mlt_eval()`` do? Aha! That's the subroutine that checks for a user-defined subroutine if we set ``use_other_mlt = .true.`` in the inlist. ``do1_mlt_eval()`` is in turn called from ``do1_mlt()``, which finally attaches the results of the MLT calculation to the corresponding stellar variables. ``do1_mlt()`` then is called from ``set_mlt_vars()``, which loops over a range of cells of interest and sets the MLT variables for each cell. We now know where our changes will show up in ``mlt_info.f``, and what subroutines MESA will use to interface with this.

Where does ``set_mlt_vars()`` appear in the overall structure of the evolution? It is only called from the ``set_hydro_vars()`` subroutine located in ``mesa/star/private/hydro_vars.f``. ``set_hydro_vars()`` is called in two main places:

1. During the Newton solve

Here, the calls initial come from subroutines in ``mesa/star/private/hydro_mtx.f``, which contains subroutines that provide the implementation of various subroutines that get passed to the actual Newton solver. ``set_hydro_vars()`` is called in ``set_vars_for_solver()``, which is wrapped by ``set_newton_vars()``. This is then called from ``eval_equations()`` in ``mesa/star/private/hydro_newton_procs.f`` which is passed to the Newton solver though the ``setequ()`` subroutine in ``mesa/star/private/star_newton.f``. The actual Newton solve happens in the ``do_newton()`` subroutine which is wrapped by ``newton()``. The Newton solve is called from ``hydro_newton_step()`` inside ``mesa/star/private/solve_hydro.f`` through the wrapper ``newt()``. ``hydro_newton_step()`` is called from ``do_hydro_newton()`` which is called from the function ``do_hydro_converge()``. ``do_hydro_converge()`` is then called from ``do_struct_burn_mix()`` in ``mesa/star/private/struct_burn_mix.f``, which is finally called during the top-level ``do_evolve_step()`` subroutine located in ``evolve.f``.

2. After the Newton solve, during the main evolve step

In this part, ``set_hydro_vars()`` is called by the ``update_vars()``, which is called by ``set_some_vars()``, which is called by two top-level subroutines in ``mesa/star/private/hydro_vars.f``, ``set_vars()`` and ``set_final_vars()`` (oh my). Finally, ``set_vars()`` and ``set_final_vars()`` are called in the top-level ``do_evolve_step()`` subroutine located in ``evolve.f`` (``set_vars()`` is called through the wrapper ``do_set_vars()``, since its functionality changes slightly before/after element diffusion). Specifically, ``do_set_vars()`` is called during the implicit :math:`\dot{M}` loop, and ``set_final_vars()`` is called near the end of ``do_evolve_step()``.

So in summary - after 4 levels of calls, our MLT calculations in ``run_star_extras.f`` bubble up to the subroutine ``set_hydro_vars()``, which is called during every step of the Newton iteration (another 9 levels of calls below the actual Newton solve in ``do_evolve_step()``) as well as at the end of each call of each call of ``do_evolve_step()`` (via ``set_final_vars()``).

Exercise 1 - Enabling other_mlt()
---------------------------------
Following the ``run_star_extras.f`` tutorial on the MESA homepage, there are a few steps in telling MESA to use ``other_mlt()``. 

1. The first step is to copy/paste the contents of ``mesa/star/job/standard_run_star_extras.inc`` into your local ``work_kevin_part3/src/run_star_extras.f`` file. 

1. The first step is to copy/paste the ``null_other_mlt()`` subroutine into ``run_star_extras.f``, and rename it to something like ``my_other_mlt()``.

2. Next, in the ``extras_controls`` subroutine of ``run_star_extras.f``, you need to say you want MLT calls to go though this new subroutine by setting

.. code-block:: fortran

	s% other_mlt => my_other_mlt
   
3. Finally, you also need to enable this new subroutine with the ``inlist`` command

.. code-block:: fortran

	use_other_mlt = .true.
   
in the ``&controls`` section. While it may seem redundant, this inlist flag is there so you can turn your custom implementation on/off easily without having to recompile things.

Try everything out by compiling and running things as you normally would. Nothing should change in the output yet when you toggle the ``use_other_mlt`` flag since ``my_other_mlt()`` is still calling the normal MLT subroutine. It's good practice to put a write statement in there to make absolutely sure it's getting called - also so you can see how often it's being called since we have some expectation from tracing the subroutine calls above. *Note: you should be able to get an idea of how MESA is splitting the MLT calls among the different OpenMP threads from this output if your environment variable ``OMP_NUM_THREADS`` is > 1. Try it out if you can!* 

Once you've verified that things are working as intended, you can comment out the output statements if you want since they really slow things down.

Exercise 2 - Modifying the standard convection prescription outside mlt_eval()
------------------------------------------------------------------------------
MESA includes a variety of MLT calculations (see eg. Cox and Giuli's Principles of Stellar Structure for the gory details) that relate the local chemical/thermodynamic conditions to the temperature gradient and compositional mixing rate. 

Some aspects of this calculation are easier to change in the MESA implementation than others. We'll go through some of simpler cases first before tackling a more involved one in the next exercise. The first thing we'll try is to slightly increase the value for :math:`\nabla \equiv \partial \log T/\partial \log P` that ``mlt_eval()`` returns. 

Look at the call signature of ``mlt_eval()`` as well as its implementation in ``mesa/mlt/public/mlt_lib.f`` along with variables in ``mesa/mlt/public/mlt_def.f``. Figure out where :math:`\nabla` gets returned and modify it after the ``mlt_eval()`` so that it gets increased/decreased by a fixed percentage. Start by making it 0.01% larger and increase from there. How much can you increase :math:`\nabla` before MESA has trouble converging. What do you think may be causing problems? 

You can also modify the chemical diffusion coefficent returned by ``mlt_eval()`` to adjust the speed at which convecttion smooths out chemical gradients. There is an inlist control for this in the ``&controls`` section,

.. code-block:: fortran

   ! mixing coefficients are multiplied by this factor
   mix_factor = 1

which scales all of the mixing coefficients (eg. convection, thermohaline, semiconvection, rotational mixing, etc.). While the specific rotational mixing components can be multiplied by specific factors (see ``mesa/star/defaults/controls.defaults``), you have to go into the code if you want to modify the mixing rate of, say, convection alone. 

Modify the variables returned by ``mlt_eval()`` so that you can scale the diffusion coefficient by a constant factor. You may want to change the stopping criterion to be somewhere past the ZAMS so you can see a difference in the evolution. *In practive, perhaps you'd want to make this scaling a function of position, or thermodynamic conditions - hopefully you can see how this could be done (you don't need to actually do something like this here though).*

Exercise 3 - Adding custom inlist controls
------------------------------------------
For the modifications we just made, it would be convenient if there was some way to toggle them or control their strength from the inlist. We can use additional controls, defined through the following variables (see ``mesa/star/defaults/controls.defaults``)

.. code-block:: fortran

      ! extra params as a convenience for developing new features
      ! note: the parameter `num_x_ctrls` is defined in `star_def.f`

      x_ctrl(1:num_x_ctrls) = 0d0
      x_integer_ctrl(1:num_x_ctrls) = 0
      x_logical_ctrl(1:num_x_ctrls) = .false.
      
What is ``num_x_ctrls`` set to by default? Can you change this in the inlist? How do you refer to these parameters through the ``star_info`` pointer, ``s``?

Once you've answered those questions, try adding some sensible inlist controls to the modifications you just made. As an example, one could be toggling the :math:`\nabla` increaese on/off, or adjusting its strength. Check that such things work before moving onto the next part.

Exercise 4 - Modifying the standard convection prescription inside mlt_eval()
-----------------------------------------------------------------------------
Here, we'll slightly modify the MLT calculation to make the mixing length scale with temperature scale height instead of pressure scale height. This part is much more involved, so take your time.

From looking at the call signature of ``mlt_eval()``, it doesn't get the scale height passed to it (just a logical flag ``alt_scale_height`` and the ``mixing_length_alpha`` parameter that multiplies the scale height calculated inside). That means we have to actually go into the internals to change the definition of the scale height used in the MLT calculation.

A brief outline of how to go about this:

1. Since we want to change how ``mlt_eval()`` functions without touching the private code, we should copy the entire subroutine into our ``run_star_extras.f`` file. Search through ``mesa/mlt`` to find where it is defined, you should find it in (highlight to reveal):

.. raw::  html

	<font color="FFFFFF">mesa/mlt/public/mlt_lib.f</font>
   
Unfortunately, it looks like ``mlt_eval()`` is just a wrapper for the private function ``do_mlt_eval()``, which we can again search for, finding it in (highlight to reveal):

.. raw::  html

	<font color="FFFFFF">mesa/mlt/private/mlt.f</font>
   
The actual code we want to modify is ``do_mlt_eval()``, and the various supporting methods it calls. You should therfore copy/paste all the subroutines and functions in this file into your ``run_star_extras.f``. Once they're in there, then they should replace the call to MESA's public subroutine ``mlt_eval()`` with a call to your copy/pasted local implementation, ``do_mlt_eval()``. You should comment out the 

.. code-block:: fortran

   use mlt_lib, only: mlt_eval
   use mlt_def

lines in ``my_other_mlt()`` to make sure that it looks in this file for the modified subroutines instead of using the MESA defaults.

2. In order to compile this new code, you'll have include the MLT module at the top of ``run_star_extras.f``. Along with the other ``use`` statements, add the others that the copy/pasted subroutines are expecting,

.. code-block:: fortran

      use mlt_def
      use mlt_lib
      use crlibm_lib

You will also need to add the module variable (can put it right between the ``implicit none`` and ``contains`` statements)

.. code-block:: fortran

   integer, parameter :: nvbs = num_mlt_partials
   
Finally, you also need to remove all the ``#ifdef`` blocks to get things to compile with the default makefile.
   
3. Once you've copy/pasted the private implementation and made the changes above, check that everything still works as it should by compiling and running the code.  Make sure all the functions required by ``do_mlt_eval()`` are now in ``run_star_extras.f``.

4. Now we can finally start making modifications to the convection routine! In the interest of time, the subroutine that actually calculated the MLT results for convection is ``standard_scheme()``, while the scale height is define in ``Get_results()``. Search for all the places where the scale height is used and instead of using the pressure scale height, use the temperature scale height. 

*How do you calculate the temperature scale height here? The star_info pointer doesn't get passed here!* One way is to extract the ``star_info`` pointer from the ``id`` variable in ``my_other_mlt()`` with the ``star_ptr()`` subroutine (located in ``mesa/star/public/star_def.f``) that fills in a ``star_info`` pointer that you have to declare (``s`` here) via the ``id`` variable.

.. code-block:: fortran

   call star_ptr(id, s, ierr)
   
You can then extract the data necessary to calculate the temperature scale height and pass it down to ``Get_results()``. You have to be careful here and pass it as an argument rather than making it a module variable since there is a different value for every cell and the ``my_other_mlt()`` subroutine will be called in parallel (as you saw in Exercise 1). You may be able to get away with making it a module variable using the OpenMP directive,

.. code-block:: fortran

   !$OMP THREADPRIVATE(temperature_scale_height)
   
but I'm not sure if that would prevent all the possible problems, and that's way beyond the scope of this exercise (did anyone actually get this far anyway??).

One last important tip is when you calculate things from ``star_info`` in these subroutines, you can't always count on the arrays being allocated and filled with data (especially while the star is evolving on the pre-MS). If you want to do a calculation in ``my_other_mlt()`` using the ``star_info`` pointer ``s`` you extracted, then you need to check that the arrays are actually populated. One way to do this (not sure if it's the best, but it has worked for me) is to wrap all your calculations in an if-statement such as

.. code-block:: fortran

   if(k.gt.0) then
      !Calculate things using arrays like s% P(k) without running into segfaults
   endif

After all that effort, can you think of some stellar evolution contexts where this change would make a significant difference?


Defining your own mixing prescription
-------------------------------------

You can also write your own mixing routine, based on what you see calculated in other subroutines such as ``standard_scheme()``, ``set_thermo_haline()``, or ``semiconvection()``. Notice how all of these are called in similar ways from ``Get_results()``. They all set a few variables:

1. ``mixing_type`` - an integer specifying what kind of mixing is occuring in this cell. See ``mesa/mlt/public/mlt.def`` for their definition.

2. ``gradT`` - the actual temperature gradient determined for this cell, :math:`\nabla`.

3. ``D`` - the compositional diffusion coefficient which controls the speed of chemical mixing.

4. ``conv_vel`` - a characteristic velocity of this type of mixing, usually set to 3D/(mixing length) in an isotropic process (cf. Fick's Law).

5. ``d_gradT_dvb`` - partial derivatives of ``gradT`` with respect to all the other MLT variables (see the call structure of ``Get_results()`` and compare to what's passed to it in ``do_mlt_eval()``) .

According to Bill (see `this thread <http://sourceforge.net/p/mesa/mailman/message/32175061/>`_), the only partial derivative you explicitly need to calculate is that of ``gradT`` - the others are already done for you in ``Get_results()``.

We won't have time to implement a new mixing prescription here, but you will in Pascale's lab - have fun!
