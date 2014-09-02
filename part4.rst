Part 4 - ``other_*`` reference page
===================================

Intro
-----

There are quite a few ``other_*`` routines (also referred to as hooks) that MESA provides. Rather than modifying the private code, these are the way to access and modify MESA's internal functions. If you'd like to modify something that you don't see an obvious subroutine for, then Bill will be happy to add a new one for you. Here's a list of some of the more commonly used hooks:

**Please let us know (eg. by email to the list) if something becomes out of date, if you find an error, or if there's another use case you think we should explicitly mention for one of the hooks.**

Hooks
-----

* ``other_mlt()`` - Allows you to change the implementation of mixing length theory (MLT) that MESA uses to determine what type of mixing occurs in a cell. The actual temperature gradient is also calculated here, along with the diffusion coeffienct for mixing, among other related quantities. For details on the expected return variables, see ``mesa/mlt/public/mlt_def.f`` and ``mesa/mlt/public/mlt_lib.f``. Example uses for this hook include defining your own local mixing prescription, modifying the thermal and compositional transport rates for standard convection (or any other mixing type), etc. *Note: Mixing is not actually performed in this subroutine. If you want to modify how MESA mixes things, you need to look at other hooks such as ``other_split_mix()`` or ``other_diffusion()``*

* ``other_energy()`` - Allows you to specify an anomalous heating/cooling rate in the star. This is pretty simple, since it's not replacing a call to another MESA routine like some of the other hooks.

* ``other_wind()`` - Allows you to specify a mass-loss prescription (eg. a formula for :math:`\dot{M}` as a function of other stellar variables). Can also be used for more general mass-loss scenarios like Roche-lobe overflow.

* ``other_eos()`` - Allows you to use your own implementation of an equation of state. An example would be using `FreeEOS <http://freeeos.sourceforge.net>`_ within MESA.

* ``other_kap()`` - Allows you to implement your own routine for calculating opacities. Examples include trying to calculate them on the fly instead of looking them up from tables (still too slow according to those who have tried), as well as forcing things like pure electron conduction or electron scattering in certain regions of a star/planet.

* ``other_torque()`` - Allows you to implement other sources of torque in rotating stars. This could be used, for example, to implement a magnetic breaking prescription.

* ``other_mesh_functions()`` - Allows you to define other functions that MESA will examine when determining when to increase/decrease resolution.

* ``other_diffusion()`` - Allows you to implement your own diffusion routine in between stellar time steps. This bypasses the entire default diffusion solver, so you either need to copy/paste and make appropriate changes or write your own from scratch.

* ``other_atm()`` - Allows you to specify a custom atmosphere boundary condition. Could be useful in trying to contstruct models to match known planets.

* ``other_split_mix()`` - Allows you to modify the composition profiles after the mixing step.