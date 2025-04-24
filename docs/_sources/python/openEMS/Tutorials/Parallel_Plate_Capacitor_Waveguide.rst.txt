Parallel-Plate Capacitor and Waveguide
=======================================

This tutorial covers
^^^^^^^^^^^^^^^^^^^^^^

* **Model**: Learn how to set up the components of a basic 3D FDTD model, including:

    * Create conductors in the simulation box.
    * Create a Cartesian mesh (Yee cells) and correctly apply the 1/3-2/3 rule to
      ensure accuracy.
    * Inspect the finished 3D model with AppCSXCAD.

* **Boundary Conditions**: Understand the available boundary conditions in openEMS and
  their physical interpretation.

* **Ports and Excitations.** Add an excitation port with signal. Understand the purpose
  of ports and the existence of different port types. Recognize a Gaussian pulse's
  waveform, frequency spectrum, and its advantages in FDTD simulations.

* **Simulate**: Run simulation to obtain the S-parameters (frequency response) of
  the waveguide. Understand the meanings of convergence, divergence, and "blow-ups".

* **Built-in Post-Processing**: Calculate the final frequency response (S-parameters)
  and time-domain signal waveforms.

     * Plot S-parameters, Z-parameters, time-domain waveforms via matplotlib.

     * Save the raw electromagnetic field dump.

* **Third-Party Post-Processing**: Know the common tools for data analysis.

     * Understand that S-parameters are the parameters that characterize a linear
       circuit's frequency response as a black box without knowledge of the underlying
       physics. The physics is open to interpretation.

     * Plot S-parameters on Smith, line charts, and calculate Time-Domain Reflectometry
       via ``scikit-rf``.

     * Simulate the waveguide's time-domain response and eye diagrams via the Python
       package :program:`SignalIntegrity`

     * Plot the S-parameters using :program:`Qucs-S`, compare our results with an ideal
       microstrip transmission line, and with experimental data.

     * Visualize electromagnetic fields using ParaView.

* **Coding Standard**: Refactor the simulation code with good programming practice
  in mind, by keeping 3D modeling, simulation, and post-processing as separate
  functions. This ensures that different simulations can be ran with different
  parameters in the future.

Problem
^^^^^^^^

A rectangular parallel-plate capacitor in a vacuum is a staple of
physics classes, but the physics involved is far more complex than
an ideal capacitor in introductory textbooks. In the RF/microwave
regime, the signal's wavelength becomes comparable to or even
smaller than the physical size of the capacitor. The plates
form a two-conductor transmission line, known as a *parallel-plate
waveguide*.

However, the ideal parallel-plate waveguide is just a simplified model,
much like the ideal parallel-plate capacitor. A real, finite-size
waveguide still exhibits a fringe field, typically neglected in
theoretical treatments. The large plate separation further complicates
the situation; it's easy for the signal to escape from the plates
because of radiation loss. This is especially problematic when a signal
is injected by a bare wire without a smooth transition - strong
reflections occur right at the connector before the signal can even
enter the structure.

Multimoding is another challenge - in addition to the familiar signal
mode that travels along two-conductor cables (TEM wave), the non-uniform
electric field across the plate may allow TE/TM-mode waves to propagate,
which are more commonly associated with enclosed, single-conductor
waveguides.

Given these complexities, a 3D full-wave field solver like openEMS
is the most practical way to simulate the behavior of a real,
non-ideal parallel-plate waveguide. It does so by numerically
solving Maxwell's equations, the first principles of electromagnetism.

Our task is to find the frequency response of the rectangular
parallel-plate waveguide shown below. It's formed by two metal
sheets each measuring 100 mm x 100 mm, assumed perfect without
resistance or thickness, separated by 16 mm of vacuum.

The 2D cross-section of the 3D waveguide is as follows:

.. image:: images/Parallel_Plate_Capacitor_Waveguide/capacitor1.py.svg

Create Conductors
^^^^^^^^^^^^^^^^^^^

An openEMS simulation always starts by creating a 3D model of the Device-Under-Test
(DUT) using the CSXCAD library. We import the library in Python, and create an empty
structure called ``csx``::

    import CSXCAD

    csx = CSXCAD.ContinuousStructure()

The next step is to define a material type by creating an material instance,
which can then be used to build an object. Since we're modeling metal plates
with no resistance, we use the CSXCAD function
:meth:`~CSXCAD.ContinuousStructure.AddMetal` to obtain an instance of a Perfect
Electric Conductor (PEC)::

    # Create an instance of material named "plate".
    # AddMetal() creates a Perfect Electric Conductor.
    metal = csx.AddMetal('plate')

.. note::
   Internally, PEC is implemented by forcing the tangential electric field in this
   region to be zero, which is characteristic of an ideal conductor that can't be penetrated
   by electric field lines. If resistive losses are unimportant, one can use PEC
   rather than a realistic material model for simplicity and efficiency.

   To add a general material (defined by a permittivity, permeability, electric
   conductivity, magnetic conductivity) such as dielectrics, use
   :meth:`~CSXCAD.ContinuousStructure.AddMaterial`. In principle, it works for
   3D metals as well. But thin metal sheets are challenging in FDTD: to capture
   effects like surface current (skin effect) requires an impractically high
   resolution mesh. Thus, it's recommended to use
   :meth:`~CSXCAD.ContinuousStructure.AddConductingSheet` which mimics the
   resistive loss in metals using a simplified model that still treats metals
   as zero-thickness 2D planes, modeling the observed loss rather than the full
   physics.

In the next step, we build a box using our material "metal", using the
:meth:`~CSXCAD.ContinuousStructure.AddBox` function. It accepts two 3D
coordinates to specify the region occupied by the box.

Our waveguide plates are 100 mm x 100 mm rectangular metal sheets, separated by a
16 mm vacuum (i.e. empty space). Following the common CAD conventions, we center
the plates around the origin. Thus, for the upper plate, both its X and Y
coordinates are (-50, 50). Their Z coordinates are -8 and 8 respectively, which
means these metal plates have no thickness::

    # Build two 3D shapes from -50 to 50 on the X/Y axes, located at Z = -8
    # and Z = 8 respectively. Note that the starting and stopping Z coordinates
    # of each plate are the same, resulting in metal plates with zero thickness.
    metal.AddBox(start=[-50, -50, -8], stop=[50, 50, -8])  # lower plate
    metal.AddBox(start=[-50, -50,  8], stop=[50, 50,  8])  # upper plate

Now it's a good time to take a look at the structure that we've just created
by saving the CSXCAD structure as a file. It's best to keep a call to
:meth:`~CSXCAD.ContinuousStructure.Write2XML` always at the bottom of a script
during development, so that we can readily review the model during development::

    import sys
    import pathlib

    # find the directory of the script itself
    filepath = pathlib.Path(__file__)

    # create a directory named after the script, but without ".py"
    simdir = filepath.with_suffix("")
    simdir.mkdir(parents=True, exist_ok=True)

    # find the filename of the script itself, and replace ".py" with ".xml"
    xmlname = filepath.with_suffix(".xml").name

    # concat two paths
    xmlpath = simdir / xmlname

    csx.Write2XML(str(xmlpath))  # convert Path object to string

Once saved by executing our Python script, one can open the file via AppCSXCAD
to visualize the model::

    $ AppCSXCAD Parallel_Plate_Capacitor_Waveguide.xml

.. important::
   If you're using Wayland, AppCSXCAD may not work correctly, see :ref:`wayland`.

One can inspect the model by mouse drags, looking at it from different camera
angles.
Here's the XY and YZ cross-sections of the correct 3D model:

.. image:: images/Parallel_Plate_Capacitor_Waveguide/capacitor_xy_nomesh.png
   :width: 49%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/capacitor_yz_nomesh.png
   :width: 49%

Requirements for the Mesh
^^^^^^^^^^^^^^^^^^^^^^^^^^^

At this point, the 3D object itself is materialized. In the next step, we
create a mesh to partition the continuous 3D box into discrete rectangular
cuboids. In the Finite-Difference Time-Domain (FDTD) algorithm, these
cuboids discretize the 3D space and form the basic unit of electromagnetic
calculations. These are known as Yee cells (named after Kane S. Yee's 1966
algorithm).

General Requirements
"""""""""""""""""""""

In general, the mesh must satisfy four requirements:

#. **Frequency Resolution**. Its interval must be small enough to resolve the
   shortest wavelength (highest
   frequency component) of the signal, so that electromagnetic field details are
   not missed. Thus, we need several cells per wavelength.

#. **Spatial Resolution**. Its interval must be small enough to resolve the
   shapes of the simulated
   structure, so that small details of the structure are not missed. Thus, we
   need at least a few cells around the important shapes (such as the waveguide
   plates) of the structure. openEMS uses a rectilinear mesh with variable
   spacing. To save time, only use a fine mesh interval around details on
   a structure; use a coarse mesh for the rest.

#. **Smoothness**. Its interval should change smoothly, not by a sudden jump.
   It's recommended that the mesh spacing vary by no more than a factor of
   1.5 between adjacent lines.

#. **1/3-2/3 Rule**. The edges of a conductor have singularities with strong
   electric field. To improve FDTD accuracy, ideally, around the metal edge,
   the metal should occupy 1/3 of a cell, while the vacuum or insulator
   occupies 2/3 of a cell. More on that later.

As a rule of thumb, we want a mesh resolution of at least:

.. math::

   l_\mathrm{cell} \le \dfrac{1}{10} \lambda

.. note::
   This is the same "1/10 wavelength" rule of thumb used for
   determining whether transmission line effects are significant in a
   distributed circuit.

The relationship between wavelength and frequency is:

.. math::

   \lambda = \frac{v}{f_\mathrm{max}}

in which :math:`\lambda` is the wavelength of the electromagnetic wave
in meters, :math:`v` is the speed of light in a medium, in meters per second,
and :math:`f_\mathrm{max}` is the highest frequency component of the signals
used in simulation.

The speed of light in a medium is given by:

.. math::

   v = \frac{c_0}{\sqrt{\epsilon_r\mu_r}}

in which :math:`c_0` is the speed of light in vacuum, :math:`\epsilon_r`
is the relative permittivity of the medium (in engineering, it's sometimes
also denoted as a material's dielectric constant :math:`D_k = \epsilon_r`).
The :math:`\mu_r` term is the medium's relative permeability, which is
usually omitted in engineering for the purpose of signal transmission.
Most dielectrics (like plastics or fiberglass) in cables are non-magnetic -
unless one is dealing with inductors or circulators.

In vacuum, :math:`\epsilon_r = 1` and :math:`\mu_r = 1` exactly.

Courant-Friedrichs-Lewy (CFL) Criterion
""""""""""""""""""""""""""""""""""""""""

In general, the CFL criterion governs the stability of FDTD
calculations. It states that the simulation timestep can't be
larger than:

.. math::
  \Delta t \le
  \frac{1}{v\sqrt{
  {1 \over {\Delta x^2}} + {1 \over {\Delta y^2}} + {1 \over {\Delta z^2}}
  }}

where :math:`v` is the wave speed, :math:`\Delta x`,
:math:`\Delta y`, :math:`\Delta z` are the distances between mesh
lines.

This creates a peculiar limitation: to resolve the smallest mesh
cell, one must use a small timestep regardless of frequency.

Even for simulations at low frequencies, as long as the simulated
structure has small features (i.e. mesh distance is short), we
must use very small timesteps, advancing the simulation only a few
nanoseconds per iteration. Under 100 MHz, at the bottom of the VHF
band or in the HF band, the required number of iterations becomes
impractically large. This means openEMS (and other textbook
FDTD solvers with explicit timestepping) is unsuitable if there's a
large mismatch between the signal wavelength and the physical size
of the structure, such as a 10 cm circuit board operating at 1
MHz, where the wavelength is 300 m in vacuum.

.. note::
   In openEMS, you only need to define an appropriate mesh.
   There's no need to calculate the required timestep size,
   By default, openEMS uses a modified timestep criterion
   named *Rennings2*, not CFL. *Rennings2* improves timestep
   selection in non-uniform meshes, but the general limitation
   (explained using CFL) still applies. `Rennings2` is derived
   in an unpublished paper [11]_,
   see :meth:`~openEMS.openEMS.SetTimeStepMethod` for details.

   If you really need small cells
   (e.g. to resolve some important feature of your structure) you
   will have to live with long execution times, or perhaps FDTD is
   not the right method for your problem. As a workaround, one may
   also try extracting the circuit parameters using a higher signal
   frequency, then using those equivalent-circuit parameters in a
   low-frequency simulation with a general-purpose linear circuit
   simulator.

Create a Mesh
^^^^^^^^^^^^^^

Obtain the Mesh Object
"""""""""""""""""""""""""

We obtain the ``mesh`` object by calling :meth:`~CSX.ContinuousStructure.GetGrid`
to manipulate it further, and set its unit of measurement to 1 mm (``1e-3`` meters).
This will be used as the base unit of all coordinates::

    mesh = csx.GetGrid()
    unit = 1e-3
    mesh.SetDeltaUnit(unit)

Decide the Upper Frequency and Mesh Resolution
""""""""""""""""""""""""""""""""""""""""""""""

Let's pick an arbitrary lower frequency of 100 MHz and an upper frequency
of 10 GHz. Based on the previous analysis, one can calculate the desired
mesh resolution via the following code::

    import math
    from openEMS.physical_constants import C0

    f_min = 100e6  # post-processing only, not used in simulation
    f_max = 10e9   # determines mesh size and excitation signal bandwidth
    epsilon_r = 1
    v = C0 / math.sqrt(epsilon_r)
    wavelength = v / f_max / unit  # convert to millimeters
    res = wavelength / 10

A Simple But Flawed Mesh
""""""""""""""""""""""""""

Now, the most straightforward way of meshing the model is to draw some lines
in the [-50, 50] range on the X and Y axes, and some lines in the [-8, 8] range
on the Z axis.  This is best done by drawing two lines at the beginning and
end of each axis, using the :meth:`~CSX.ContinuousStructure.AddLine` method::

    mesh.AddLine('x', [-50, 50])  # two lines at -50, 50
    mesh.AddLine('y', [-50, 50])  # two lines at -50, 50
    mesh.AddLine('z', [-8, 8])    # two lines at -8, 8

Once we have two lines per axis, one can ask CSXCAD to automatically insert
additional lines by interpolating between existing lines, smoothing the final
mesh. This feature provided by the function
:meth:`~CSXCAD.CSRectGrid.CSRectGrid.SmoothMeshLines`. Its second argument
is the minimum spacing between the lines, here we set it to ``res``. This is the
desired mesh resolution we've just calculated::

    mesh.SmoothMeshLines('x', res)
    mesh.SmoothMeshLines('y', res)
    mesh.SmoothMeshLines('z', res)

Now is a good time to rerun the script and inspect the 3D model again in AppCSXCAD.

.. hint::
   You may be unable to see the mesh lines in AppCSXCAD, as they may be blocked by the
   object due to their overlaps. If it happens, it's necessary to view the model from
   an angle or a different direction (e.g. if there are no mesh lines when viewed from
   the top, try dragging the model to view it from an oblique angle). It's also useful
   to change the "Grid opacity" slider to the maximum (but it still requires viewing from
   an angle).

   AppCSXCAD only renders 2D and 3D cells, if only one axis has mesh lines, no lines
   will be displayed.

   .. image:: images/Parallel_Plate_Capacitor_Waveguide/appcsxcad-opacity-slider.png

Our 3D model's XY and YZ cross-sections are:

.. image:: images/Parallel_Plate_Capacitor_Waveguide/capacitor_xy_naivemesh.png
   :width: 49%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/capacitor_yz_naivemesh.png
   :width: 49%

A Practical Mesh
"""""""""""""""""

Unfortunately, using the mesh as previously defined in a simulation will
produce incorrect results due to several problems.

Simulation Box Size
'''''''''''''''''''

Our simulation box exactly fits the waveguide; there's no empty
space around the waveguide in the simulation box, so the electric
field around the waveguide is truncated in the model. To be fair,
there are legitimate use cases if the surrounding field is irrelevant,
such as when modeling an infinite-length waveguide, or a capacitor
stuck in a metal box. But here, we are modeling a finite waveguide
with realistic behaviors, including fringe fields and radiation.

As a quick fix to the problem, one can make the simulation box several
times as big in volume in comparison to the waveguide, giving plenty of
room for the simulation to breathe. This allows fringe fields to
naturally decay.

We delete the original three
:meth:`~CSX.ContinuousStructure.AddLine` calls and replace them with
the following code::

    # delete these lines
    # mesh.AddLine('x', [-50, 50])  # two lines at -50, 50
    # mesh.AddLine('y', [-50, 50])  # two lines at -50, 50
    # mesh.AddLine('z', [-8, 8])    # two lines at -8, 8
    # mesh.SmoothMeshLines('x', res)
    # mesh.SmoothMeshLines('y', res)
    # mesh.SmoothMeshLines('z', res)

    mesh.AddLine('x', [-100, 100])  # two lines at -100, 100
    mesh.AddLine('y', [-100, 100])  # two lines at -100, 100
    mesh.AddLine('z', [-50,   50])  # two lines at -50, 50

.. important::
   If it's necessary to capture fringe fields, increase the
   size of the simulation box beyond the object. The first
   and line mesh lines on each axis define the simulation
   box's boundary.

Zero-Thickness Metal Alignment
''''''''''''''''''''''''''''''

The Z mesh lines are now going from -50 to 50, and we're going to
rely on automatic interpolation
(by :meth:`~CSXCAD.CSRectGrid.CSRectGrid.SmoothMeshLines`)
to fill the gaps with additional lines.
The lack of guaranteed mesh line alignment becomes a problem.
Zero-thickness objects like the metal plates must align to an exact mesh
line, otherwise these objects can't be simulated and will be ignored
by the simulator. Thus, we create two mesh lines on the Z axis at the
exact level of the plates::

    # zero-thickness metal plates need mesh lines at their exact levels
    mesh.AddLine('z', [-8, 8])

.. important::
   Zero-thickness metal plates (and other planar objects) need mesh
   lines at their exact levels.

1/3-2/3 Rule
'''''''''''''

After solving the misalignment on the Z axis, ironically, we
have an opposite problem on the XY plane. On both planes, the mesh
is perfectly aligned with the edges of the waveguide plates, which
introduces numerical inaccuracies.

In FDTD, a fundamental error source is the singularities around metal
edges, which have strong electric fields that are difficult to calculate
properly. As long as the mesh line and metal edge are aligned exactly,
the simulation accuracy is degraded unnecessarily - increasing the
mesh resolution is inefficient and ineffective. It's wasteful and
only has a marginal effect.

To mitigate this technical limitation, the mesh should be intentionally
misaligned with metal edges. For the best results, we introduce additional
cells with different sizes around the edge, creating a non-uniform
rectilinear mesh with variable spacing. Around the metal edge, the metal
occupies 1/3 of a cell, while the vacuum or insulator occupies 2/3 of a
cell. This is known as the **1/3-2/3 rule**.

Let's apply the rule to the X and Y axis::

    # use a smaller cell size around metal edge, not just the base mesh size
    highres = res / 1.5

    # strategically draw lines misaligned with the waveguide's left and right edges,
    # so that the edge occupies 33% space within a cell.
    mesh.AddLine('x', [
        # apply 1/3-2/3 rule on the X axis
        -50 + highres * 1/3, -50 - highres * 2/3,  # left edge
         50 - highres * 1/3,  50 + highres * 2/3   # right edge
    ])

    mesh.AddLine('y', [
        # apply 1/3-2/3 rule on the Y axis
        -50 + highres * 1/3, -50 - highres * 2/3,  # lower edge
         50 - highres * 1/3,  50 + highres * 2/3   # upper edge
    ])

The interval between the 1/3 and 2/3 mesh lines (i.e. the length of the
cell) is ``highres``, which is smaller than the base interval ``res`` by a
factor of 1.5. This is a workaround: If the same mesh resolution is
used for all cells, when we later smooth the mesh,
:meth:`~CSXCAD.CSRectGrid.CSRectGrid.SmoothMeshLines` may add additional mesh
lines within our handcrafted cells, effectively undoing the 1/3-2/3 rule.
Using the `highres` interval works around the problem, since
:meth:`~CSXCAD.CSRectGrid.CSRectGrid.SmoothMeshLines` is not allowed to
subdivide an interval smaller than ``res``. A factor of 1.5 is recommended to
preserve mesh smoothness.

Finally we resmooth the mesh::

    mesh.SmoothMeshLines('x', res)
    mesh.SmoothMeshLines('y', res)
    mesh.SmoothMeshLines('z', res)

Now let's inspect the model again; it's now much better.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/capacitor_xy_bettermesh.png
   :width: 49%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/capacitor_yz_bettermesh.png
   :width: 49%

.. hint::
   **Perspective.** The mesh line alignment may look misleading in AppCSXCAD
   due to an oblique camera angle in the 3D scene. Click the
   :guilabel:`2D` button for a planar view.

   .. image:: images/Parallel_Plate_Capacitor_Waveguide/appcsxcad-2d-button.png
   .. image:: images/Parallel_Plate_Capacitor_Waveguide/appcsxcad-opacity-slider.png

   When in doubt, use :guilabel:`Rectilinear Grid` panel's :guilabel:`Edit` button
   (or call :meth:`~CSXCAD.CSRectGrid.CSRectGrid.GetLines()`) to check mesh
   coordinates manually.

   **Mesh interval.** Use a smaller mesh interval ``highres = res / 1.5``
   for cells where 1/3-2/3 rules is applied
   to prevent :meth:`~CSXCAD.CSRectGrid.CSRectGrid.SmoothMeshLines`
   from undoing it. Alternatively, one may avoid calling this function by:

   * Adding mesh lines one by one manually.
   * Extracting all generated lines and removing unwanted ones.
   * Moving the whole simulated structure by an offset.

   For our purpose, the ``highres`` workaround is the most convenient solution.

   **Imperfect rule is still better than no rule.** If the
   simulated structure is complicated, making it difficult to apply the
   1/3-2/3 rule, at least try avoiding an exact alignment between metal
   edge and the mesh. This won't be as accurate as the 1/3-2/3 rule, but
   still gives a small accuracy boost at no cost.

.. _fielddump:

Optional: Create A Field Dump Box
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Some applications make use of the raw electromagnetic fields, not just the
input and output signals. We can do this by creating a "dump box" (a region
in space where field values are recorded) to save field samples to disk.
For troubleshooting malfunctioning setups, this is especially helpful as one can
identify the problematic region through direct visualization.

Several kinds of dump boxes exist.

#. Time-domain dumps of electric field :math:`\mathbf{E}`, auxiliary magnetic
   field :math:`\mathbf{H}`, electric conduction current :math:`\mathbf{J}`,
   total current density :math:`\mathrm{\nabla} \times \mathbf{H}`, electric
   displacement field :math:`\mathbf{D}`, and magnetic field (flux density)
   :math:`\mathbf{B}`, with their ``dump_type`` numbered from ``0`` to ``5``.
#. Frequency-domain dumps of electric field, auxiliary magnetic field,
   electric conduction current, total current density, electric displacement
   field, and magnetic field (flux density), numbered from ``10`` to ``15``.
#. Specific Absorption Rate (SAR) for biological EM radiation exposure analysis.
#. Near-Field to Far-Field Transformation (NF2FF) for antenna analysis
   (special setup required, via :meth:`openEMS.openEMS.CreateNF2FFBox` and a
   separate post-processing tool).

.. seealso::
   For usage, see :meth:`CSXCAD.ContinuousStructure.AddDump`. For a detailed
   list of parameters, see :class:`CSXCAD.CSProperties.CSPropDumpBox`.

In this example, we will dump the total current density (``dump_type=3``)
on the upper waveguide plate as a 2D surface, using the
:meth:`~CSX.ContinuousStructure.AddDump` function::

    dump = csx.AddDump("curl_H_upper", dump_type=3)
    dump.AddBox(start=[-100, -100, 8], stop=[100, 100, 8])

We do something slightly unusual here: The XY slice of the entire
simulation box is dumped at Z = 8, not just the waveguide plate.
This way, we can visualize the current both on the waveguide plates
and in vacuum, allowing us to see radiated field in the surrounding
free space.

.. note::

   openEMS calculates the total current density via Ampere-Maxwell's
   law :math:`\mathrm{\nabla} \times \mathbf{H}`, which is
   :math:`\mathbf{J} + \frac{\partial \mathbf{D}}{\partial t}`
   (i.e. the sum of conduction current and displacement current).

.. _port:

Purpose of Ports
^^^^^^^^^^^^^^^^^

The bulk of the 3D modeling work is nearly complete. The final step is
introducing a port and its excitation signal.

Ports are best understood as the virtual 3D counterpart of physical ports
on RF/microwave components, such as the standard 50 Ω input or output ports
on circuit boards, signal generators, oscilloscopes, and especially Vector
Network Analyzers (VNA). A port is treated as a lumped circuit, located
at a defined position. It acts as a voltage source or load with a
resistive impedance. It can inject a signal to the Device-Under-Test (DUT)
or measure the DUT's response, either from its own signal or from another
port.

.. figure:: images/Parallel_Plate_Capacitor_Waveguide/vna_ports.svg
   :class: with-border
   :width: 49%

   The "port" in openEMS serves a purpose similar to the physical
   ports on Vector Network Analyzers and circuit boards. Both kinds of
   ports are used to inject an input signal at a particular point in the
   Device-Under-Test (DUT), and to measure what comes out at another point.
   The DUT is thus characterized as a black box, solely represented using
   its input-output relationships without an internal structure.
   Note that port implementations are fundamentally different in
   physical instruments (via circuits) and in openEMS simulations
   (by loading numerical values into Yee cells). Image by Julien Hillairet,
   from the ``scikit-rf`` project, licensed under BSD-3, modified for clarity.

Internally, a port is implemented by locating the Yee cells occupied
by the port, and setting the numerical values of the electric fields in
these cells based on the excitation waveform. Thus, a port can be understood
as a source that injects electromagnetic energy into the simulation, helping
establish
initial conditions for the system. Simultaneously, a lumped resistor and
a probe are also created at the same location as the port, allowing it to
provide a matched load for the signal, or to measure the voltage or current
at this region.

Select the Right Port Type
""""""""""""""""""""""""""""

In openEMS, ports are ideal sources of EM fields, but they are not ideal
*launchers* of EM waves into structures due to a discontinuity at the
boundary between the port and the structure.
If port placement is not optimized,
this region of discontinuity may introduce artifacts such
as reflections or excitation of spurious modes. Optimizing the placement
and implementation of a port reduces these artifacts. This can be done
by using smooth transitions or by shaping the electric fields initially
injected by the port.

In openEMS, the standard port is the lumped port
(:meth:`~openEMS.openEMS.AddLumpedPort`) that works with most structures.
If an optimal transition is needed, openEMS also provides optimized implementations
of curved (``AddCurvePort.m``), microstrip (:meth:`~openEMS.ports.MSLPort`),
stripline (``AddStripLinePort.m``), coplanar waveguide (``AddCPWPort.m``),
and coax cable (``AddCoaxialPort.m``) ports. For now, the lumped ports
suffice for our purpose.

.. note::
   Some port types are not *ported* (no pun intended) to Python yet.

Most specialized ports in openEMS are signal integrity optimizations rather
than strict requirements. However, in some cases, specialized ports are *required*
to excite the structure properly. In a rectangular or circular waveguide,
there is only one conductor. An ordinary port can't excite it correctly,
as the waveguide is essentially a DC short circuit (unlike our parallel-plate
waveguide, which has two conductors). Enclosed waveguides require
special waveguide ports to excite the unique TE-mode waves. This is why
openEMS provides general waveguides (:meth:`~openEMS.ports.WaveguidePort`),
rectangular waveguides (:meth:`~openEMS.ports.RectWGPort`), and circular
waveguides (``AddCircWaveGuidePort.m``) ports.

.. note::
   Like physical ports on real devices, the virtual ports in openEMS are not
   perfect. They're ideal sources of EM fields, but they are not ideal
   *launchers* of EM waves into structures. A port creates a region of
   discontinuity, so they may introduce artifacts.
   Optimizing the placement and implementation of a port reduces artifacts.
   Alternatively, these artifacts
   can be removed through calibration or de-embedding algorithms, an
   advanced topic beyond the scope of this tutorial.

   .. figure:: images/Parallel_Plate_Capacitor_Waveguide/error-box.svg
      :class: with-border
      :width: 60%

      The artifacts introduced by a two-port measurement can be viewed
      as two linear circuits (left error box, right error box) cascaded in
      series with the DUT. All three circuits are represented as three matrices,
      called their S-parameters. Measurement error can be reduced by making
      error boxes nearly transparent using optimized port transitions.
      Alternatively, by mathematically removing the port's contributions from
      the measured response using linear algebra, a process known as
      calibration or de-embedding (image by Ziad Hatab et, al., licensed
      under CC BY-SA 4.0 [14]_)

Test Fixture as 1-Port or 2-Port Network
"""""""""""""""""""""""""""""""""""""""""

To analyze the frequency response of the DUT, both 1-port, 2-port, and
multi-port measurements are available. In a 1-port measurement (*shunt*
method), a signal is injected into the DUT using a 50 Ω port, and the
reflected signal at the same port is measured. In a 2-port measurement,
the DUT is connected either in series (*series* method) or parallel
(*shunt-through* method) between the transmitter and receiver.
A signal is injected into the DUT using a 50 Ω transmitter port, and the
received signal is measured by the 50 Ω receiver port at the other side.
The 2-port techniques are shown in the following circuit diagram (the
impedance ``Z`` is a function of frequency).

.. figure:: images/Parallel_Plate_Capacitor_Waveguide/vna-shunt-thru-series.svg
   :class: with-border

   2-port shunt-through measurement (DUT in parallel), and 2-port series
   (DUT in series) measurement.

In the real world, the 1-port measurement method is only accurate when
the DUT's impedance is close to the port impedance. If the impedance
of the DUT is too high or too low, almost all energy is reflected back.
overwhelming the receiver with strong reflections, making them are hard to
distinguish.
For low-impedance DUTs, results are also sensitive to the port's contact
resistance. Both result in significant measurement errors.

Instead, two-port measurements detect the weak signal transmitted across
the DUT ("filtered" by a large series impedance or a low parallel impedance),
taking advantage of the sensitive radio receiver in the VNA. This allows
us to determine the DUT's impedance.

To illustrate the general principle of two-port networks and S-parameters
in microwave measurements, we follow the 2-port method here.

.. seealso::
   See [21]_ [22]_ on impedance determination using the 2-port methods
   with VNAs.

Create Ports
^^^^^^^^^^^^^

Ports and excitations are associated with the FDTD simulation object
:meth:`~openEMS.openEMS`. We need to create an instance of the
simulation, and link the CSXCAD 3D structure ``csx`` with the
simulator::

    import openEMS
    fdtd = openEMS.openEMS()
    fdtd.SetCSX(csx)

To use a lumped port, call the ``openEMS`` object's
:meth:`~openEMS.openEMS.AddLumpedPort` method.

Let's create two ports at the leftmost and rightmost sides of the
waveguide. Port 1 should be centered at x = -50, y = 0, and Port 2
should be centered at x = 50, y = 0. Lumped ports should touch both
waveguide plates, as the port is connected across them, so their Z
coordinates span from [-8, 8]. This matches the intuition of a
circuit diagram, in which a lumped port is vertically connected
across the horizontal transmission line. The port's internal voltage
source acts as both a wire and a source of electric field.

.. figure:: images/Parallel_Plate_Capacitor_Waveguide/lumped-port.svg
   :class: with-border

   An ideal lumped port is connected vertically across a horizontal
   circuit. Its internal voltage source acting as both a wire and a
   source of electric field. A lumped excitation port in openEMS
   works similarly here.

To complicate the issue further, a port must cross at least a mesh line, since
excitation is implemented by setting the electric field at the corresponding
Yee cells. Since we have enforced the 1/3-2/3 rule, if the port is placed
exactly at the edge of the plate, no mesh lines pass through the port. Thus,
as a compromise, we shift the port's location to the nearest mesh lines instead.
This may introduce a small error due to a measurement plane. But these
issues are negligible for this demo.

.. important::
   A port must cross or align exactly with at least one mesh line, otherwise
   it can't be simulated.

Another consideration is whether the port should have a length and width. For
transmission line simulations, the port should have a width as wide as the
line to minimize discontinuity. On the other hand, one can usually
keep the length 0, it represents a measurement plane which doesn't extend
into the line. Here to make the problem more interesting, we simulate a
port much smaller than the width of the plates to study the consequences
of impedance mismatch. We choose to create a port with a 5 mm width.

Both ports have an internal impedance of 50 Ω, but only the first port
is used for excitation, so we set Port 1's ``excite`` attribute to ``1`` (input),
and Port 2's ``excite`` attribute to ``0`` (load and output). The parameter ``z``
is the direction of the field, pointing from one plate to another on the Z
axis.

Both ports are also added to a Python list to keep track of them for
future processing::

    z0 = 50
    port = [None, None]
    port[0] = fdtd.AddLumpedPort(1, z0, [-50 + 1/3 * highres, -2.5, -8], [-50 + 1/3 * highres, 2.5, 8], 'z', excite=1)
    port[1] = fdtd.AddLumpedPort(2, z0, [ 50 - 1/3 * highres, -2.5, -8], [ 50 - 1/3 * highres, 2.5, 8], 'z', excite=0)

The following images show the ports we just created.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/capacitor_ports.png
   :width: 49%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/capacitor_ports_2.png
   :width: 49%

Create Excitation
^^^^^^^^^^^^^^^^^^^

We need to assign an input signal to the excitation port.

In FDTD simulations, a Gaussian pulse is normally used as it provides a
wideband and smooth signal without discontinuous jumps. In openEMS,
the Gaussian pulse is implemented as a sinusoidal carrier at the center
frequency, with its amplitude modulated by a Gaussian function. This signal
produces a well-distributed spectrum with a wide range of frequencies in both
sidebands around the carrier. This relatively clean spectrum enables
openEMS to accurately extract the DUT's frequency response in the
frequency domain.

Once the DUT's frequency response is known, linear circuit analysis
tools can evaluate its behavior under other input signals, so there's
no loss of generality.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/port1_vt_input.svg
   :width: 49%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/port1_vt_fft.svg
   :width: 49%

.. note::
   In contrast, pulses such as the Dirac impulse
   (:meth:`~openEMS.openEMS.SetDiracExcite`) or the Heaviside step
   function (:meth:`~openEMS.openEMS.SetStepExcite`) have discontinuous
   jumps, reaching the full value in just a single timestep. This can
   cause problems like the unrealistic excitation of rarely-seen high-order
   modes in transmission lines, or the overshoot and ringing of its rising
   edge due to numerical dispersion.
   As a workaround, users can provide custom
   bandwidth-limited expressions to openEMS (no built-in implementations
   exist currently) via :meth:`~openEMS.openEMS.SetCustomExcite`. For an
   example, see [7]_.  However, these pulses also suffer from rapid energy
   roll-off at high frequencies. The Gaussian pulse, by comparison,
   retains significant energy in both sidebands around the center carrier
   frequency.

To apply a Gaussian excitation, we call the
:meth:`~openEMS.openEMS.SetGaussExcite` method of the ``openEMS`` object.
The first argument is its center frequency,
the second argument is its -20 dB bandwidth, defining the signal's
spectral "spread". For a wideband simulation, both arguments should
be set to 1/2 of the simulation's upper frequency target::

    # Centered around 5 GHz with a 5 GHz 20 dB bandwidth.
    # Lower sideband covers 0 - 5 GHz, upper sideband covers 5 - 10 GHz.
    fdtd.SetGaussExcite(f_max / 2, f_max / 2)

Specify the Boundary Conditions
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In openEMS, all physical phenomena occur in a small simulation box, so
we must decide what happens to the electromagnetic fields at its edges.
These are known as the *boundary conditions* of Partial Differential
Equations (PDEs). To create an effective simulation, we must select the
appropriate boundary conditions. There are six in total, located at
the six faces of the box: ``x_min``, ``x_max``, ``y_min``, ``y_max``,
``z_min``, ``z_max``. Each is independently adjustable.

Reflecting (Dirichlet) Boundary Conditions
"""""""""""""""""""""""""""""""""""""""""""

If the finite nature of the simulation box and the reflection of EM waves at
the boundaries are acceptable, reflecting boundary conditions offer a simple
zero-overhead solution. They efficiently model structures inside metal
enclosures, above a ground plane, or with an electric or magnetic field
symmetry across a plane. No explicit modeling is necessary; the boundary
conditions implicitly enforce these behaviors.

.. figure:: images/Parallel_Plate_Capacitor_Waveguide/reverberation_chamber.jpg
   :class: with-border
   :width: 60%

   A simulation box with PEC at all boundaries acts like a shielded enclosure.
   EMC test labs use room-sized metal enclosures to create reverberation
   chambers. Taking advantage of the standing waves due to reflections,
   one can create strong electric fields at localized regions to generate
   electromagnetic interference for testing.
   Image by Dr. Hans Georg Krauthäuser (Hgk at English Wikipedia),
   licensed under CC BY-SA 3.0.

**Perfect Electric Conductor (PEC)**: The simplest treatment sets
the (tangential) electric field at the boundary to 0.
Since the electric field lines can't penetrate this boundary, it's
equivalent to a Perfect Electric Conductor (PEC) with infinite
conductivity, also known as an Electric Wall. All incoming waves are
fully reflected back, analogous to a 1D transmission line terminated
by a short circuit.

**Perfect Magnetic Conductor (PMC)**: By duality, it sets the magnetic
field to 0 at the boundary. It acts as a boundary at which
the magnetic field lines can't penetrate. It's equivalent to a Perfect
Magnetic Conductor (PMC) with infinite permeability, also known as the
Magnetic Wall. All incoming waves are fully reflected back as well,
but with a phase opposite to that of the PEC, analogous to a 1D
transmission line terminated by an open circuit. It's used mainly as
a mathematical tool to enforce field symmetry when simulating a
half-structure, as no natural material in the real world behaves like
a PMC.

Mathematically, both PEC and PMC are Dirichlet boundary conditions
that enforce fixed field values (e.g. zero).

.. note::
   For our simulation, instead of explicitly modeling metal
   plates, we can model a vacuum with nothing inside, taking advantage
   of the PEC boundary conditions at ``z_min`` and ``z_max`` for
   fast computation. However, this is out of this tutorial's scope,
   and it won't capture the fringe fields above and below. We won't use
   PEC or PMC in this example.

Absorbing Boundary Conditions (ABC)
"""""""""""""""""""""""""""""""""""

In open-boundary problems, such as antennas or structures with radiation
loss, *Absorbing Boundary Conditions (ABC)* must be used to suppress
reflections to prevent spurious simulation results. At its boundaries, the
electromagnetic waves are absorbed and dissipated without reflecting back,
creating the illusion of an infinitely large free space. If PEC and PMC
are analogous to a shorted or opened transmission line, the ABC is analogous
to a termination resistor.

In general, these simulations should use a simulation box larger than the
structure to avoid intrusion of strong fringe fields at the boundary.
However, deliberately running a transmission line into the PML (defined
below) can be used as a perfect termination. This allows one
to terminate a transmission line without knowing its characteristic
impedance, enabling impedance measurement. In fact, openEMS's microstrip
ports don't create a lumped termination resistor by default, expecting
users to place them into the PML at boundaries.

.. figure:: images/Parallel_Plate_Capacitor_Waveguide/anechoic_chamber.jpg
   :class: with-border
   :width: 60%

   A simulation box with PML at all boundaries acts like an anechoic chamber
   in an EMC test lab, mimicking an infinitely large free space. Image by
   Adamantios (at English Wikipedia), licensed under CC BY-SA 3.0.

Two kinds of Absorbing Boundary Conditions are implemented in openEMS.

#. **Mur's Boundary Condition (MUR)**. This is a
   first-generation boundary condition purely defined by differential
   equations, originally invented by Gerrit Mur in the 1980s. It has
   a moderate computational overhead, but it works only if the EM wave
   is traveling at a direction orthogonal to the boundary, with a
   well-defined phase velocity (e.g. the speed of light). Thus,
   reflections may cause errors if strong radiation exists due to
   imperfect absorption.

#. **Perfectly Matched Layer (PML)**. This is the
   second-generation boundary condition proposed in the 1990s, modeling
   the behavior of a hypothetical EM wave-absorbing material. Unlike
   Mur's ABC, PML occupies some physical cells in the simulation box
   (the actual boundary at the true edge remains PEC).
   This mimics the wave-absorbing foam on the wall of an anechoic chamber
   in EMC test labs.

   PML is a more effective absorber and is easy to use, but it has the
   highest computational overhead (especially in openEMS, due to suboptimal
   implementation). Avoid it if efficiency is critical (e.g. only use
   PML at the simulation box's face directly hit by radiation, and use
   MUR for other boundaries). Intrusion of fringe fields and evanescent
   waves into the PML can destabilize it, causing the simulation to "blow-up".
   Radiating structures must be kept at a distance of :math:`\lambda / 4`.

.. important::
   In openEMS, ``PML_8`` is commonly used, meaning the nearest 8 mesh lines
   of the simulation box are dedicated to the PML. This is an assumption made
   by the simulator and not visible in AppCSXCAD. Ensure your structures do not
   overlap with edge cells (unless intentionally terminating a region with
   the PML). Radiating structures must be kept at a distance of :math:`\lambda / 4`
   from PML as intrusion of evanescent waves (fringe fields) can create
   numerical instability.

Set Boundary Conditions
""""""""""""""""""""""""

To set the boundary conditions of the simulator, use
:meth:`openEMS.openEMS.SetBoundaryCond()` of the ``openEMS`` class. Its syntax is:
``SetBoundaryCond(bc)`` where ``bc`` is a list with 6 elements with the order
of ``[x_min, x_max, y_min, y_max, z_min, z_max]``. Each element is a string or
integer according to the following table. Note that the use of integers is
discouraged due to poor readability, but one may encounter them in older examples.

+-----------------------------+-----------+----+-------------------------------------------------+
|      Boundary Condition     | String    | ID |                   Notes                         |
+=============================+===========+====+=================================================+
|  Perfect Electric Conductor | ``PEC``   | 0  | Reflective. Fast.                               |
+-----------------------------+-----------+----+-------------------------------------------------+
|  Perfect Magnetic Conductor | ``PMC``   | 1  | Reflective. Fast.                               |
+-----------------------------+-----------+----+-------------------------------------------------+
|  Mur's Absorbing Boundary   | ``MUR``   | 2  | Absorbing. Slow.                                |
|                             |           |    |                                                 |
|                             |           |    | Only absorbs waves orthogonal to the boundary.  |
|                             |           |    |                                                 |
|                             |           |    | Named after Gerrit Mur.                         |
+-----------------------------+-----------+----+-------------------------------------------------+
|  Perfectly Matched Layer    | ``PML_8`` | 3  | Absorbing. Slowest.                             |
|                             |           |    |                                                 |
|                             | ``PML_x`` |    | ``x`` has a range [6, 20], 8 by default.        |
|                             |           |    |                                                 |
|                             |           |    | Occupies ``x`` mesh lines near the boundary.    |
|                             |           |    |                                                 |
|                             |           |    |                                                 |
|                             |           |    | Keep structures ``x`` cells away from boundary. |
|                             |           |    |                                                 |
|                             |           |    | For radiating structures, λ / 4 away.           |
+-----------------------------+-----------+----+-------------------------------------------------+

``PML_8`` is the boundary condition that we're going to use, so we write::

    fdtd.SetBoundaryCond(["PML_8", "PML_8", "PML_8", "PML_8", "PML_8", "PML_8"])

Start Simulation
^^^^^^^^^^^^^^^^^

Finally, to start the simulation, run::

    fdtd.Run(simdir)


Code Checkpoint 1
^^^^^^^^^^^^^^^^^^^

Our Existing Code So Far
"""""""""""""""""""""""""

As a checkpoint of our progress, the following is the complete code one
obtains if the previous instructions are followed. Note that in this code
snippet, we rearranged the order of some lines according to Python's coding
convention of import first, constant definitions second, and main logic the
last::

    import sys
    import math
    import pathlib

    import CSXCAD
    import openEMS
    from openEMS.physical_constants import C0

    # calculate mesh resolution according to simulation frequency
    unit = 1e-3
    f_min = 100e6  # post-processing only, not used in simulation
    f_max = 10e9   # determines mesh size and excitation signal bandwidth
    epsilon_r = 1
    v = C0 / math.sqrt(epsilon_r)
    wavelength = v / f_max / unit  # convert to millimeters
    res = wavelength / 10
    # use a smaller cell size around metal edge, not just the base mesh size
    highres = res / 1.5

    # port impedance
    z0 = 50

    # determine the simulation output path

    # find the directory of the script itself
    filepath = pathlib.Path(__file__)
    # create a directory named after the script, but without ".py"
    simdir = filepath.with_suffix("")
    simdir.mkdir(parents=True, exist_ok=True)
    # find the filename of the script itself, and replace ".py" with ".xml"
    xmlname = filepath.with_suffix(".xml").name
    # concat two paths
    xmlpath = simdir / xmlname

    csx = CSXCAD.ContinuousStructure()
    fdtd = openEMS.openEMS()

    # associate the CSXCAD structure with the FDTD simulator
    fdtd.SetCSX(csx)

    # set unit of measurement in the CSXCAD drawing
    mesh = csx.GetGrid()
    mesh.SetDeltaUnit(unit)

    # Create an instance of material named "plate".
    # AddMetal() creates a Perfect Electric Conductor.
    metal = csx.AddMetal('plate')

    # Build two 3D shapes from -50 to 50 on the X/Y axes, located at Z = -8
    # and Z = 8 respectively. Note that the starting and stopping Z coordinates
    # of each plate are the same, so these metal plates have zero thickness.
    metal.AddBox(start=[-50, -50, -8], stop=[50, 50, -8])  # lower plate
    metal.AddBox(start=[-50, -50,  8], stop=[50, 50,  8])  # upper plate

    mesh.AddLine('x', [-100, 100])  # two lines at -100, 100
    mesh.AddLine('y', [-100, 100])  # two lines at -100, 100
    mesh.AddLine('z', [-50,   50])  # two lines at -50, 50

    # zero-thickness metal plates need mesh lines at their exact levels
    mesh.AddLine('z', [-8, 8])    # two lines at -8, 8

    # strategically draw lines misaligned with the waveguide's left and right edges,
    # so that the edge occupies 33% space within a cell.
    mesh.AddLine('x', [
        # apply 1/3-2/3 rule on the X axis
        -50 + highres * 1/3, -50 - highres * 2/3,  # left edge
         50 - highres * 1/3,  50 + highres * 2/3   # right edge
    ])

    mesh.AddLine('y', [
        # apply 1/3-2/3 rule on the Y axis
        -50 + highres * 1/3, -50 - highres * 2/3,  # lower edge
         50 - highres * 1/3,  50 + highres * 2/3   # upper edge
    ])

    mesh.SmoothMeshLines('x', res)
    mesh.SmoothMeshLines('y', res)
    mesh.SmoothMeshLines('z', res)

    dump = csx.AddDump("curl_H_upper", dump_type=3)
    dump.AddBox(start=[-100, -100, 8], stop=[100, 100, 8])

    port = [None, None]
    port[0] = fdtd.AddLumpedPort(1, z0, [-50 + 1/3 * highres, -2.5, -8], [-50 + 1/3 * highres, 2.5, 8], 'z', excite=1)
    port[1] = fdtd.AddLumpedPort(2, z0, [ 50 - 1/3 * highres, -2.5, -8], [ 50 - 1/3 * highres, 2.5, 8], 'z', excite=0)

    # save structure to file for inspection
    csx.Write2XML(str(xmlpath))

    # Centered around 5 GHz with a 5 GHz 20 dB bandwidth.
    # Lower sideband covers 0 - 5 GHz, upper sideband cover 5 - 10 GHz.
    fdtd.SetGaussExcite(f_max / 2, f_max / 2)
    fdtd.SetBoundaryCond(["PML_8", "PML_8", "PML_8", "PML_8", "PML_8", "PML_8"])
    fdtd.Run(simdir)

Code Refactoring
"""""""""""""""""

The program above is not entirely satisfactory. If we want make some
small adjustments to the model, it forces us to run the simulation. For
improved modularity, the 3D modeling, ports creation, simulation, and
post-processing should each be written in separated functions, so each
part can be ran and reran separately. This is important for repeated
simulations with different parameters, which are encountered in all
but the most trivial simulations.

The following refactored program provides a reasonable skeleton to
build upon further::

    import sys
    import math
    import pathlib

    import CSXCAD
    import openEMS
    from openEMS.physical_constants import C0

    # calculate mesh resolution according to simulation frequency
    unit = 1e-3
    f_min = 100e6  # post-processing only, not used in simulation
    f_max = 10e9   # determines mesh size and excitation signal bandwidth
    epsilon_r = 1
    v = C0 / math.sqrt(epsilon_r)
    wavelength = v / f_max / unit  # convert to millimeters
    res = wavelength / 10
    # use a smaller cell size around metal edge, not just the base mesh size
    highres = res / 1.5

    # port impedance
    z0 = 50

    # determine the simulation output path

    # find the directory of the script itself
    filepath = pathlib.Path(__file__)

    # use a directory named after the script, but without ".py"
    simdir = filepath.with_suffix("")
    simdir.mkdir(parents=True, exist_ok=True)

    # find the filename of the script itself, and replace ".py" with ".xml"
    xmlname = filepath.with_suffix(".xml").name

    # concat two paths
    xmlpath = simdir / xmlname


    def generate_structure(csx):
        """
        Generate and return the 3D structure used for simulation.

        This function should return a CSXCAD instance, but without changing any
        simulation parameters.
        """
        # set unit of measurement in the CSXCAD drawing
        mesh = csx.GetGrid()
        mesh.SetDeltaUnit(unit)

        # Create an instance of material named "plate".
        # AddMetal() creates a Perfect Electric Conductor.
        metal = csx.AddMetal('plate')

        # Build two 3D shapes from -50 to 50 on the X/Y axes, located at Z = -8
        # and Z = 8 respectively. Note that the starting and stopping Z coordinates
        # of each plate are the same, so these metal plates have zero thickness.
        metal.AddBox(start=[-50, -50, -8], stop=[50, 50, -8])  # lower plate
        metal.AddBox(start=[-50, -50,  8], stop=[50, 50,  8])  # upper plate

        mesh.AddLine('x', [-100, 100])  # two lines at -100, 100
        mesh.AddLine('y', [-100, 100])  # two lines at -100, 100
        mesh.AddLine('z', [-50,   50])  # two lines at -50, 50

        # zero-thickness metal plates need mesh lines at their exact levels
        mesh.AddLine('z', [-8, 8])    # two lines at -8, 8

        # strategically draw lines misaligned with the waveguide's left and right edges,
        # so that the edge occupies 33% space within a cell.
        mesh.AddLine('x', [
            # apply 1/3-2/3 rule on the X axis
            -50 + highres * 1/3, -50 - highres * 2/3,  # left edge
             50 - highres * 1/3,  50 + highres * 2/3   # right edge
        ])

        mesh.AddLine('y', [
            # apply 1/3-2/3 rule on the Y axis
            -50 + highres * 1/3, -50 - highres * 2/3,  # lower edge
             50 - highres * 1/3,  50 + highres * 2/3   # upper edge
        ])

        mesh.SmoothMeshLines('x', res)
        mesh.SmoothMeshLines('y', res)
        mesh.SmoothMeshLines('z', res)

        dump = csx.AddDump("curl_H_upper", dump_type=3)
        dump.AddBox(start=[-100, -100, 8], stop=[100, 100, 8])

        return csx


    def setup_ports(fdtd, csx):
        """
        Create and return ports to inject and measure signals at a particular mesh
        location.

        This function should only create ports, but without changing the structure
        or simulation parameters.
        """
        port = [None, None]
        port[0] = fdtd.AddLumpedPort(1, z0, [-50 + 1/3 * highres, -2.5, -8], [-50 + 1/3 * highres, 2.5, 8], 'z', excite=1)
        port[1] = fdtd.AddLumpedPort(2, z0, [ 50 - 1/3 * highres, -2.5, -8], [ 50 - 1/3 * highres, 2.5, 8], 'z', excite=0)
        return port


    def simulate(fdtd, csx):
        """
        Setup boundary conditions, excitation signals, and finally run the
        simulator.

        This function should run the simulator from the given "fdtd" and "csx"
        instance, without changing them.
        """
        # Centered around 5 GHz with a 5 GHz 20 dB bandwidth.
        # Lower sideband covers 0 - 5 GHz, upper sideband cover 5 - 10 GHz.
        fdtd.SetGaussExcite(f_max / 2, f_max / 2)
        fdtd.SetBoundaryCond(["PML_8", "PML_8", "PML_8", "PML_8", "PML_8", "PML_8"])
        fdtd.Run(simdir)


    def postproc(port):
        """
        Process the data generated by a complete simulation. Only knowledge of ports
        are necessary. This function is agnostic about the structure and simulator
        parameters.
        """
        pass


    if __name__ == "__main__":
        csx = CSXCAD.ContinuousStructure()
        fdtd = openEMS.openEMS()

        # associate CSXCAD structure with an openEMS simulation
        fdtd.SetCSX(csx)

        if len(sys.argv) <= 1:
            print('No command given, expect "generate", "simulate", "postproc"')
        elif sys.argv[1] in ["generate", "simulate"]:
            # generate 3D structure
            generate_structure(csx)
            setup_ports(fdtd, csx)
            csx.Write2XML(str(xmlpath))

            if sys.argv[1] == "simulate":
                # run simulator
                simulate(fdtd, csx)
        elif sys.argv[1] == "postproc":
            # run post-processing only, without running the simulator
            port = setup_ports(fdtd, csx)
            postproc(port)
        else:
            print("Unknown command %s" % sys.argv[1])
            exit(1)

This program accepts three separate commands: ``generate``, ``simulate``,
and ``postproc``. This allows incremental development.

One can immediately inspect the model in AppCSXCAD without starting
the simulation via::

    python3 Parallel_Plate_Capacitor_Waveguide.py generate

Once checked, one can manually start the simulation via::

    python3 Parallel_Plate_Capacitor_Waveguide.py simulate

Likewise, one can experiment with different post-processing
routines based on existing data without wasting time on
re-running the simulation via::

    python3 Parallel_Plate_Capacitor_Waveguide.py postproc

In the future, it's also easy to modify the program to add
parameterized structure generation, multi-pass simulations,
and other features.

Run Simulation
^^^^^^^^^^^^^^^

Expected Simulator Output
"""""""""""""""""""""""""

If everything works as expected, the following screen appears.
For this small simulation, it should finish within a few minutes::

    $ python3 Parallel_Plate_Capacitor_Waveguide.py
     ----------------------------------------------------------------------
     | openEMS 64bit -- version v0.0.36-16-g7d7688a
     | (C) 2010-2023 Thorsten Liebig <thorsten.liebig@gmx.de>  GPL license
     ----------------------------------------------------------------------
    	Used external libraries:
    		CSXCAD -- Version: v0.6.3-4-g9257bf1
    		hdf5   -- Version: 1.12.1
    		          compiled against: HDF5 library version: 1.12.1
    		tinyxml -- compiled against: 2.6.2
    		fparser
    		boost  -- compiled against: 1_76
    		vtk -- Version: 9.1.0
    		       compiled against: 9.1.0

    Create FDTD operator (compressed SSE + multi-threading)
    FDTD simulation size: 70x70x37 --> 181300 FDTD cells
    FDTD timestep is: 5.3429e-12 s; Nyquist rate: 9 timesteps @1.0398e+10 Hz
    Excitation signal length is: 108 timesteps (5.77033e-10s)
    Max. number of timesteps: 1000000000 ( --> 9.25926e+06 * Excitation signal length)
    Create FDTD engine (compressed SSE + multi-threading)
    Running FDTD engine... this may take a while... grab a cup of coffee?!?
    [@        4s] Timestep:         1602 || Speed:   72.5 MC/s (2.499e-03 s/TS) || Energy: ~2.70e-19 (-41.66dB)
    [@        8s] Timestep:         3820 || Speed:  100.5 MC/s (1.805e-03 s/TS) || Energy: ~5.35e-20 (-48.70dB)
    [@       12s] Timestep:         6510 || Speed:  121.9 MC/s (1.488e-03 s/TS) || Energy: ~1.86e-20 (-53.30dB)
    [@       16s] Timestep:         9320 || Speed:  127.3 MC/s (1.424e-03 s/TS) || Energy: ~1.09e-20 (-55.59dB)
    [@       20s] Timestep:        12136 || Speed:  127.6 MC/s (1.421e-03 s/TS) || Energy: ~5.33e-21 (-58.72dB)
    [@       24s] Timestep:        14748 || Speed:  118.4 MC/s (1.532e-03 s/TS) || Energy: ~3.62e-21 (-60.39dB)
    Multithreaded Engine: Best performance found using 5 threads.
    Time for 14748 iterations with 181300.00 cells : 24.01 sec
    Speed: 111.35 MCells/s

If there's a mesh or port alignment alignment problem, openEMS may
generate the following warnings. See the linked sections for their
respective solution.

* :ref:`unused_plate`
* :ref:`unused_excite`
* :ref:`voltage_integral_error`

Convergence and Divergence (Blow-up)
""""""""""""""""""""""""""""""""""""""

The simulation runs until the total energy in the simulation box
decays to nearly zero, reaching 60 dB below the initial energy
injected by the excitation port. When this occurs, the simulation
achieves convergence, meaning the transients in the system have
dissipated, and the system has reached a steady-state. Thus, the
simulation terminates.

Conversely, incorrect or unphysical
modeling or meshing may destabilize the simulation, causing
*blow-ups*. The simulation box's EM field strength diverges over
time due to the accumulation of small numerical errors. The
total energy may gradually increase unbounded, eventually
reaching the floating-point infinity.
If the energy shows signs of rapid increases, the simulation
should be stopped early via :kbd:`Control-C` to avoid wasting time.

Note that the displayed energy value is only a rough, indicative estimate.
Factors such as material properties are ignored for simulation speed.
For resonating structures (such as cavity resonators and antennas),
the energy indicator may fluctuate up and down repeatedly
due to the oscillating EM field strengths. The convergence time
required for low-loss (high Q) resonators which have minimal energy
dissipation, is notoriously long in FDTD simulations. The absence
of termination resistances or Absorbing Boundary Conditions
makes it difficult to dissipate the injected energy.

.. note::

   The energy decay threshold for termination is adjustable
   via :meth:`~openEMS.openEMS.SetEndCriteria`, but 60 dB is a
   good default. For advanced usage,
   :meth:`~openEMS.openEMS.SetNumberOfTimeSteps`
   and
   :meth:`~openEMS.openEMS.SetMaxTime` can limit the total
   number of timesteps (in iterations) or wall-clock time
   (in seconds) to truncate the simulation earlier before
   convergence.

Built-In Post-Processing
^^^^^^^^^^^^^^^^^^^^^^^^^

By now, the electromagnetic field simulation is complete. It's up to us to
analyze and interpret the data.

To analyze the frequency response of the structure, we need to specify a
discrete list of frequencies of interest. One can generate such a list with the
help of numpy's ``linspace()`` function. The following code creates a ``freq_list``
with 1000 evenly-spaced elements from 100 MHz to 10 GHz::

    import numpy as np

    points = 1000
    freq_list = np.linspace(f_min, f_max, points)

Then, call the :meth:`~openEMS.Port.CalcPort` function on each port object one
by one to calculate its response. This is the reason that we've chosen to
append each port object to a list called ``port``, instead of leaving them
as individual variables::

    for p in port:
        p.CalcPort(simdir, freq_list, ref_impedance=z0)

Port Attribute References
"""""""""""""""""""""""""""

The following port attributes are often necessary in post-processing
for frequency response, impedance, and time-domain calculations.

+-----------------------------+-----------------+--------------------------------------------------+
|         Attribute           |    Domain       |  Definition                                      |
+=============================+=================+==================================================+
|         ``uf_inc[n]``       | Frequency Sample| Incident Voltage                                 |
+-----------------------------+-----------------+--------------------------------------------------+
|         ``uf_ref[n]``       | Frequency Sample| Reflected Voltage                                |
+-----------------------------+-----------------+--------------------------------------------------+
|         ``if_tot[n]``       | Frequency Sample| Total Voltage                                    |
+-----------------------------+-----------------+--------------------------------------------------+
|         ``if_inc[n]``       | Frequency Sample| Incident Current                                 |
+-----------------------------+-----------------+--------------------------------------------------+
|         ``if_ref[n]``       | Frequency Sample| Reflected Current                                |
+-----------------------------+-----------------+--------------------------------------------------+
|         ``if_tot[n]``       | Frequency Sample| Total Current                                    |
+-----------------------------+-----------------+--------------------------------------------------+
|          ``P_inc[n]``       | Frequency Sample| Incident Power                                   |
+-----------------------------+-----------------+--------------------------------------------------+
|          ``P_ref[n]``       | Frequency Sample| Reflected Power                                  |
+-----------------------------+-----------------+--------------------------------------------------+
|          ``P_acc[n]``       | Frequency Sample| Accepted Power (Incident - Reflected)            |
+-----------------------------+-----------------+--------------------------------------------------+
|         ``ut_inc[n]``       | Time Sample     | Incident Voltage                                 |
+-----------------------------+-----------------+--------------------------------------------------+
|         ``ut_ref[n]``       | Time Sample     | Reflected Voltage                                |
+-----------------------------+-----------------+--------------------------------------------------+
|         ``ut_tot[n]``       | Time Sample     | Total Voltage                                    |
+-----------------------------+-----------------+--------------------------------------------------+
|         ``it_inc[n]``       | Time Sample     | Incident Current                                 |
+-----------------------------+-----------------+--------------------------------------------------+
|         ``it_ref[n]``       | Time Sample     | Reflected Current                                |
+-----------------------------+-----------------+--------------------------------------------------+
|         ``it_tot[n]``       | Time Sample     | Total Current                                    |
+-----------------------------+-----------------+--------------------------------------------------+
|   ``u_data.ui_val[0][n]``   | Time Sample     | Raw Voltage (``ut_tot`` Recommended)             |
+-----------------------------+-----------------+--------------------------------------------------+
|   ``u_data.ui_time[0][n]``  | Time Sample     | Raw Time of Voltage Samples                      |
+-----------------------------+-----------------+--------------------------------------------------+
|    ``i_data.ui_val[0][n]``  | Time Sample     | Raw Current (``it_tot`` Recommended)             |
+-----------------------------+-----------------+--------------------------------------------------+
|    ``i_data.ui_time[0][n]`` | Time Sample     | Raw Time of Current Samples                      |
+-----------------------------+-----------------+--------------------------------------------------+
| Note for Americans: ``u`` is the symbol of voltage (U) in ISO/IEC convention.                    |
+--------------------------------------------------------------------------------------------------+

Introduction to S-parameters
"""""""""""""""""""""""""""""

In radio and high-speed digital electronics engineering, the *S-parameters*
are the universal language to express the characteristics of a linear circuit,
allowing one to treat the circuit as a black box with several *ports*, defined
by their input-output relationships.

For a two-port measurement, the circuit is modeled by a 2x2 matrix with 4
complex numbers at each frequency of interest. These parameters can be interpreted
as the transmission and reflection of voltage waves in a transmission line.

.. |s_param_filter| image:: images/Parallel_Plate_Capacitor_Waveguide/s_param_filter.jpg

.. table::
  :widths: 50 50
  :align: center

  +--------------------------------------+-----------------------------------------------------------------+
  | .. math::                            |                                                                 |
  |                                      |                                                                 |
  |   \large{                            |                                                                 |
  |   \begin{bmatrix}                    |                                                                 |
  |   b_1 \\ b_2 \end{bmatrix} =         |                                                                 |
  |   \begin{bmatrix} S_{11} & S_{12} \\ |     |s_param_filter|                                            |
  |   S_{21} & S_{22} \end{bmatrix}      |                                                                 |
  |   \begin{bmatrix} a_1 \\ a_2         |                                                                 |
  |   \end{bmatrix}                      |                                                                 |
  |   }                                  |                                                                 |
  |                                      |                                                                 |
  +--------------------------------------+-----------------------------------------------------------------+
  | S-parameters can be understood as the transmitted and reflected signals (voltage waves) at the         |
  |                                                                                                        |
  | port of a DUT. The DUT itself is seen as a black box, defined only by their input-output relationship  |
  |                                                                                                        |
  | Image by Charly Whisky, licensed under CC BY-SA 4.0.                                                   |
  +--------------------------------------------------------------------------------------------------------+

At Port 1, its total voltage can be seen as a superposition of two parts:
an incident voltage wave entering Port 1, and a reflected wave leaving from
Port 1. Their ratio :math:`V_\mathrm{ref} / V_\mathrm{inc}` is the parameter
:math:`S_{11}`. This parameter has several other names: the reflection
coefficient :math:`\Gamma = S_{11}`. When its magnitude is plotted on a log
scale, it's called the return loss :math:`-10 \cdot \log_{10}(|S_{11}|^2)`.

At Port 2, the voltage or power it receives can also be calculated as a
ratio of two quantities: an incident voltage wave entering Port 1, and a
transmitted voltage wave leaving from Port 2, called :math:`S_{21}`. This
parameter is also known as transmission coefficient  When its magnitude is
plotted on the log scale, it's called the insertion loss
:math:`-10 \cdot \log_{10}(|S_{21}|^2)`. This corresponds to the
attenuation of a cable or a filter.

Since a circuit may modify both the amplitude and the phase of a voltage
wave, all S-parameters are complex numbers.

Calculate S-parameters
"""""""""""""""""""""""

In openEMS, after calling ``CalcPort()`` on a port object, its incident and reflected
voltages can be accessed via its ``uf_inc`` and ``uf_ref`` attributes, which are
``numpy`` lists with the same number of elements as ``freq_list`` (which we previously
passed to ``CalcPort()``).

Thus, by definition, one can calculate :math:`S_{11}` and :math:`S_{21}` as the
following::

    s11_list = port[0].uf_ref / port[0].uf_inc
    s21_list = port[1].uf_ref / port[0].uf_inc

.. hint::
  We utilize `numpy`'s broadcasting feature here. The quantities ``uf_ref`` and
  ``uf_inc`` are arrays, not scalars. Every element represents a single value at
  the a frequency point. But instead of looping over each element explicitly,
  we can work on all elements simultaneously::

      a = np.array([1, 2, 3])
      b = np.array([4, 5, 6])
      c = a + b  # [5, 7, 9]
      d = c * 2  # [10, 14, 18]

  This way, we can do element-wise arithmetic automatically on the whole
  ``numpy`` arrays. We will use this feature extensively throughout the
  rest of this tutorial.

Two S-parameters :math:`S_{12}` and :math:`S_{22}` are still missing here. The
correct way of doing so is restarting the simulation, but with port 2 as
the excitation port instead of port 1. This means that in the general case, we
must refactor our code to move the port creation and simulation logic into separate
functions.

But here, since our parallel-plate waveguide is symmetric and reciprocal, it
doesn't matter if we excite the left side and terminate the right side, or vice
versa, so we can assume :math:`S_{12} = S_{21}` and :math:`S_{11} = S_{22}`::

    # hack: assume symmetry and reciprocity
    # This only correct for passive linear reciprocal circuit!
    s22_list = s11_list
    s12_list = s21_list

Plot S-parameters via matplotlib
"""""""""""""""""""""""""""""""""

After going through the long exercise, it's now a good time to take a
quick look at the S-parameters we've obtained, using `matplotlib`::

    from matplotlib import pyplot as plt

Here, we're interested in two types of graphs that all RF/microwave
engineers care about (Smith charts and more sophisticated analysis
will be introduced later, using third-party software).

The scalar return loss (magnitude of :math:`S_{11}` in decibels) on
a line chart shows how much power is reflected by the DUT back to
the input port at each frequency. A high loss like 20 dB implies no
reflection, indicating a good impedance match or a resonance frequency::

    s11_db_list = -10 * np.log10(np.abs(s11_list) ** 2)

    plt.figure()
    plt.plot(freq_list / 1e9, s11_db_list, label='$S_{11}$ dB')
    plt.grid()
    plt.legend()
    plt.xlabel('Frequency (GHz)')
    plt.ylabel('Return Loss (dB)')

    # By convention, return loss values increase downward on the Y axis.
    # This is consistent with the plot shapes on spectrum analyzers, and
    # perhaps explains why the customary "wrong" sign convention is used.
    plt.gca().invert_yaxis()
    plt.show()

Since :math:`S_{11}` is a vector (complex number), we can also plot its
phase angle for completeness. But note that the phase is difficult to
interpret, so the Smith chart (covered later) is a better tool for this
job::

    s11_deg_list = np.angle(s11_list, deg=True)

    plt.figure()
    plt.plot(freq_list / 1e9, s11_deg_list, label='$S_{11}$ deg')
    plt.grid()
    plt.legend()
    plt.xlabel('Frequency (GHz)')
    plt.ylabel('S11 Phase (deg)')

    plt.show()

As we can see from the plot, there's a sharp 40 dB notch at 500 MHz,
indicating almost all power is delivered into the load with almost no
reflection. This is usually the sign that the DUT is resonating. Above
2 GHz, the reflection becomes extremely strong, as the return loss is
only between 0 dB to 6 dB, suggesting a serious physical discontinuity
or electrical impedance mismatch.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/s11_db_sim.svg
   :width: 49%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/s11_deg_sim.svg
   :width: 49%

The scalar insertion loss (magnitude of :math:`S_{21}` in decibels)
on a line chart shows how much power is transmitted into the second port,
which shows the attenuation of signals at each frequency::

    s21_db_list = -10 * np.log10(np.abs(s21_list) ** 2)

    plt.figure()
    plt.plot(freq_list / 1e9, s21_db_list, label='$S_{21}$ dB')
    plt.grid()
    plt.legend()
    plt.xlabel('Frequency (GHz)')
    plt.ylabel('Insertion Loss (dB)')

    # By convention, insertion loss values increase downward on the Y axis.
    # This is consistent with the plot shapes on spectrum analyzers, and
    # perhaps explains why the customary "wrong" sign convention is used.
    plt.gca().invert_yaxis()
    plt.show()

    s21_deg_list = np.angle(s21_list, deg=True)

    plt.figure()
    plt.plot(freq_list / 1e9, s21_deg_list, label='$S_{21}$ deg')
    plt.grid()
    plt.legend()
    plt.xlabel('Frequency (GHz)')
    plt.ylabel('S21 Phase (deg)')

    plt.show()

From the simulation data, we can see that this parallel-plate waveguide's
insertion loss is anomalously high, around 20 dB at 2 GHz.
This behavior contradicts expectations, as both lumped capacitors and
transmission lines are typically lossless.
Unusual results like
this one may indicate a modeling mistake that either causes unphysical
behavior or shows theoretically-possible but unrealistic physical effects.
Alternatively, it may reveal actual physics rarely discussed in circuit
textbooks, so it can be "new" to us. We will discuss this issue later.

On the other hand, the phase angle runs between -180° and 180°. This is
the expected behavior consistent with real-world measurements.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/s21_db_sim.svg
   :width: 49%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/s21_deg_sim.svg
   :width: 49%

.. note::
   **Loss or Gain?**
   There's a technicality to nitpick. The term "loss" and "gain" are often
   used interchangeably in the RF/microwave industry. One may say a device
   has a return loss of -20 dB, but strictly speaking a negative loss implies
   a gain. This convention is technically incorrect but usually tolerated and
   widely used. On the other hand, its critics insist on inverting the sign
   (by multiplying all values by a factor of -1) when one is speaking of "loss".
   Otherwise, the proper term to speak of is "reflection coefficient", not
   "return loss".

   This issue can sometimes be quite controversial. For example, *IEEE Antennas
   and Propagation Magazine* started rejecting the former convention to promote
   rigor as of 2009. While the author of this tutorial has no particular opinion,
   to satisfy the potential nitpickers while following the tradition, here we use
   a compromise. Plots are labeled as "Return loss" and use the positive sign
   convention, but also with the Y axis inverted, so that resonances appear
   as familiar valleys.

   For more information, see [1]_ [2]_.

Plot Z-parameters (Impedances) via matplotlib
""""""""""""""""""""""""""""""""""""""""""""""

It's straightforward to transform S-parameters to Z-parameters (impedances) using
well-known formulas, so saving a redundant set of Z-parameters is unnecessary.
If :math:`Z_0` is the port impedance, :math:`S_{11}` (also known as the
reflection coefficient :math:`\Gamma`) is related to the load impedance
seen at the port via:

.. math::

   Z = Z_0 \frac{1 + \Gamma }{1 - \Gamma}

Nevertheless, it's possible to directly calculate the impedance seen by
a port via the total voltage ``uf_tot`` and total current ``if_tot`` attributes
as well.
The following example plots the impedance seen by port 1. Like S-parameters,
all impedances are also complex numbers, so we are only looking at their
magnitude (absolute value) here::

    # direct impedance calculation
    z11_list = np.abs(port[0].uf_tot / port[0].if_tot)

    # derive impedance from S11
    z11_from_s11_list = np.abs(z0 * (1 + s11_list) / (1 - s11_list))

    plt.figure()
    plt.plot(freq_list / 1e9, z11_list, label='$|Z_{11}|$ (Ω)')
    plt.plot(freq_list / 1e9, z11_from_s11_list, label='$|Z_{11}|$ (Ω)')
    plt.grid()
    plt.legend()
    plt.xlabel('Frequency (GHz)')
    plt.ylabel('Impedance Magnitude (Ω)')

    plt.show()

We find that the impedances derived from :math:`S_{11}` and from direct calculations
are indistinguishable.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/z11_mag_sim.svg
   :width: 49%

.. important::
   We repeat the warning that was already given in the :ref:`port`
   section. Beware that the impedance :math:`Z_{11}` is not the actual impedance
   of the DUT, such as this parallel-plate waveguide. It's only the total impedance
   seen by the port, including the contribution of both the DUT and port
   imperfections, which can lead to frequency-dependent artifacts. The DUT impedance
   can't be read from :math:`Z_{11}` without removing measurement artifacts. This
   can be achieved by eliminating port discontinuities or separating the contributions
   from the ports via de-embedding, calibration, or time gating. These are
   advanced topics not discussed here.

   Futhermore, even after de-embedding, it's not meaningful to associate this DUT's
   impedance with a pure capacitance, since this DUT is not a lumped capacitor but
   a parallel-plate waveguide. Only the RLCG transmission line is be the correct
   model here.

Code Checkpoint 2: Eliminate Repetitions in ``matplotlib`` Code
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

The matplotlib examples shown above are meant to be instructional. In
an actual program, we should avoid unproductive repetitions when we see
them. We can write a separate plotting function to avoid repeating the
same steps::

    def plot_param(x_list, y_list, linelabel, xlabel, ylabel, invert_y=True):
        plt.figure()
        plt.plot(x_list, y_list, label=linelabel)
        plt.grid()
        plt.legend()
        plt.xlabel(xlabel)
        plt.ylabel(ylabel)

        # By convention, loss values increase downward on the Y axis.
        # This is consistent with the plot shapes on spectrum analyzers, and
        # perhaps explains why the customary "wrong" sign convention is used.
        if invert_y:
            plt.gca().invert_yaxis()

        plt.show()


    def postproc(port):
        """
        Process the data generated by a complete simulation. Only knowledge of ports
        are necessary. This function is agnostic about the structure and simulator
        parameters.
        """
        for p in port:
            p.CalcPort(simdir, freq_list, ref_impedance=z0)

        s11_list = port[0].uf_ref / port[0].uf_inc
        s21_list = port[1].uf_ref / port[0].uf_inc

        # hack: assume symmetry and reciprocity
        # This only correct for passive linear reciprocal circuit!
        s22_list = s11_list
        s12_list = s21_list

        s11_db_list = -10 * np.log10(np.abs(s11_list) ** 2)
        s21_db_list = -10 * np.log10(np.abs(s21_list) ** 2)
        plot_param(freq_list / 1e9, s11_db_list, '$S_{11}$ dB', 'Frequency (GHz)', 'Return Loss (dB)')
        plot_param(freq_list / 1e9, s21_db_list, '$S_{21}$ dB', 'Frequency (GHz)', 'Insertion Loss (dB)')

        s11_deg_list = np.angle(s11_list, deg=True)
        s21_deg_list = np.angle(s21_list, deg=True)
        plot_param(freq_list / 1e9, s11_deg_list, '$S_{11}$ deg', 'Frequency (GHz)', 'S11 Phase')
        plot_param(freq_list / 1e9, s21_deg_list, '$S_{21}$ deg', 'Frequency (GHz)', 'S21 Phase')

        # direct impedance calculation
        z11_list = np.abs(port[0].uf_tot / port[0].if_tot)
        plot_param(freq_list / 1e9, z11_list, '$|Z_{11}|$ (Ω)', 'Frequency (GHz)', 'Impedance Magnitude (Ω)', invert_y=False)

Plot Time-Domain Waveforms via matplotlib
""""""""""""""""""""""""""""""""""""""""""

FDTD is fundamentally a time-domain field solver. In some cases it's
desirable to create custom excitations and observe transient responses
directly. For example, it allows one to truncate a simulation early
without waiting for convergence. It may help improving data quality
by preserving the DC term, or by avoiding violations of passivity and
causality in S-parameters due to numerical artifacts. Both may be
problematic if the frequency-domain data is used in transient simulations.

To obtain the time-domain waveforms at a port, we can use these port
attributes: ``u_data.ut_tot`` (total voltage), ``u_data.ui_time[0]`` (time
of voltage samples), ``i_data.it_tot`` (total current), ``i_data.ui_time[0]``
(time of current samples).

For the first try, let's plot the voltages at port 1 and port 2::

    plt.figure()
    plt.plot(port[0].u_data.ui_time[0], port[0].ut_tot, label="Input Voltage")
    plt.plot(port[1].u_data.ui_time[0], port[1].ut_tot, label="Output Voltage")
    plt.grid()
    plt.legend()
    plt.xlabel('Time (s)')
    plt.ylabel('Voltage (V)')
    plt.show()

The excitation signal is a sinusoid at the center frequency, with its amplitude
modulated by a Gaussian function. This is difficult to make sense of, since the
Gaussian function decays rapidly, but the simulator continues running until the
total energy has decayed below 60 dB. As a result, the waveform is a spike followed
by a flatline near 0. The tail of the waveform is almost invisible, because its
amplitude is many orders of magnitude smaller than the leading edge.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/port1_vt.svg
   :width: 49%

Let's try again, focusing only on the first 100 samples to improve
clarity::

    plt.figure()
    plt.plot(port[0].u_data.ui_time[0][0:100], port[0].ut_tot[0:100], label="Input Voltage")
    plt.plot(port[1].u_data.ui_time[0][0:100], port[1].ut_tot[0:100], label="Output Voltage")
    plt.grid()
    plt.legend()
    plt.xlabel('Time (s)')
    plt.ylabel('Voltage (V)')
    plt.show()

This is much better. The Gaussian modulation of the input signal is
evident. There's also a noticeable time delay before it reaches
the output, with an attenuation of one or two orders of magnitude. Both
are expected.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/port1_vt_early.svg
   :width: 49%

Time-domain usage of openEMS is most useful for custom excitation signals.
For example, one use case is to study the Time-Domain Transmission and
Time-Domain Reflection (TDT/TDR) characteristics of the DUT directly in the
time domain, which is beyond the scope of this tutorial. Hence, this tutorial
advocates relying on mature frequency-domain analysis tools based on
S-parameters (which can also be used in time-domain transient simulations
as well), as FDTD typically uses wideband Gaussian excitation signals that
are difficult to interpret in the time domain anyway.

.. hint::

   The raw time-domain voltages and currents at all ports are also available
   in the simulation directory as ``port_ut_1``, ``port_ut_2``, ``port_it_1``
   and ``port_it_2``, which can be analyzed by external programs.

Export S-parameters to Touchstone
""""""""""""""""""""""""""""""""""

At this point, we have nearly reached the limit of the built-in
post-processing capabilities of openEMS.
To bring our analysis further, it's now necessary to use third-party
software. Therefore we must save our S-parameters to an external file
to be used in other tools. In the RF/microwave industry, S-parameters
are encoded in a de facto standard file format called a *Touchstone*
file.

A 2-port Touchstone file has the file extension ``.s2p``, the first line
is its metadata.

The hash (``#``) symbol denotes a metadata line, not a comment line.
This line is mandatory (the real comment line begins with ``!``).
Letter ``S`` means the file contains S-parameters, keyword
``RI`` means the complex S-parameters are in the real-imaginary format,
and the final ``R 50`` means the port impedance for this measurement
is 50 Ω::

    # Hz S RI R 50

``.s2p`` file
''''''''''''''

In an ``.s2p`` file, each additional line defines the S-parameters at a new frequency
point, with the
order of ``frequency``, ``s11_real``, ``s11_imag``, ``s21_real``, ``s21_imag``, ``s12_real``,
``s12_imag``, ``s22_real``, ``s22_imag``.  All variables are floating-point numbers.

Thus we can write the simulation result as a Touchstone file using the following
code::

    # determine a file name
    s2pname = filepath.with_suffix(".s2p").name

    # concat two paths
    s2ppath = simdir / s2pname

    # write 2-port S-parameters
    with open(s2ppath, "w") as touchstone:
        touchstone.write("# Hz S RI R %f\n" % z0)  # Touchstone metadata, not comment!

        for idx, freq in enumerate(freq_list):
            s11 = s11_list[idx]
            s21 = s21_list[idx]
            s12 = s12_list[idx]
            s22 = s22_list[idx]
            entry = (freq, s11.real, s11.imag, s21.real, s21.imag, s12.real, s12.imag, s22.real, s22.imag)
            touchstone.write("%f %f %f %f %f %f %f %f %f\n" % entry)

``.s1p`` file
''''''''''''''
The ``.s2p`` file already contains the S-parameter :math:`S_{11}`, so
there's no need to create a separate 1-port Touchstone file. It's only
used for 1-port measurement data. For completeness, the Touchstone file
for 1-port measurements has the
suffix ``.s1p``. Its format is similar: at each frequency point, one
writes a new line with parameter order of ``frequency``, ``s11_real``,
``s11_imag``::

    # determine a file name
    s1pname = filepath.with_suffix(".s1p").name

    # concat two paths
    s1ppath = simdir / s1pname

    # write 1-port S-parameters
    with open(s1ppath, "w") as touchstone:
        touchstone.write("# Hz S RI R %f\n" % z0)  # Touchstone metadata, not comment!

        for idx, freq in enumerate(freq_list):
            s11 = s11_list[idx]
            entry = (freq, s11.real, s11.imag)
            touchstone.write("%f %f %f\n" % entry)

``.snp`` file
''''''''''''''

Finally, beware that for 3-port, 4-port, or more ports, the parameters order is
different. The website Microwaves101 contains useful information here. [3]_

Code Checkpoint 3
^^^^^^^^^^^^^^^^^^^

As a checkpoint of our progress, the following is the complete code one
obtains if the previous instructions are followed::

    import sys
    import math
    import pathlib

    import numpy as np
    from matplotlib import pyplot as plt

    import CSXCAD
    import openEMS
    from openEMS.physical_constants import C0

    # calculate mesh resolution according to simulation frequency
    unit = 1e-3
    f_min = 100e6  # post-processing only, not used in simulation
    f_max = 10e9   # determines mesh size and excitation signal bandwidth
    epsilon_r = 1
    v = C0 / math.sqrt(epsilon_r)
    wavelength = v / f_max / unit  # convert to millimeters
    res = wavelength / 10
    # use a smaller cell size around metal edge, not just the base mesh size
    highres = res / 1.5

    # calculate frequency points
    points = 1000
    freq_list = np.linspace(f_min, f_max, points)

    # port impedance
    z0 = 50

    # determine the simulation output path

    # find the directory of the script itself
    filepath = pathlib.Path(__file__)

    # use a directory named after the script, but without ".py"
    simdir = filepath.with_suffix("")
    simdir.mkdir(parents=True, exist_ok=True)

    # find the filename of the script itself, and replace ".py" with ".xml"
    xmlname = filepath.with_suffix(".xml").name

    # concat two paths
    xmlpath = simdir / xmlname


    def generate_structure(csx):
        """
        Generate and return the 3D structure used for simulation.

        This function should return a CSXCAD instance, but without changing any
        simulation parameters.
        """
        # set unit of measurement in the CSXCAD drawing
        mesh = csx.GetGrid()
        mesh.SetDeltaUnit(unit)

        # Create an instance of material named "plate".
        # AddMetal() creates a Perfect Electric Conductor.
        metal = csx.AddMetal('plate')

        # Build two 3D shapes from -50 to 50 on the X/Y axes, located at Z = -8
        # and Z = 8 respectively. Note that the starting and stopping Z coordinates
        # of each plate are the same, so these metal plates have zero thickness.
        metal.AddBox(start=[-50, -50, -8], stop=[50, 50, -8])  # lower plate
        metal.AddBox(start=[-50, -50,  8], stop=[50, 50,  8])  # upper plate

        mesh.AddLine('x', [-100, 100])  # two lines at -100, 100
        mesh.AddLine('y', [-100, 100])  # two lines at -100, 100
        mesh.AddLine('z', [-50,   50])  # two lines at -50, 50

        # zero-thickness metal plates need mesh lines at their exact levels
        mesh.AddLine('z', [-8, 8])    # two lines at -8, 8

        # strategically draw lines misaligned with the waveguide's left and right edges,
        # so that the edge occupies 33% space within a cell.
        mesh.AddLine('x', [
            # apply 1/3-2/3 rule on the X axis
            -50 + highres * 1/3, -50 - highres * 2/3,  # left edge
             50 - highres * 1/3,  50 + highres * 2/3   # right edge
        ])

        mesh.AddLine('y', [
            # apply 1/3-2/3 rule on the Y axis
            -50 + highres * 1/3, -50 - highres * 2/3,  # lower edge
             50 - highres * 1/3,  50 + highres * 2/3   # upper edge
        ])

        mesh.SmoothMeshLines('x', res)
        mesh.SmoothMeshLines('y', res)
        mesh.SmoothMeshLines('z', res)

        dump = csx.AddDump("curl_H_upper", dump_type=3)
        dump.AddBox(start=[-100, -100, 8], stop=[100, 100, 8])

        return csx


    def setup_ports(fdtd, csx):
        """
        Create and return ports to inject and measure signals at a particular mesh
        location.

        This function should only create ports, but without changing the structure
        or simulation parameters.
        """
        port = [None, None]
        port[0] = fdtd.AddLumpedPort(1, z0, [-50 + 1/3 * highres, -2.5, -8], [-50 + 1/3 * highres, 2.5, 8], 'z', excite=1)
        port[1] = fdtd.AddLumpedPort(2, z0, [ 50 - 1/3 * highres, -2.5, -8], [ 50 - 1/3 * highres, 2.5, 8], 'z', excite=0)
        return port


    def simulate(fdtd, csx):
        """
        Setup boundary conditions, excitation signals, and finally run the
        simulator.

        This function should run the simulator from the given "fdtd" and "csx"
        instance, without changing them.
        """
        # Centered around 5 GHz with a 5 GHz 20 dB bandwidth.
        # Lower sideband covers 0 - 5 GHz, upper sideband cover 5 - 10 GHz.
        fdtd.SetGaussExcite(f_max / 2, f_max / 2)
        fdtd.SetBoundaryCond(["PML_8", "PML_8", "PML_8", "PML_8", "PML_8", "PML_8"])
        fdtd.Run(simdir)


    def postproc(port):
        """
        Process the data generated by a complete simulation. Only knowledge of ports
        are necessary. This function is agnostic about the structure and simulator
        parameters.
        """
        for p in port:
            p.CalcPort(simdir, freq_list, ref_impedance=z0)

        s11_list = port[0].uf_ref / port[0].uf_inc
        s21_list = port[1].uf_ref / port[0].uf_inc

        # hack: assume symmetry and reciprocity
        # This only correct for passive linear reciprocal circuit!
        s22_list = s11_list
        s12_list = s21_list

        s11_db_list = -10 * np.log10(np.abs(s11_list) ** 2)
        s21_db_list = -10 * np.log10(np.abs(s21_list) ** 2)
        plot_param(freq_list / 1e9, s11_db_list, '$S_{11}$ dB', 'Frequency (GHz)', 'Return Loss (dB)')
        plot_param(freq_list / 1e9, s21_db_list, '$S_{21}$ dB', 'Frequency (GHz)', 'Insertion Loss (dB)')

        s11_deg_list = np.angle(s11_list, deg=True)
        s21_deg_list = np.angle(s21_list, deg=True)
        plot_param(freq_list / 1e9, s11_deg_list, '$S_{11}$ deg', 'Frequency (GHz)', 'S11 Phase')
        plot_param(freq_list / 1e9, s21_deg_list, '$S_{21}$ deg', 'Frequency (GHz)', 'S21 Phase')

        z11_list = np.abs(port[0].uf_tot / port[0].if_tot)  # direct impedance calculation
        plot_param(freq_list / 1e9, z11_list, '$|Z_{11}|$ (Ω)', 'Frequency (GHz)', 'Impedance Magnitude (Ω)', invert_y=False)

        plt.figure()
        plt.plot(port[0].u_data.ui_time[0][0:100], port[0].ut_tot[0:100], label="Input Voltage")
        plt.plot(port[1].u_data.ui_time[0][0:100], port[1].ut_tot[0:100], label="Output Voltage")
        plt.grid()
        plt.legend()
        plt.xlabel('Time (s)')
        plt.ylabel('Voltage (V)')
        plt.show()

        save_s2p(s11_list, s21_list, s12_list, s22_list)


    def plot_param(x_list, y_list, linelabel, xlabel, ylabel, invert_y=True):
        plt.figure()
        plt.plot(x_list, y_list, label=linelabel)
        plt.grid()
        plt.legend()
        plt.xlabel(xlabel)
        plt.ylabel(ylabel)

        # By convention, loss values increase downward on the Y axis.
        # This is consistent with the plot shapes on spectrum analyzers, and
        # perhaps explains why the customary "wrong" sign convention is used.
        if invert_y:
            plt.gca().invert_yaxis()

        plt.show()


    def save_s2p(s11_list, s21_list, s12_list, s22_list):
        # determine a file name
        s2pname = filepath.with_suffix(".s2p").name

        # concat two paths
        s2ppath = simdir / s2pname

        # write 2-port S-parameters
        with open(s2ppath, "w") as touchstone:
            touchstone.write("# Hz S RI R %f\n" % z0)  # Touchstone metadata, not comment!

            for idx, freq in enumerate(freq_list):
                s11 = s11_list[idx]
                s21 = s21_list[idx]
                s12 = s12_list[idx]
                s22 = s22_list[idx]
                entry = (freq, s11.real, s11.imag, s21.real, s21.imag, s12.real, s12.imag, s22.real, s22.imag)
                touchstone.write("%f %f %f %f %f %f %f %f %f\n" % entry)


    if __name__ == "__main__":
        csx = CSXCAD.ContinuousStructure()
        fdtd = openEMS.openEMS()

        # associate CSXCAD structure with an openEMS simulation
        fdtd.SetCSX(csx)

        if len(sys.argv) <= 1:
            print('No command given, expect "generate", "simulate", "postproc"')
        elif sys.argv[1] in ["generate", "simulate"]:
            # generate 3D structure
            generate_structure(csx)
            setup_ports(fdtd, csx)
            csx.Write2XML(str(xmlpath))

            if sys.argv[1] == "simulate":
                # run simulator
                simulate(fdtd, csx)
        elif sys.argv[1] == "postproc":
            # run post-processing only, without running the simulator
            port = setup_ports(fdtd, csx)
            postproc(port)
        else:
            print("Unknown command %s" % sys.argv[1])
            exit(1)

Third-Party Post-Processing
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

At this point, we have nearly reached the limit of the built-in
post-processing capabilities of openEMS. The only three post-processing
features we haven't used are Near-Field to Far-Field Transformation
(NF2FF) for antenna analysis, and Specific Absorption Rate (SAR)
analysis, but both are irrelevant for this simulation.

To bring our analysis further, it's now necessary to use third-party
software with the S-parameters we've saved.

S-parameters Analysis via ``scikit-rf``
"""""""""""""""""""""""""""""""""""""""

The library ``scikit-rf`` is an open source Python package for RF/microwave
engineering, including a powerful set of features for plotting, circuit
analysis, calibration, reading and writing data.

Install and Import
'''''''''''''''''''
One can install it via ``pip3`` in your home directory (or another ``venv``
environment you prefer)::

    pip3 install scikit-rf --user

Once installed, import the library::

    import skrf

``scikit-rf`` comes with a matplotlib chart theme with a modern look-and-feel.
Before using the library to do anything, one may call `stylely()` to
globally apply this theme. This affects all matplotlib charts, so you may
want to borrow it in other unrelated projects::

    skrf.stylely()

Plot Smith Charts
''''''''''''''''''

The basic entity in ``scikit-rf`` is a ``skrf.Network``, representing an
n-port network defined by an S-parameter. This object can be constructed
directly via the path to a Touchstone file::

    network = skrf.Network(s2ppath)

Now let's plot the complex reflections coefficient :math:`S_{11}` on
the Smith chart, the most important chart ever invented in RF/microwave
engineering. This chart makes it easier to interpret the phase angle in
the reflection coefficient and impedance by visually showing whether the
the DUT is resistive, capacitive, or inductive. To do so in ``scikit-rf``,
use the `plot_s_smith()` method of the `Network` object::

    network.plot_s_smith()
    plt.show()

By default, all S-parameters are plotted.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/s_all_smith_sim.svg

But it makes little sense to plot non-reflection parameters, so we specify
two optional parameters ``m`` and ``n``. To plot :math:`S_{11}` only, use::

    network.plot_s_smith(m=0, n=0)
    plt.show()

As we can see from the chart, the input impedence of the DUT is inductive.
This is likely due to the mismatches of both the electrical impedance
and the physical geometry of the excitation and termination ports at the
abrupt transition from the port to the waveguide.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/s11_smith_sim.svg

An alternative way to plot :math:`S_{11}` is to create a 1-port network
by extracting it from a multi-port network, using the ``s11`` attribute
(or ``s21``, ``s22``, etc). But note that doing this would create a new 1-port
network, so all charts will be labeled ``S11`` regardless of the parameter
plotted. Hence plotting a sliced network is not recommended::

    network.s11.plot_s_smith()
    plt.show()

Plot Magnitude-Phase Charts
''''''''''''''''''''''''''''''''

We can also plot other familiar charts via ``scikit-rf`` automatically, such
as the scalar :math:`S_{11}` plot in decibels and phase angles. This is
much faster than the manual method using only two lines of code for each chart::

    network.plot_s_db(m=0, n=0)
    plt.show()

    network.plot_s_deg(m=0, n=0)
    plt.show()

The data is identical to manual plotting, but without the need to do any
calculations. All is derived from S-parameters by scikit-rf automatically.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/s11_db_sim_skrf.svg
   :width: 49%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/s11_deg_sim_skrf.svg
   :width: 49%

Likewise, to plot :math:`S_{21}` using the same method::

    network.plot_s_db(m=1, n=0)
    plt.show()

    network.plot_s_deg(m=1, n=0)
    plt.show()

.. image:: images/Parallel_Plate_Capacitor_Waveguide/s21_db_sim_skrf.svg
   :width: 49%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/s21_deg_sim_skrf.svg
   :width: 49%

Plot Impedance Chart
'''''''''''''''''''''

`scikit-rf` also provides several routines to plot Z-parameters charts, such
as the scalar impedance::

    network.plot_z_mag(m=0, n=0)
    plt.show()

.. image:: images/Parallel_Plate_Capacitor_Waveguide/z11_mag_sim_skrf.svg
   :width: 49%

.. _skrf_tdr:

Time Domain Reflectometry
''''''''''''''''''''''''''

And now for something completely different: going from the frequency
domain back to the time domain. By applying an inverse Fourier transform
to the existing S-parameters (without needing to run new FDTD simulations),
``scikit-rf`` can directly calculate the DUT's equivalent transient
impulse and step impedance from :math:`S_{11}`. This enables capturing
impedance discontinuities that are difficult if not impossible to
identify using the Smith chart alone::

    # TDR requires the DC component to be physical
    network_dc = network.extrapolate_to_dc(kind='linear')

    plt.figure()
    plt.title("Time Domain Reflectometry - Step")
    network_dc.s11.plot_z_time_step(window='hamming', label="impedance")
    plt.xlim([-1, 2])  # look at the first two nanoseconds
    plt.show()

    plt.figure()
    plt.title("Time Domain Reflectometry - Impulse")
    network_dc.s11.plot_z_time_impulse(window='hamming', label="impedance")
    plt.xlim([-1, 2])  # look at the first two nanoseconds
    plt.show()

The calculated TDR plot with step excitation shows a strong reflection
at an instantaneous impedance of 140 Ω, followed by another reflection
of 70 Ω one nanoseconds later (the impulse excitation TDR plot is also
included here for completeness).

.. image:: images/Parallel_Plate_Capacitor_Waveguide/tdr_step_sim.svg
   :width: 49%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/tdr_impulse_sim.svg
   :width: 49%

This clearly indicates both ports exhibit discontinuities due to
impedance and geometry mismatches, as the lumped port impedance
differs from the waveguide's characteristic impedance.

In transmission line characterization, one should address such
mismatches by improving the simulation setup via better port
transitions. However, in this simulation, the mismatches are
intentionally introduced to demonstrate openEMS's ability to
accurately model real-world effects.

.. seealso::
   The features introduced in this manual are only the tip of the
   iceberg, it's not possible to cover all of its aspects here. See
   the full manual [12]_ for usage. Here we mention three important
   use cases.

   **Calibration and de-embedding.** If a port mismatch is unavoidable,
   post-processing techniques commonly used in Vector Network Analyzers
   can be applied to openEMS simulations as well, such as the SOLT
   calibration algorithm, or the newer IEEE P370 de-embedding algorithm.
   This involves making additional measurements (simulations) with known
   loads or structures at the port, so that the port or the test fixture's
   influence can be solved and removed. See [15]_ [16]_.

   **Time-gating.** An alternative possibility is working in the time
   domain using the equivalent TDR responses. If we transform the
   measured S-parameters into a TDR plot, a *time gate* can be applied
   that focuses on the response of only the DUT without the early and
   late reflections by the ports. This gated waveform can then be transformed
   back to the frequency domain to "clean" our S-parameters. This is
   not a rigorous solution in comparison to applying proper calibration
   algorithms, but is a quick-and-dirty solution. See [17]_.

   **Passivity and causality.** For time-domain simulations using
   S-parameters, the measured data must satisfy two criteria. The
   DUT itself must not amplify the input signal, violating energy
   conservation (passivity). The DUT must also not generate an output
   signal before an input signal arrives, violating the arrow of time
   (causality). Unfortunately, apparent perpetual motion machines and
   time machines are often created when S-parameters from measurements
   or simulations contain artifacts and noise, resulting in unphysical
   time-domain responses. This necessitates data quality checks and
   post-processing if violations are found. See [18]_ [19]_ [20]_.

Transient Analysis via ``SignalIntegrity``
""""""""""""""""""""""""""""""""""""""""""""

This tutorial has repeatedly claimed that the frequency response
is all you need: Once the DUT's S-parameters are known, linear
circuit analysis tools can evaluate its behavior under other
input signals, so there's no loss of generality. In fact, performing
a time-domain transient simulation using frequency-domain S-parameters
is a common feature in many proprietary commercial circuit simulators,
such as :program:`HyperLynx`, :program:`ADS`, or :program:`AWR`. A
third-party tool also exists for PSpice. [4]_

However, in the free and open source world, few (if any) free and
open source circuit simulators have this capability. For example,
Ngspice supports S-parameter calculation but not transient
simulation. [5]_ :program:`Qucs`, too, doesn't support transient
simulation with S-parameters, although it's possible to define
devices using them.

The Python package :program:`SignalIntegrity`, designed for signal
integrity and eye diagram simulations from the ground up, is a rare
exception. It's developed by Pete Pupalaikis, a former signal
integrity expert at LeCroy.

.. seealso::
   **Book.** The software's internal theory of operation is also published almost
   in full in the textbook *S-Parameters for Signal Integrity* [6]_. Each
   concept is accomplished by both formulas and executable code, making
   it an invaluable reference in this field. The author of this tutorial
   recommends everyone who simulates or measures RF/microwave devices to
   get a copy.

   **PyBERT.** It's another circuit simulator designed with time-domain
   S-parameter simulation and signal integrity in mind, developed by David
   Banas. See [23]_.

Install
''''''''

To install ``SignalIntegrity``, download the latest ``.zip`` file
at the project's release page using a Web browser::

    https://github.com/Nubis-Communications/SignalIntegrity/releases

As of writing, the latest version was 1.4.1. Once downloaded, unzip
the file and install it locally via :program:`pip`::

    unzip SignalIntegrity-1.4.1.zip
    cd SignalIntegrity-1.4.1

    pip3 install . --user

``SignalIntegrity`` is both a software library and a Tcl/Tk (Tkinter)
GUI application, installed to ``~/.local/bin/`` (or another standard
local path). You should be able to start it via::

    $ SignalIntegrity

If not, you may need to add this local :file:`bin` directory into
your shell's search path.

.. figure:: images/Parallel_Plate_Capacitor_Waveguide/si1.png
   :class: with-border
   :width: 60%

   The classic 1990s Tcl/Tk (Tkinter) GUI may look old-fashioned,
   but it works, and will keep working until the end of time.

Impulse Signal Analysis
'''''''''''''''''''''''''

For the first example, let's examine the circuit's time-domain
response to a short impulse.

**Add an S-parameter defined 2-port network.** Right click on the
empty schematic, choose :guilabel:`Add Part`. In the :guilabel:`Add
Part` diagram, click :menuselection:`Files --> Two Port File`. In
the opened setting window, click
:guilabel:`browse` and select the ``s2p`` S-parameter file created
by our simulation. Click :guilabel:`OK`, and finally click the empty
schematic to place the item. The item can be moved by selecting it
and dragging it.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_1.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_2.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_3.png
   :width: 30%

**Add a voltage pulse generator.** In the :guilabel:`Add Part` dialog,
choose :menuselection:`Generators --> One Port Voltage Pulse Generators`.
In the opened setting window, change the :guilabel:`risetime (s)` property
to "100 ps", and place this item on the schematic.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_4.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_5.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_21.png
   :width: 30%

**Add a series resistor to represent the generator's output impedance.**
In the :guilabel:`Add Part` dialog, choose :menuselection:`Resistors --> Two
Port Resistor`. Place this item on the schematic.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_6.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_7.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_8.png
   :width: 30%

**Add a resistor to ground to represent the input impedance at the end
of the waveguide.** In the :guilabel:`Add Part` dialog, choose
:menuselection:`Resistors --> One Port Resistor to Ground`. Place this
item on the schematic.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_9.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_10.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_11.png
   :width: 30%

**Wire the circuit.**
Right click on the schematic, choose :guilabel:`Add Wire`,
connect the components following this order: pulse generator, series
resistor, DUT Port 1, DUT Port 2, resistor to ground.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_12.png
   :width: 30%

**Add an input probe.** In the :guilabel:`Add Part` window, choose
:menuselection:`Ports and Probes --> Output`. Place this probe on
the wire after the series resistor at the DUT Port 1.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_13.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_14.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_15.png
   :width: 30%

**Add an output probe.** Repeat the above step, add another probe on top
(in parallel) of the resistor to ground, at the DUT Port 2.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_13.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_16.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_17.png
   :width: 30%

**Run simulation.** Click :guilabel:`Calculate` on the menu bar, and choose
:guilabel:`Simulate`. After simulation finishes, it opens a time-domain
line chart. After pressing the "Zoom" (magnifying glass) button and
dragging a rectangle on a part of the waveform of interest, one can finally
see the pulse response of the DUT.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_18.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_19.png
   :width: 30%

**Results.** As expected, the +/- 1 V input signal (+/- 0.5 V with 50 Ω
output and input impedance) has severe overshoots and undershoots due to
port impedance mismatches, while the output shows significant rise
time degradation due to high insertion loss above 2 GHz.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_20.png
   :width: 60%

Eye Diagram Analysis
''''''''''''''''''''''

For the next example, let's examine the circuit's time-domain signal
integrity using a Pseudo-Random Bit Stream (PRBS) generator as the
input, and to plot its output on an eye diagram.

**Delete the voltage pulse generator.** Right-click the voltage
pulse generator we added previously, select :guilabel:`delete`.

.. important::
   Don't select :guilabel:`convert`, it seems buggy and would prevent
   the generation of eye diagram after simulation.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/eye_sim_1.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/eye_sim_2.png
   :width: 30%

**Add a Pseudo-Random Bitstream Generator (PSBG).** In the
:guilabel:`Add Part` dialog, choose
:menuselection:`Generators --> One-Port Voltage PRBS`. Change its
:guilabel:`risetime (s)` to "100 ps". Place this item on the schematic.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/eye_sim_3.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/eye_sim_4.png
   :width: 30%

**Add an eye probe.** In the :guilabel:`Add Part` dialog, choose
:menuselection:`Ports and Probes --> EyeProbe`. Click :guilabel:`Eye
Diagram Configuration`. Set :guilabel:`Measure Eye Parameters` to
``True``, and change the ::guilabel::`Color` to yellow (255, 255, 0)
to improve diagram readability. By default, the eye diagram is
black-and-white. Then close this window.

.. tip::
   Press :kbd:`Enter` to apply changed R, G, or B values. It's
   *not* necessary to click :guilabel:`Save Properties to Global
   Preferences`, by default these options are already applied
   to a single schematic.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/eye_sim_5.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/eye_sim_6.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/eye_sim_7.png
   :width: 30%

**Place the probe and wire the circuit.** Place the eye probe on
the schematic. Wire the eye probe to DUT's Port 2.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/eye_sim_8.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/eye_sim_9.png
   :width: 30%

**Run simulation.** Click :guilabel:`Calculate` on the menu, choose
:guilabel:`Simulate`. After simulation finishes, it opens two plots,
an eye diagram plot and a time-domain line chart.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/transient_sim_18.png
   :width: 30%

**Results.** The eye diagram clearly shows our parallel-plate waveguide
has an extremely poor signal integrity. We will discuss the significance
of this result at the end of this tutorial.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/eye_sim_10.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/eye_sim_11.png
   :width: 30%

**Zoom in.** The time-domain line chart can be zoomed by pressing the
"Magnifying Glass" button and drag a rectangle on a part of the waveform
of interest.  One can see that the overshoots, undershoots and
rise-time degradation is similar to the previous impulse signal
analysis.

S-Parameters Analysis via Qucs-S
""""""""""""""""""""""""""""""""

Qucs is a free and open source circuit simulator for RF/microwave
engineering, it's capable of simulating circuits in both time and
frequency domains.

Install
''''''''

Qucs has a somewhat confusing development history, three different
spinoffs exist.

#. :program:`Qucs`. Development of the original :program:`Qucs` started
   in the early 2000s, but was slowed down over time. Finally in ~2019,
   its underlying GUI framework Qt4 has been discontinued, making it
   uninstallable on most operating systems. Its :program:`Qucsator`` engine
   has good RF simulation features, but limited time-domain capabilities
   with performance and convergence issues.

#. :program:`QucsStudio` is a free-of-charge spinoff of Qucs with
   additional features, but it's **not** free and open source software.

#. :program:`Qucs-S`. The project :program:`Qucs-S` is a fork of
   :program:`Qucs` with a modernized codebase, aiming to support
   multiple SPICE backends for improved time-domain simulations,
   such as `Ngspice`. RF simulations still use the original
   :program:`Qucsator` engine externally before Qucs-S 24.2.0
   (meaning that an installation of the now-unavailable Qucs is
   still needed). After Qucs-S 24.2.0, :program:`Qucsator` is
   now a builtin option.

.. warning::
   For this tutorial, we use Qucs-S 25.1.0. Other Qucs spinoffs
   or older Qucs-S versions have features and user interface
   changes that are in conflict with this tutorial, so one should
   use Qucs-S 25.1.0 or newer when following this tutorial.

The Qucs-S packages in most operating systems are outdated, thus,
Debian, Ubuntu, Fedora and openSUSE users should use the third-party
maintained by Qucs-S projcet developers. It's hosted on Open Build
Service (OBS), installation instructions can be found in the following
link.

* https://software.opensuse.org/download.html?project=home%3Ara3xdh&package=qucs-s

Alternatively, if no package is available, a self-contained AppImage
version is also available as a stop-gap measure before it's packaged.

* https://github.com/ra3xdh/qucs_s/releases/tag/25.1.0

Download ``Qucs-S-25.1.0-linux-x86_64.AppImage``, and grant it
execution permission::

    chmod +x Qucs-S-25.1.0-linux-x86_64.AppImage

.. _qucsator:

Enable Qucsator
'''''''''''''''''''

As previously mentioned, Qucs-S uses Ngspice for simulation by default,
but it lacks some features required for RF/microwave applications that
we need. Hence, we must switch Qucs-S's engine from Ngspice to Qucsator.

**Switch the engine.** Press the drop-down list at the top-right of
the Qucs-S window. By default, it should show `Ngspice`, click
it and change it to `Qucsator`.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-engine-1.png
   :width: 30%

.. important::

   If the option :guilabel:`Qucsator` does not exist (it usually
   disappears when one quits and restarts the application), it
   means you're affected by a bug in the AppImage version of
   Qucs-S. See :ref:`qucsator_bug` for the solution.

S-Parameter Simulation Skeleton
''''''''''''''''''''''''''''''''

To perform frequency-domain analysis in Qucs (Qucs-S), we
need to add several components in multiple steps: adding two
power sources, entering two equations for calculating scalar
S-parameters :math:`S_{11}` and :math:`S_{21}`, adding an
"S-parameter simulation" block with parameters, and inserting
the DUT in the middle between the two ports.

.. important::

   *Before* a filter is synthesized, Qucs-S must be switched to
   use Qucsator as its engine as described in :ref:`qucsator`.
   Otherwise a broken schematic would be synthesized with non-functional
   equations - S-parameter simulations won't work correctly.

**Open the filter wizard.** A trick allows us to quickly setting up a
skeleton - "borrowing" the setup generated by Qucs's filter synthesis
wizard. Click :menuselection:`Tools --> Filter synthesis`.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-1.png
   :width: 60%

**Synthesize a filter.** In the opened dialog, generate the default
low-pass LC filter by pressing :guilabel:`Calculate and put into Clipboard`,
without changing any parameters.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-2.png
   :width: 30%

**Paste it into schematic**. Do *not* close the filter dialog yet.
Now right-click the schematic, click :guilabel:`Paste`. Paste the
synthesized filter circuit onto the page by left-clicking. After
pasting, it's free to close the filter dialog.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-3.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-4.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-5.png
   :width: 30%

**Save the schematic to disk.** Press the :guilabel:`💾` (floppy
disk) button, and choose a path. This is required before starting a
simulation.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-6.png
   :width: 30%

**Run simulation.** Select :menuselection:`Simulation --> Simulate`.
A dialog should pop up without warnings or errors.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-7.png
   :width: 30%

**Add a Cartesian plot.** On the :guilabel:`Main Dock` on the left,
switch from :guilabel:`lumped components` to :guilabel:`diagrams`.
Click the :guilabel:`Cartesian` component.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-8.png
   :width: 30%

**Plot dbS11.** Click the schematic to place the component. In
the dialog, double-click the scalar ``dbS11`` to add it into the
:guilabel:`Graph` item list, then press :guilabel:`OK`.

.. important::

   The scalars ``dBS11`` and ``dbS21`` are not to be confused with the
   complex S-parameters ``S[1,1]`` and ``S[2,1]``. In particular, the
   Smith chart won't work with a scalar value.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-9.png
   :width: 30%

**Plot dbS21**. Repeat the same steps above, add another
:guilabel:`Cartesian` component to the schematic. Click the schematic to
place it. In the dialog, double-click the scalar ``dBS21`` (not the
complex ``S[2,1]``) to add this array into the :guilabel:`Graph` item
list. Click :guilabel:`OK`.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-10.png
   :width: 30%

**Add a Smith chart.** Click the :guilabel:`Smith Chart` component on the
:guilabel:`Main Dock`.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-11.png
   :width: 30%

**Plot S[1,1].** Click the schematic to place the component. In the
dialog, double-click ``S[1,1]`` (not dBS11) to add this array into the
:guilabel:`Graph` item list. Click "OK".

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-12.png
   :width: 30%

.. hint::
   All subsequent mouse clicks will keep adding components to the schematic.
   To stop, press :kbd:`Esc` or click the :guilabel:`Select` button (cursor
   icon) on the menu bar.

   .. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-13.png
      :width: 30%

**Plotting.** Now we have a fully-functional S-parameter simulation skeleton
with S-parameters on two line chart and a Smith chart.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-14.png
   :width: 30%

**Delete the text label.** The next step is to delete the filter, leaving
only the simulation skeleton. First, select the text label :guilabel:`Bessel
low-pass filter 1 GHz cutoff, ...` by left-clicking it and pressing the
:kbd:`Delete` key on the keyboard (or by right-cilking it and pressing
the :guilabel:`Delete` option).  This text label would be useless and
misleading if it's left here.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-15.png
   :width: 30%

**Delete the inductor, capacitors, and wire segments.** Select the
inductor and delete it by pressing the :kbd:`Delete` key on the keyboard.
Delete the capacitors using the same procedure.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-16.png
   :width: 30%

.. hint::
   It's difficult to delete a wire segment without highlighting the
   entire wire.  If you have trouble selecting a segment, try dragging
   the mouse to highlight a rectangle that contains only that segment.

   .. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-17.png
      :width: 30%
   .. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-18.png
      :width: 30%

**Clean Skeleton Obtained.** After all useless components are wires have
been deleted, the simulation skeleton is clean to use.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-19.png
   :width: 30%

Two-Port S-Parameter Analysis
''''''''''''''''''''''''''''''

**Add an S-parameter-defined component.** On the left :guilabel:`Main Dock`,
switch from :guilabel:`diagram` (or :guilabel:`lumped components`) to
:guilabel:`file components`. Click :guilabel:`2-port S-parameters`, place
it on the schematic.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-20.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-21.png
   :width: 30%

**Wire component together.** Click the :guilabel:`Wire` icon on the menu
bar. Connect the DUT's Port 1 to the left, connect Port 2 to the right,
and wire the Reference Port downwards.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-22.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-23.png
   :width: 30%

**Add a ground.** Click :guilabel:`Ground Node` on the menu bar, add a
ground below wire to the reference port.

.. hint::
   Don't forget to press :kbd:`Esc` to exit from insert mode, otherwise
   every mouse click results in more :guilabel:`Ground Nodes` on the
   schematic.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-24.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-25.png
   :width: 30%

**Set S-parameter path.** Double click the file component :guilabel:`X1`.
In the opened dialog, press the :guilabel:`...` button to change the
path of the Touchstone file that backs this parameter-defined component.
Pick the Touchstone file created by the openEMS simulation. Also, uncheck
the :guilabel:`Show` option, since the file path is extremely long and
visually cluttering. Press :guilabel:`OK`.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-26.png
   :width: 30%

**Move Component Text.** One may also want to adjust the position of
the text label :guilabel:`X1` to be directly above the component box
(since the space occupied by the file path has gone) to improve readability.
This can be done by right-clicking the component, select :guilabel:`Move
Component Text`, then drag the text to a new position.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-27.png
   :width: 30%

**Run simulation**. Select :guilabel:`Simulation` and click
:guilabel:`Simulate`. A dialog should pop up without warnings or errors.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-7.png
   :width: 30%

**Results**. Now the S-parameters from our simulation should appear on
the Smith and line charts.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-28.png
   :width: 30%

Comparison with Microstrip
'''''''''''''''''''''''''''

In the analysis of RF/microwave devices, We often need to compare the
behavior of different circuits. Using the microstrip as an example (as
the limiting case of a parallel-plate waveguide when the lower plate has
an infinite size), we perform a simple comparison here.

**Select and move the existing circuit.** Drag the mouse and select the
entire circuit, move it upwards to free up some space. Press :kbd:`Control-C`
to copy this circuit, then press :kbd:`Control-V` to paste a new instance
and place it below the original.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-microstrip-1.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-microstrip-2.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-microstrip-3.png
   :width: 30%

**Delete the extra file-defined component**. Delete X2 and its ground
connect.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-microstrip-4.png
   :width: 30%

**Add a microstrip.**  On the left :guilabel:`Main Dock`, switch to
:guilabel:`transmission lines`. Click :guilabel:`Microstrip Line`,
and place it between Port 3 and Port 4.
Change its :guilabel:`W` (width) and :guilabel:`L` to ``100 mm``. Apply the
numer change by pressing :kbd:`Enter`.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-microstrip-5.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-microstrip-6.png
   :width: 30%

**Add a substrate that supports the microstrip.** On the left
:guilabel:`Main Dock`, click :guilabel:`Substrate`, and place it
at the bottom of the schematic. Change parameter :guilabel:`er` to ``1.001``
(don't use ``1.0``, the Qucs-S's internal formula is singular here).
Change parameter :guilabel:`h` (height) to ``16 mm``.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-microstrip-7.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-microstrip-8.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-microstrip-9.png
   :width: 30%

**Add new equations for scalar S33 and S43**. Double-click the
:guilabel:`Equation` component.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-microstrip-11.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-microstrip-10.png
   :width: 30%

Change its formulas to the following::

    dBS21 = dB(S[2,1])
    dBS11 = dB(S[1,1])
    dBS33 = dB(S[3,3])
    dBS43 = dB(S[4,3])

.. important::
   Make sure ``S[3,3]`` and ``S[4,3]`` matches the actual port numbers
   on the schematics. If ports have been repeatedly deleted and inserted
   into the schematic, they could be renumbered. If so, change the
   port numbers back to ``3`` and ``4``.

**Run simulation.** Select :guilabel:`Simulation` and click :guilabel:`Simulate`.
A dialog should pop up without warnings or errors.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-filter-7.png
   :width: 30%

**Plots.** Double-click the original :math:`S_{11}` chart, add
:guilabel:`dBS33` to the graph by double-clicking this item. Similarly,
do the same to add :guilabel:`dbS41` to the original `S_{21}` chart, and
finally add :guilabel:`S[3,3]` to the Smith chart.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-microstrip-12.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-microstrip-13.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-microstrip-14.png
   :width: 30%

**Results.** By setting this Qucs-S simulation, we can compare the frequency
response of a parallel-plate waveguide against a microstrip transmission line.
From the simulation result, we find that their behaviors are significantly
different. Both transmission lines have a resonance at 1.5 GHz, but their
similarity ends here. At high frequencies, this parallel-plate waveguide has
extremely high losses, while the microstrip transmission line is lossless.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-microstrip-15.png
   :width: 60%

We will discuss the significance of this result at the end of this tutorial.

Field Dumps Analysis via ParaView
"""""""""""""""""""""""""""""""""

ParaView is a visualization program commonly used in the High-Performance
Computing (HPC) and scientific computing world. One can install it using
the operating system's package manager.

In openEMS, we rely on ParaView to visualize raw electromagnetic fields
created during the simulation. For troubleshooting malfunctioning simulations
or understanding the physical behavior of a structure, this is especially helpful
as one can identify the problematic region directly by visualization.

We continue our total current density visualization example, introduced
as an optional step in :ref:`fielddump`. If the simulation finishes with
field dump enabled, a series of `vtr` files (which is a type of the VTK
file format) is created under the simulation directory.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/paraview-1.png
   :width: 30%

**Open ParaView.** To visualize these results, open ParaView first.
Click :menuselection:`Files --> Open`. Navigate to the simulation
data directory, open the `vtr` file series.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/paraview-2.png
   :width: 30%

**Apply cell/point array for visualization.** Make sure ``RotH-Field``
appears in the :guilabel:`Cell/Point Array` list on the bottom left
of the ParaView window, and is checked. Click :guilabel:`Apply`. By
default, the visualization shows an empty rectangle because the displayed
physical quantity is set to :guilabel:`Solid Color` without any data.
Click :guilabel:`Solid Color` and change it to :guilabel:`RotH-Field`.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/paraview-3.png
   :width: 30%

.. image:: images/Parallel_Plate_Capacitor_Waveguide/paraview-4.png
   :width: 30%

**Play animated visualization.** Now it's possible to see the
current density by clicking the :guilabel:`Play` button to see an
animated visualization of electromagnetic wave propagation.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/paraview-5.png
   :width: 30%

**Rescale the color-grading.** However, ParaView can still render
this field dump incorrectly due to a problem in color-grading. If
ParaView's color-grading scale uses a small values for reference
(as the field at t = 0 is zero), all fields (or current) injected
by the excitation source in later timesteps can be rendered as a deep
red as the color saturates, making them indistinguishable. This
happens by default, as shown in the following diagram. We can
solve this problem by clicking :guilabel:`Rescale to Visible Data Range`.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/paraview-6.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/paraview-7.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/paraview-8.png
   :width: 30%

**Color scale sensitivity to timestep.** On the other hand, if large
values are used for references, the beginning of the visualization
works okay. But in later timesteps, colors will be very shallow and
difficult to see. Thus the color-grading scale is quite sensitive to
the timestep at which the :guilabel:`Rescale to Visible Data Range`
button is pressed.

**Manual color scale.** Another problem is that the excitation port
always have strong fields and currents, so as long as ParaView's
color-grading is normalized to the magnitudes around a port, all currents
and fields
at other locations may appear dark and weak. Thus, manually overriding
the color-grading may be necessary. To do so, press :guilabel:`Rescale
to Custom Data Range`, enter a new value (such as `0.001`) and press
:guilabel:`Rescale and disable automatic rescaling`.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/paraview-10.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/paraview-9.png
   :width: 30%

**Logarithmical color scale.** For good visualization, an alternative
solution is to color-grade the values logarithmically. It can be done
by clicking :guilabel:`Edit Color Map`. In the :guilabel:`Color Map
Editor` on the right, check :guilabel:`Use log scale when mapping data
to colors`. A warning message may immediately appear, as the log of 0
is undefined, but it's safe to ignore.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/paraview-11.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/paraview-12.png
   :width: 30%

**Result.** In this simulation, we find that rescaling at timestep
55 and enabling logarithmical color map allow us to obtain a
satisfactory video below.

.. video:: images/Parallel_Plate_Capacitor_Waveguide/curlH.webm

.. important::

   To correctly visualize the fields in ParaView, one must apply
   the corresponding Cell/Point Array, select the variable that
   represents the physical quantity, rescale the color-grading
   (and optionally enable logarithmical scale) to match the data
   range. Otherwise, an empty box appears when the array is not
   selected. A solid color appears when the color map's data range
   is saturated.

   Also, 3D dump boxs are tricky to visualize. By default it's
   rendered as an empty box. It's possible to render a 2D slice
   of the data, or possibly a 3D vector field. But it's beyond
   the scope of this tutorial.

.. seealso::
   ParaView is a large program used in many scientific applications,
   it's impossible to cover all of its aspects here. See the full
   manual [13]_ for usage.

Experimental Validation and Discussion
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Motivation
"""""""""""

As shown in earlier sections using different methods, the presented
frequency response of this parallel-plate waveguide has anomalous
losses at high frequencies, around 20 dB at 2 GHz. This behavior
runs against our expectation for lumped capacitors and transmission
lines, both are essentially lossless.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/s21_db_sim_skrf.svg
   :width: 49%

Unusual results like this one may indicate a modeling mistake that
either causes unphysical behavior or the excitation of theoretical
but theoretically-possible but unrealistic effects. Alternatively,
it may reveal actual physics rarely discussed in circuit textbooks,
so it can be “new” to us.

Experiment
"""""""""""

Thus, this result requires validation. To test the validity of
this simulation result empirically, the author of this tutorial performed
an experiment using a self-built parallel-plate waveguide test fixture.
The fixture is
made of two brass plates with a thickness of 0.5 mm. Two SMA coax
connectors are mounted on the left and right edges. Conveniently, these
special connectors have lengthened inner conductors (meant for
connections to air lines). The connectors are screwed onto the upper
plate, while their inner conductors touch the lower plate below.
The plates are held in position using Nylon screws at four corners,
with a spacing of 16 mm.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/experiment-fixture.svg
   :width: 49%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/experiment_waveguide_1.jpg
   :width: 49%

Air is the only dielectric of this parallel-plate waveguide, so this
test fixture's frequency response is sensitive to nearby conductors and
insulators. To minimize interference, plastic zip ties were used
to suspend the fixture in free air, ensuring a reasonable clearance
from nearby objects. Care was made to minimize the zip ties' passing
through the gap between the plates.

Its frequency response is measured by a 2-port Vector Network Analyzer
(VNA) from 100 MHz to 4.4 GHz.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/experiment_waveguide_2.jpg
   :width: 49%

.. note::
   This experiment was meant to be only a quick qualitative test, not an
   exact quantitative test, so the VNA was *not* fully calibrated. SOLT
   calibration was performed, but the Open, Short, Load, and Thru standards
   were all assumed ideal, which introduces errors at high frequencies,
   and may affect the flatness of frequency response across the spectrum.
   But the *trend* remains largely unaffected.

Results
"""""""

A Qucs-S simulation is used to visualize the S-parameters from both
the simulation and experiment for comparison. This is done by setting
a simulation with
two S-parameter-defined components, one from the simulation, and another
from measurement. The obtained result is the following.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/experiment-qucs-1.png
   :width: 49%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/experiment-qucs-2.png
   :width: 49%

As we can see, both the :math:`S_{11}` and :math:`S_{21}` magnitudes
from the simulation and measurement are correlated, including the
resonance at 500 MHz, the resonance at 1.5 GHz, and the 20 dB loss
around 2 GHz. On the Smith chart, the :math:`S_{11}` phases also
show good correlations between from 100 MHz to 1.3 GHz.

We also use ``scikit-rf`` to compute the equivalent step-input TDR
curves of the simulation and experiment. The resulting plot is shown
below for comparison. Both curves show a high-impedance discontinuity at the
first port, and another high-impedance reflection at the second
port (with a weaker reflection). The peak impedance magnitudes are
quantitatively different in the simulation and experiment, 100 Ω and
140 Ω respectively. The exact number is sensitive to the port/connector
transition and geometry, and we didn't model the SMA RF connectors.
But both results are qualitatively similar.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/experiment-tdr.svg
   :width: 49%

Therefore, the observed high-frequency loss is a measurable effect of a
real parallel-plate waveguide. What is its origin? We can make several
educated guesses:

1. **Non-TEM modes.** Ordinary two-conductor transmission lines and
   circuits operate in TEM (Transverse Electro-Magnetic) mode, in
   which the directions of electric and magnetic fields are
   orthogonal to the direction of wave propagation. But if the
   operating frequency is sufficiently high, high-order waveguide
   modes may be excited, resulted in multimoding. This often occurs
   if the separation is large enough to allow the electric field
   across the conductors to vary. This does not necessarily
   increase the loss, but certainly is the source of confusing data.

   The analytic solution of the TM01 or TE01 mode in a parallel-plate
   waveguide is given by :math:`\lambda = 2d / m`, :math:`f_c =
   v / \lambda`, in which :math:`d` is the separation between the
   plates, :math:`m = 1, 2, 3...` is the waveguide mode index,
   :math:`v` is the speed of light in the medium, :math:`\lambda` is
   the cutoff wavelength, and :math:`f_c` is the cutoff frequency. [9]_
   Using this calculation, we find that multimoding appears at 9.3 GHz
   and above, so it indeed plays a small role.

2. **Radiation losses.** Ordinary transmission lines are either fully
   enclosed (like a rectangular waveguide) or has a tiny separation
   distance to the ground plane much smaller than the wavelength.
   But here, our plate separation is as large as 16 mm, which is
   a significant portion of the signal wavelength, more than half at
   10 GHz (30 mm). Radiation potentially escapes from this wavelength.

3. **Poor port termination.** In ordinary transmission lines such
   as microstrips, the line has a length much greater than its width, and
   the width is comparable to the width of the port. The termination
   resistance of the port also matches the characteristic impedance of
   the line. Thus, the port perfectly absorbs the electromagnetic energy
   with little secondary reflection. But in this demo, the port is both
   physically and electrically mismatched on purpose. Thus it can't
   fully capture the energy of the incoming signal. These mismatches
   can also exacerbate non-TEM modes and radiation losses near the
   ports in many cases, not just those found in analytic solutions.

.. important::
   Nearly everything can be an RF waveguide or resonator
   when the excitation frequency is high enough. For example,
   the book [10]_ lists four possible high-order microstrip modes.
   In fact, unrealistic excitation of high-order modes is fairly common
   in EM field solvers as unphysical excitation signals or
   port placements are easily created. Be sure to watch for it,
   if the simulation results don't seem to make sense.

Impedance Transformation
"""""""""""""""""""""""""

Another notable effect is the resonance at 500 MHz and 1.5 GHz.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/s11_db_sim_skrf.svg
   :width: 49%

Curiously, even the microstrip transmission line simulation in
:program:`Qucs-S` shows a resonance at 1.5 GHz as well.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-microstrip-15.png
   :width: 60%

Is it an intrinsic resonance caused by the waveguide or transmission line
itself, or an artifact from the mismatched ports?

This can be revealed by a simple wavelength calculation. In vacuum or air,
the wavelength of an electromagnetic signal at 500 MHz is approximately 600 mm.
At 1.5 GHz, the wavelength is approximately 200 mm. Both are integer
multiples of the 100 mm length of our parallel-plate waveguide,
suggesting it's a quarter-wave impedance transformer effect.

In transmission line theory, a remarkable observation is the following:
if the length of a lossless transmission line is a multiple of
:math:`\lambda / 2`, its input impedance will be *exactly* equal to the
load impedance, regardless of its characteristic impedance!

The implication is that when a mismatched line with a length of
:math:`\lambda / 2` connects a transmitter and a receiver, and if
both ports have real and identical impedances (i.e. they're
mismatched in the same way), *any* transmission line can be used
regardless of its characteristic impedance.
We can show this from a simple argument. Consider a 50 Ω transmission
line, connected to a 600 Ω receiver. If we measure its impedance using a
50 Ω transmitter at a fixed frequency, we would find its reflection
coefficient has a phase shift that changes depending on the length of
the line.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/half-wave-1.svg
   :width: 49%
   :alt: A 50 Ω transmitter is connected to a 600 Ω receiver via a 50 Ω
         transmission line with a length of 0.2λ, its input reflection coefficient
         is 0.8∠-144.0°, its input impedance is 16.8∠-74.1° Ω.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/half-wave-2.svg
   :width: 49%
   :alt: A 50 Ω transmitter is connected to a 600 Ω receiver via a 50 Ω
         transmission line with a length of 0.4λ, its input reflection coefficient
         is 0.8∠72.0°, its input impedance is 68.5∠80.0° Ω.

But when the line is :math:`\lambda / 2`, the phase
shift is zero, since the signal has shifted 180 degrees going forward,
and 180 degrees going back. As a result, the measured reflection
coefficient has a phase angle of 0 degrees. Thus, the input impedance looking
into the beginning of the transmission line is exactly 600 Ω, without
an imaginary part.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/half-wave-3.svg
   :width: 49%
   :alt: A 50 Ω transmitter is connected to a 600 Ω receiver via a 50 Ω
         transmission line with a length of 0.5λ, its input reflection coefficient
         is 0.8∠0.0°, its input impedance is 600.0∠0.0° Ω.

If we run the same 50 Ω transmission line using a 600 Ω transmitter
instead, we find the same conclusion. Due to phase shifts of the
reflection coefficient, the measured impedance changes dramatically
depending on how long the transmission line is.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/half-wave-4.svg
   :width: 49%
   :alt: A 600 Ω transmitter is connected to a 600 Ω receiver via a 50 Ω
         transmission line with a length of 0.2λ, its input reflection coefficient
         is 1.0∠-176.9°, its input impedance is 16.8∠-74.1° Ω.
.. image:: images/Parallel_Plate_Capacitor_Waveguide/half-wave-5.svg
   :width: 49%
   :alt: A 600 Ω transmitter is connected to a 600 Ω receiver via a 50 Ω
         transmission line with a length of 0.4λ, its input reflection coefficient
         is 1.0∠167.2°, its input impedance is 68.5∠80.0° Ω.

But the load impedance remains constant regardless of the output
impedance of the transmitter. When the line is :math:`\lambda / 2`,
we also get a *perfect* impedance match, as the overall input
impedance of the :math:`\lambda / 2` transmission line is,
exactly equal to the 50 Ω load again.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/half-wave-6.svg
   :width: 49%
   :alt: A 600 Ω transmitter is connected to a 600 Ω receiver via a 50 Ω
         transmission line with a length of 0.5λ, its input reflection coefficient
         is 0.0∠0.0°, its input impedance is 600.0∠0.0° Ω.

Therefore, we can conclude the 500 MHz and 1500 MHz resonances are
effects of the mismatched ports resonating with the transmission line's
length, not the intrinsic characteristics of the parallel-plate waveguide.

.. note::
   **Quarter-Wave Impedance Transformer** - This half-wave
   transmission line phenomenon is a special case of the more
   powerful quarter-wave impendance transformer. Learn more
   about it at [8]_.

   **Port Discontinuities** - This shows why an optimized
   port transition, as well as the use of de-embedding or calibration,
   are crucial in both simulations and measurements. Otherwise
   port mismatches can easily generate misleading results.

Troubleshoot
^^^^^^^^^^^^^

.. _wayland:

Notes for Wayland Users
"""""""""""""""""""""""""

Due to a technical limitation, Wayland desktop users may need to run
AppCSXCAD under X11 by unsetting the ``WAYLAND_DISPLAY`` environmental
variable::

    $ env WAYLAND_DISPLAY= AppCSXCAD Parallel_Plate_Capacitor.py.xml

Otherwise, significant performance degradation may be experienced
due to this repeating error on every draw: "GLEW could not be
initialized: Unknown Error." The problem originated from the
internal implementations of GLEW and VTK, namely, GLEW attempts to
initialize X11 GLX when it's compiled for X11 and Wayland at the same
time. The fix requires changing VTK, so openEMS has little control in
this matter.

.. note::
   See https://github.com/thliebig/AppCSXCAD/issues/11 for details.

.. _unused_plate:

Warning: Unused primitive (type: Box) detected in property: plate!
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

If you see the following warnings in the simulation::

    Create FDTD engine (compressed SSE + multi-threading)
    Warning: Unused primitive (type: Box) detected in property: plate!
    Warning: Unused primitive (type: Box) detected in property: plate!
    Running FDTD engine... this may take a while... grab a cup of coffee?!?
    [@        4s] Timestep:         1666 || Speed:   71.4 MC/s (2.401e-03 s/TS) || Energy: ~6.79e-22 (-68.99dB)
    Time for 1666 iterations with 171500.00 cells : 4.00 sec
    Speed: 71.42 MCells/s

It indicates the metal plates are not actually used in the simulation.

This is likely a meshing problem. All CSXCAD structure must pass at least a
single mesh line, including zero-thickness structures like thin metal plates.
Structures that stay in the middle of two mesh lines can't be simulated.

To fix this problem, add mesh lines at the exact coordinate of zero-thickness
plates::

    # zero-thickness metal plates need mesh lines at their exact levels
    mesh.AddLine('z', [-8, 8])

.. important::
   A zero-thickness plate must cross or align exactly with at least one mesh
   line, otherwise it can't be simulated.

.. _unused_excite:

Warning: Unused primitive (type: Box) detected in property: port_excite_1!
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

If you see the following warnings in the simulation::

    Create FDTD engine (compressed SSE + multi-threading)
    Warning: Unused primitive (type: Box) detected in property: port_excite_1!
    Running FDTD engine... this may take a while... grab a cup of coffee?!?
    [@        4s] Timestep:         1588 || Speed:   72.0 MC/s (2.519e-03 s/TS) || Energy: ~0.00e+00 (- 0.00dB)
    [@        8s] Timestep:         3882 || Speed:  104.0 MC/s (1.744e-03 s/TS) || Energy: ~0.00e+00 (- 0.00dB)

It indicates the excitation port is not actually used in the simulation, this
is further confirmed by the energy of ``~0.00e+00``: it means the port is disabled
so it didn't inject any energy into the simulation box.

This is likely a meshing problem. All CSXCAD structure must pass at least a
single mesh line, including zero-thickness structures like the ports.
Structures that stay in the middle of two mesh lines can't be simulated.

For example, because we used the 1/3-2/3 rule around the edges of the metal
plates, there's no mesh line at the left and right edge of the metal plates.
Thus the following code won't work::

    port[0] = fdtd.AddLumpedPort(1, z0, [-50, -2.5, -8], [-50, 2.5, 8], 'z', excite=1)
    port[1] = fdtd.AddLumpedPort(2, z0, [ 50, -2.5, -8], [ 50, 2.5, 8], 'z', excite=0)

As a compromise, we can shift the port's location to the nearest mesh lines
along the X axis instead, this may introduce a small error as the port's
measurement plane has shifted. But these issues is negligible for this demo::

    port[0] = fdtd.AddLumpedPort(1, z0, [-50 + 1/3 * res, -2.5, -8], [-50 + 1/3 * res, 2.5, 8], 'z', excite=1)
    port[1] = fdtd.AddLumpedPort(2, z0, [ 50 - 1/3 * res, -2.5, -8], [ 50 - 1/3 * res, 2.5, 8], 'z', excite=0)

.. important::
   A port must cross or align exactly with at least one mesh line, otherwise
   it can't be simulated.

An alternative solution is to create a mesh line aligned with the port,
using the :meth:`CSX.ContinuousStructure.AddLine` function. This violates
the 1/3-2/3 rule, but is acceptable for long and narrow structures like
a transmission line with weak fringe fields on both ends.

.. _voltage_integral_error:

CalcVoltageIntegral: Error, only a 1D/line integration is allowed
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

If the openEMS output is flooded with the following error message::

    Engine_Interface_FDTD::CalcVoltageIntegral: Error, only a 1D/line integration is allowed
    Engine_Interface_FDTD::CalcVoltageIntegral: Error, only a 1D/line integration is allowed
    Engine_Interface_FDTD::CalcVoltageIntegral: Error, only a 1D/line integration is allowed
    Engine_Interface_FDTD::CalcVoltageIntegral: Error, only a 1D/line integration is allowed
    Engine_Interface_FDTD::CalcVoltageIntegral: Error, only a 1D/line integration is allowed
    Engine_Interface_FDTD::CalcVoltageIntegral: Error, only a 1D/line integration is allowed

It means that openEMS is not able to calculate the voltage at a port
because the port is located at an ill-defined position. This happens
if the start and stop coordinates are different (i.e. not a 1D port),
but the size is smaller than a single mesh cell (i.e. when the port
is built from its start coordinate to its stop coordinate on each axis,
it does not overlap with at least two mesh lines).

For example, the following port is functional because it's strictly
a 1D port, with identical start and stop coordinates::

    port[0] = fdtd.AddLumpedPort(1, z0, [any_x, any_y, -8], [any_x, any_y, -8], 'y', excite=1)
    port[1] = fdtd.AddLumpedPort(2, z0, [any_x, any_y, -8], [any_x, any_y, -8], 'y', excite=0)

The following port is also functional because the 2D port passes
(overlaps with) at least two mesh lines when it's built from Z = -8
to Z = 8::

    port[0] = fdtd.AddLumpedPort(1, z0, [any_x, any_y, -8], [any_x, any_y, 8], 'z', excite=1)
    port[1] = fdtd.AddLumpedPort(2, z0, [any_x, any_y, -8], [any_x, any_y, 8], 'z', excite=0)

The following port is also functional, because although there
is no mesh line at the stop position Z = 8.1, but the 2D port has
already crossed at least one mesh cell (two mesh lines) when it's
built from Z = -8 to its stop coordinate::

    port[0] = fdtd.AddLumpedPort(1, z0, [any_x, any_y, -8], [any_x, any_y, 8.1], 'z', excite=1)
    port[1] = fdtd.AddLumpedPort(2, z0, [any_x, any_y, -8], [any_x, any_y, 8.1], 'z', excite=0)

But the following port is not functional, because the 2D port
does not cross a single mesh cell (two mesh lines) when it's
built from Z = -8 to Z = -7.9. Although there's a mesh line at Z = -8,
there is no mesh line between Z = -8 and Z = -7.9::

    port[0] = fdtd.AddLumpedPort(1, z0, [any_x, any_y, -8], [any_x, any_y, -7.9], 'z', excite=1)
    port[1] = fdtd.AddLumpedPort(2, z0, [any_x, any_y, -8], [any_x, any_y, -7.9], 'z', excite=0)

To fix the problem, either redefine the port with the correct
coordinates, or to add additional mesh lines.

.. important::
   On each axis, a port must either be one-dimensional and aligned to a
   mesh line, or two-dimensional and crosses (or overlaps) with two
   mesh lines (a single mesh cell). Otherwise it can't be simulated.

   Furthermore, a port's excitation direction must be two-dimensional.
   If the port excites the Z direction, it must have a length on the Z axis,
   satisfying the two constraints mentioned.

.. _qucsator_bug:

No :guilabel:`Qucsator` in Qucs-S
"""""""""""""""""""""""""""""""""""

Unfortunately, the AppImage version of Qucs-S has a bug that prevents
Qucs-S from finding Qucsator. Each time an AppImage is launched, it's
extracted to a random directory in `/tmp`, so Qucsator works exactly
once on each machine - after closing and relaunching it, the previously
configured `Qucsator` path would become invalid and can no longer
be found. Thus it disappears from the engine list.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-engine-2.png
   :width: 30%

To work around this bug, the path to `Qucsator` must be configured
manually each time.


Programmatic Solution
'''''''''''''''''''''''

We can solve this problem programmatically by clearing the `Qucsator`
variable setting from the Qucs-S configuration
file, usually located at ``~/.config/qucs/qucs_s.conf``::

    sed -i '/Qucsator=/d' ~/.config/qucs/qucs_s.conf

After quitting Qucs-S, running the command above, and restarting
Qucs-S, the `Qucsator` option should return to the engine list.

GUI Solution
''''''''''''''

For completeness, here's how to do reconfige the path to Qucsator
manually. To do so, click :guilabel:`Simulation` and choose
:guilabel:`Simulators Settings...`

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-engine-3.png
   :width: 30%

In the dialog, click :guilabel:`Select` under the :guilabel:`Qucsator
Settings` section.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-engine-4.png
   :width: 30%

Navigate the file picker to ``/tmp``, right-click an empty area of the
file view, select :guilabel:`Show Hidden Files`. A directory with a suffix
named ``/tmp/.mount_Qucs-`` should appear, it has a random prefix that
varies every time it's launched.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-engine-5.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-engine-6.png
   :width: 30%

Enter this directory, enter directory ``usr``, then enter ``bin``, find
the executable named ``qucsator_rf``. Select the executable and press
:guilabel:`Open` to pick it.  Finally, press :guilabel:`Apply changes`
in the :guilabel:`Setup simulators executable location` dialog.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-engine-7.png
   :width: 30%
.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-engine-8.png
   :width: 30%

After doing so, the :guilabel:`Qucsator` option should reappear in
the engine drop-down list. Switch the engine from :guilabel:`Ngspice`
to :guilabel:`Qucsator` by clicking it.

.. image:: images/Parallel_Plate_Capacitor_Waveguide/qucs-engine-1.png
   :width: 30%

Bibliography
^^^^^^^^^^^^^

.. [1] T. S. Bird, `Definition and Misuse of Return Loss [Report of the
  Transactions Editor-in-Chief], <https://ieeexplore.ieee.org/document/5162049>`_
  in IEEE Antennas and Propagation Magazine, vol. 51, no. 2, pp. 166-167,
  April 2009, doi: 10.1109/MAP.2009.5162049.

.. [2] The Unknown Editor, `Microwaves101: Loss or Gain?
  <https://www.microwaves101.com/encyclopedias/loss-or-gain>`_

.. [3] The Unknown Editor, `Microwaves101: SNP Format
   <https://www.microwaves101.com/encyclopedias/snp-format>`_

.. [4] Russell Carroll, `How to Use S-parameter Models in PSpice
   <https://emisleuth.com/Articles/How_to_Use_S-parameter_Models_in_PSpice_5-28-2024.pdf>`_

.. [5] Giles Atkinson, `Reply to "How to perform transient simulation with S parameter file"
   <https://sourceforge.net/p/ngspice/discussion/120973/thread/3bce0daf1c/#fffd>`_

.. [6] Peter J. Pupalaikis, `S-parameters for Signal Integrity.
   <https://www.cambridge.org/core/books/sparameters-for-signal-integrity/2FD44B994CD054E4B844B358E8903A09>`_
   Cambridge University Press, 2020.

.. [7] Ted Yapo, `A Note on Gaussian Steps in openEMS.
   <https://cdn.hackaday.io/files/1656277086185568/gaussian_step_v11.pdf>`_

.. [8] The Unknown Editor, `Quarter-wave Tricks
   <https://www.microwaves101.com/encyclopedias/quarter-wave-tricks>`_

.. [9] Michael Steer. `Microwave and RF Design II - Transmission Lines,
   6.3: Parallel-Plate Waveguide
   <https://eng.libretexts.org/Bookshelves/Electrical_Engineering/Electronics/Microwave_and_RF_Design_II_-_Transmission_Lines_(Steer)/06%3A_Waveguides/6.03%3A_Parallel-Plate_Waveguide>`_

.. [10] Michael Steer. `Microwave and RF Design II - Transmission Lines,
   4.6: Microstrip Operating Frequency Limitations
   <https://eng.libretexts.org/Bookshelves/Electrical_Engineering/Electronics/Microwave_and_RF_Design_II_-_Transmission_Lines_(Steer)/04%3A_Extraordinary_Transmission_Line_Effects/4.06%3A_Microstrip_Operating_Frequency_Limitations>`_

.. [11] Andreas Rennings, Elektromagnetische Zeitbereichssimulationen innovativer
   Antennen auf Basis von Metamaterialien. PhD Thesis, University of Duisburg-Essen,
   2008, pp. 76, eq. 4.77

.. [12] scikit-rf. `scikit-rf Manual <https://scikit-rf.readthedocs.io/en/latest/>`_.

.. [13] scikit-rf. `ParaView Manual <https://docs.paraview.org/>`_.

.. [14] Hatab, Ziad, Michael Ernst Gadringer, and Wolfgang Bosch.
   `Indirect Measurement of Switch Terms of a Vector Network Analyzer with
   Reciprocal Devices. <https://arxiv.org/abs/2306.07066>`_

.. [15] scikit-rf. `IEEEP370 Deembedding
   <https://scikit-rf.readthedocs.io/en/latest/examples/networktheory/IEEEP370%20Deembedding.html>`_

.. [16] scikit-rf. `API / calibration (skrf.calibration.calibration)
   <https://scikit-rf.readthedocs.io/en/latest/api/calibration/index.html>`_

.. [17] scikit-rf. `Time Domain and Gating
   <https://scikit-rf.readthedocs.io/en/latest/examples/networktheory/Time%20Domain.html>`_

.. [18] Anritsu. Simberian.
   `S-parameter Quality Metrics and Analysis to Measurement Correlation
   <https://www.simberian.com/Presentations/VS43_S-Parameter_Quality_Metrics_And_Analysis_To_Measurement16x9.pdf>`_. DesignCon 2015.

.. [19] scikit-rf. `Single-Ended S-parameters quality checking
   <https://scikit-rf.readthedocs.io/en/latest/examples/networktheory/IEEEP370%20Deembedding.html#Single-Ended-S-parameters-quality-checking>`_

.. [20] SignalIntegrity project. `Enforce Both Passivity and Reciprocity
   <https://nubis-communications.github.io/SignalIntegrity/SignalIntegrity/App/Help/Help.html.LyXconv/Help-Subsubsection-6.html#Control-Help:Enforce-Both-Passivity-and-Reciprocity>`_

.. [21] Anto Davis and Steve Sandler. `The 2-Port Shunt-Thru Measurement and the Inherent Ground Loop
   <https://www.signalintegrityjournal.com/articles/1202-the-2-port-shunt-thru-measurement-and-the-inherent-ground-loop>`_, in Signal Integrity Journal.

.. [22] Brian Walker. `Make Accurate Impedance Measurements Using a VNA
   <https://www.mwrf.com/technologies/test-measurement/article/21849791/copper-mountain-technologies-make-accurate-impedance-measurements-using-a-vna>`_, in Microwave & RF website.

.. [23] PyBERT. `Project Repository.
   <https://github.com/capn-freako/PyBERT>`_
