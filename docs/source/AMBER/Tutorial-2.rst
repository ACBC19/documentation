This tutorial shows how to use the SIRAH force field to perform a coarse grained (CG) simulation of a
double stranded DNA embedded in explicit solvent (called WatFour, WT4). The main references for
this tutorial are:`Dans et al. SIRAH DNA <https://pubs.acs.org/doi/abs/10.1021/ct900653p>`_ (latest parameters are those reported in: `Darré et al. WAT4?) <https://pubs.acs.org/doi/abs/10.1021/ct100379f>`_, `Machado et al. SIRAH Tools <https://academic.oup.com/bioinformatics/article/32/10/1568/1743152>`_.
We strongly advise you to read these articles before starting the tutorial.

.. important::

    Check :ref:`install <Download amber>` section for download and set up details before to start this tutorial.
    Since this is **tutorial 2**, remember to replace ``X.X``, the files corresponding to this tutorial can be found in: ``sirah_[version].amber/tutorial/2/``


2.1. Build CG representations
______________________________

Map the atomistic structure of a 20-mer DNA to its CG representation:

.. code-block:: bash

  ./sirah.amber/tools/CGCONV/cgconv.pl -i ./sirah.amber/tutorial/2/dna.pdb -o dna_cg.pdb

The input file ``-i`` dna.pdb contains all the heavy atoms composing the DNA molecule, while the  output ``-o`` dna_cg.pdb preserves a few of them.
Please check both PDB structures using VMD:

.. code-block:: bash

  vmd -m ./sirah.amber/tutorial/2/dna.pdb dna_cg.pdb

.. tip::

  This is the basic usage of the script **cgconv.pl**, you can learn other capabilities from its help:
  ``./sirah.amber/tools/CGCONV/cgconv.pl -h``

From now on it is just normal AMBER stuff!

2.2. Prepare LEaP
__________________

Use a text editor to create the file ``gensystem.leap`` including the following lines:

.. code-block:: console

    # Load SIRAH force field
    addPath ./sirah.amber
    source leaprc.sirah

    # Load model
    dna = loadpdb dna_cg.pdb

    # Info on system charge
    charge dna

    # Add solvent, counterions and 0.15M NaCl
    # Tuned solute-solvent closeness for best hydration
    solvateoct dna WT4BOX 20 0.7
    addionsrand dna NaW 88 ClW 50

    # Save Parms
    saveAmberParmNetcdf dna dna_cg.prmtop dna_cg.ncrst

    # EXIT
    quit

.. seealso::

    The available ionic species in SIRAH force field are: ``Na⁺`` (NaW), ``K⁺`` (KW) and ``Cl⁻`` (ClW). One
    ion pair (e.g. NaW-ClW) each 34 WT4 molecules renders a salt concentration of ~0.15M (see :ref:`Appendix <Appendix>` for details). 
    Counterions were added according to `Machado et al. <https://pubs.acs.org/doi/10.1021/acs.jctc.9b00953>`_.

2.3. Run LEaP
______________

Run the LEaP application to generate the molecular topology and initial coordinate files:

.. code-block:: bash

    tleap -f gensystem.leap

.. warning::

    Warning messages about long, triangular or square bonds in ``leap.log`` file are fine and
    expected due to the CG topology of some residues.

This should create a topology file ``dna_cg.prmtop`` and a coordinate file ``dna_cg.ncrst``.

Use VMD to check how the CG model looks like:

.. code-block:: bash

  vmd dna_cg.prmtop dna_cg.ncrst -e ./sirah.amber/tools/sirah_vmdtk.tcl


.. tip::

    VMD assigns default radius to unknown atom types, the script ``sirah_vmdtk.tcl`` sets the right
    ones. It also provides a kit of useful selection macros, coloring methods and backmapping utilities.
    Use the command ``sirah_help`` in the Tcl/Tk console of VMD to access the manual pages.

2.4. Run the simulation
________________________

Make a new folder for the run:

.. code-block:: bash

    mkdir -p run; cd run

In the course of long MD simulations the capping residues may eventually separate, this effect is
called helix fraying. To avoid such behavior create a symbolic link to the file ``dna_cg.RST``, which
contains the definition of Watson-Crick restraints for the capping base pairs of this CG DNA:

.. code-block:: bash

    ln -s ../sirah.amber/tutorial/2/SANDER/dna_cg.RST

.. note::

    The file dna_cg.RST can only be read by SANDER, PMEMD reads a different restrain format.

The folder ``sirah.amber/tutorial/2/SANDER/`` contains typical input files for energy minimization
(``em_WT4.in``), equilibration (``eq_WT4.in``) and production (``md_WT4.in``) runs. Please check carefully the
input flags therein, in particular the definition of flag *chngmask=0* at *&ewald* section is **mandatory**.

.. tip::

    **Some flags used in AMBER**

   - ``sander``: The AMBER program for molecular dynamics simulations.
   - ``-i``: Input file.
   - ``-o``: Output file.
   - ``-p``: Parameter/topology file.
   - ``-c``: Coordinate file.
   - ``-r``: Restart file.
   - ``-x``: Trajectory file.

**Energy Minimization:**

.. code-block:: bash

  $ sander -O -i ../sirah.amber/tutorial/2/SANDER/em_WT4.in -p ../dna_cg.prmtop -c ../dna_cg.ncrst -o dna_cg_em.out -r dna_cg_em.ncrst &

**Equilibration (NPT):**

.. code-block:: bash

  $ sander -O -i ../sirah.amber/tutorial/2/SANDER/eq_WT4.in -p ../dna_cg.prmtop -c dna_cg_em.ncrst -ref dna_cg_em.ncrst -o dna_cg_eq.out -r dna_cg_eq.ncrst -x dna_cg_eq.nc &

**Production (100ns):**

.. code-block:: bash

  sander -O -i ../sirah.amber/tutorial/2/SANDER/md_WT4.in -p ../dna_cg.prmtop -c dna_cg_eq.ncrst -o dna_cg_md.out -r dna_cg_md.ncrst -x dna_cg_md.nc &


.. important::

  You can find example input files for CPU and GPU versions of pmemd at folder PMEMD/ within sirah.amber/tutorial/2/

2.5. Visualizing the simulation
________________________________

That’s it! Now you can analyze the trajectory.
Process the output trajectory to account for the Periodic Boundary Conditions (PBC):

  .. code-block:: bash

      echo -e "autoimage\ngo\nquit\n" | cpptraj -p ../dna_cg.prmtop -y dna_cg_md.nc -x dna_cg_md_pbc.nc --interactive

**Load the processed trajectory in VMD:**

.. code-block::

    vmd ../dna_cg.prmtop ../dna_cg.ncrst dna_cg_md.nc -e ../sirah.amber/tools/sirah_vmdtk.tcl

.. note::

    The file ``sirah_vmdtk.tcl`` is a Tcl script that is part of SIRAH Tools and contains the macros to properly visualize the coarse-grained structures in VMD. Use the command ``sirah-help`` in the Tcl/Tk console of VMD to access the manual pages.
