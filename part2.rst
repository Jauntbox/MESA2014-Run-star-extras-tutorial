Lab 2 - Adding new variables & using ``star_info`` structure
============================================================

Intro
-----
In this section, we'll practice extracting data from the ``star_info`` structure and use it to calculate some auxillary variables. In this activity, you should start by making a fresh copy of the ``mesa/star/work`` directory in your local work space. Name it whatever you want, for example

.. code-block:: bash

   cp -r $MESA_DIR/star/work work_kevin_part2


Exercise 1 - Finding what MESA calculates
-----------------------------------------
The global dynamical time scale of a star is given by

:math:`t_{\rm dyn} \sim (G\left<\rho\right>)^{-1/2}`

where :math:`\left<\rho\right>` is some measure of the mean stellar density. Within prefactors, this is the free-fall time at the stellar surface as well as the Keplerian orbital frequency there. MESA already calculates a dynamical timescale at every cell in the star. How is it defined?

*Search through MESA/star and find where this is calculated. What is the exact formula?*

But what's the name of the variable to search for? That's part of what you're looking for. This may seem like a silly activity, but when you're writing your own code for MESA this type of question comes up over and over so it's good to get some practice hunting things down. 

If you're having trouble, try narrowing your search to (highlight to reveal):

.. raw::  html

	<font color="FFFFFF">mesa/star/private</font>
   
The definition you should find is (highlight to reveal):

.. raw::  html

	<font color="FFFFFF">s% dynamic_timescale = 2*pi*sqrt(radius*radius*radius/(s% cgrav(1)*mstar))</font>
   
which definition of the dynamical time does this prefactor correspond to?

Exercise 2 - Calculating the dynamical time scale(s)
----------------------------------------------------
Now that we know what MESA is calculating, what if we want a slightly different calculation? Perhaps we have a planet with a massive core and just want to use the central density for :math:`\left<\rho\right>`, or instead take a mass-weighted average of the densities of interior cells.

The first step is to tell MESA that you're going to be adding some extra user-defined columns to the profiles. This is done through the function ``how_many_extra_profile_columns()`` simply by making the return value equal to the number of extra columns you're adding. We'll start with just one,

.. code-block:: fortran

   how_many_extra_profile_columns = 1
   
The next step will be to fill in a placeholder for our dynamical timescale calculation, just to check that the code is set up correctly before we start trying to do calculations with the ``star_info`` data. This will go in the subroutine right below, ``data_for_extra_profile_columns()``.

We can modify the ``inlist_project`` file if we want, but the default one will still spit out profiles every 50 steps, so we can leave it as it is. You should get 90 columns by default, so any you add will go at the end.

Center density approach
^^^^^^^^^^^^^^^^^^^^^^^
Let's start out really simple and make a new dynamical timescale based on just the central density of the star. This is a pretty useless value to put in a profile since it's not even spatially-dependent, but we'll start with the easiet thing to code up. Inside ``data_for_extra_profile_columns()``, there's an example calculation in the documentation, we'll start out with a similar scaffold and just set everything equal to zero to check if things still compile.

.. code-block:: fortran

   !We're going to try and calculate some dynamical time scales:
   names(1) = 't_dyn (central)'
   do k = 1, nz
      vals(k,1) = 0d0
   end do
   
Check that this works (you should get a 91st column in your profiles that's filled with all zeroes), and then figure out how to to the calculation with central density, :math:`t_{\rm dyn} = (G\left<\rho_c\right>)^{-1/2}`. One possible solution is (highlight to reveal):

.. raw::  html

	<font color="FFFFFF">vals(k,1) = sqrt(1d0/(s% cgrav(1)*s% rho(nz)))</font>
   
Running this should make all the cells in your new profile column have the same value. For ``profile10.data``, I got :math:`t_{\rm dyn} \approx 6.3\times 10^4` s.

Mass-weighted average density approach
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Another, more realistic, way to estimate the dynamical time would be to use a mass-weighted mean density of the interior cells. This requires for each cell k, looping over the cells interior to k (what indicies are those?) and finding the mass-weighted mean density.

If you get stuck, my code looked like this (highlight):

.. raw::  html

   <font color="FFFFFF">names(2) = 't_dyn (mass-weighted)'</font><br>
   <font color="FFFFFF">do k = 1, nz</font><br>
      &nbsp&nbsp&nbsp&nbsp&nbsp<font color="FFFFFF">mean_density = 0d0</font><br>
      &nbsp&nbsp&nbsp&nbsp&nbsp<font color="FFFFFF">do j = k, nz</font><br>
         &nbsp&nbsp&nbsp&nbsp&nbsp&nbsp&nbsp&nbsp&nbsp&nbsp<font color="FFFFFF">mean_density = mean_density + s% dm(j)*s% rho(j)</font><br>
      &nbsp&nbsp&nbsp&nbsp&nbsp<font color="FFFFFF">end do</font><br>
      &nbsp&nbsp&nbsp&nbsp&nbsp<font color="FFFFFF">mean_density = mean_density/s% m(k)</font><br>
      &nbsp&nbsp&nbsp&nbsp&nbsp<font color="FFFFFF">vals(k,2) = sqrt(1d0/(s% cgrav(1)*mean_density))</font><br>
   <font color="FFFFFF">end do</font>

Remember to declare any new variables you need and to set

.. code-block:: fortran

   how_many_extra_profile_columns = 2
   
You should now have two extra columns, the mass-weighted calculation should be linearly increasing from the core outward and equal to the value of the central density method at the central point since our formulae should be the same there.

Comparison to sound crossing time
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Our last goal is to compare these calculations with the sound-crossing time through the interior of the star, another measure of how fast parts of the star can respond to perturbations. This calculation should be similar to the previous section, except you'll want to approximate the integral

:math:`t_{\rm sound}(r) = 2\int_0^r \frac{dr}{c_s}`

in MESA. There are several ways to calculate this given the variables in the ``star_info`` pointer, give one a try. Again, my solution is below (highlight):

.. raw::  html

   <font color="FFFFFF">names(3) = 't_dyn (sound)'</font><br>
   <font color="FFFFFF">vals(nz,3) = 2*s% r(nz)/s% csound(nz)</font><br>
   <font color="FFFFFF">do k = nz-1, 1, -1</font><br>
      &nbsp&nbsp&nbsp&nbsp&nbsp<font color="FFFFFF">vals(k,3) = vals(k+1,3) + 2*(s% r(k) - s% r(k+1))/s% csound(k)</font><br>
   <font color="FFFFFF">end do</font>
   
How is this different from the other values we calculated? Why do you think this is?