---
title: Proposals
sectionName: proposals
position: 4
---

### File specification

One file covers the extent of one simulation, e.g. changes in temperature (time-indepedent context) 
can only span multiple simulations and thus multiple files. This means one might end up with multiple files for one realisation.
To get back a time ordering, the simulated step range should be stated in the file. 
Additionally it has to be set when continuing a realisation, i.e. starting a new simulation.

```bash
file.h5
  readdy/
    time_range # might be an observable
    obs/
      labelx-msd/ # user defined labels for observables
      labely-rdf/
      labelz-traj/
    config/ # time independent information
      particle_types
      reactions
      potentials
    snapshots/
      0/
        positions
        topologies
        time_step_number
      1/
        positions
        topologies
        time_step_number
      ...
```


### Top level usage
```python
# define system here together with the units that readdy will use
system = ReactionDiffusionSystem(length_unit=units.swedish_miles)
system.kbt = 50. * units.kJ / units.mol # units will be converted when scheme is configured, run()
system.add_species("A", diffusion_const=4. * units.nm * units.nm / units.ns)
system.add_species("B", diffusion_const=5.) # default units will be used
system.add_reaction("A+{13.}B->{42.}C", radius_unit=units.nm, rate_unit=1./units.ns)
system.add_reaction("B->{3.}C")

# create simulation (define algorithms to run the system)
simulation = system.simulation(kernel="SingleCPU")  # use defaults. 
simulation.observe_rdf(callback=lambda x: print(x), stride=5, bins=np.arange(0.,10.,1.)*units.nm, write_to_file=Ja,bitte)  # actually part of the system, but configured through the simulation object
simulation.output_file = "path/to/file"
simulation.integrator = "EulerBDIntegrator"
simulation.compute_forces = False
simulation.reaction_scheduler = "UncontrolledApproximation"
simulation.evaluate_observables = False
simulation.record_snapshots(stride=50, last_frame=True, output_dir="", overwrite=True)
simulation.run(10)  # does the configuration and runs the simulation, i.e. system and simulation are finalized here
# or
simulation.run_while(util.criterion.n_minutes(10))

# continue simulation
simulation.run(10)

# second call RAISES exception, because you can only simulate a system once.
simulation = system.simulation()  # use defaults. 

# OR
simulation = system.simulation(integrator = "EulerBDIntegrator",
                               compute_forces = False,
...
)
simulation.run(10)
```

### General suggestions

- [ ] update API as below (and create top level python api):

```python
# define system here
system = ReactionDiffusionSystem(kernel="SingleCPU")

# create simulation (define algorithms to run the system)
simulation = system.simulation()  # use defaults. 
simulation.add_observable('rdf')  # actually part of the system, but configured through the simulation object
simulation.integrator = "EulerBDIntegrator"
simulation.compute_forces = False
simulation.reaction_scheduler = "UncontrolledApproximation"
simulation.evaluate_observables = False
simulation.run(10)  # does the configuration and runs the simulation

# continue simulation
simulation.run(10)

# second call RAISES exception, because you can only simulate a system once.
simulation = system.simulation()  # use defaults. 

# OR
simulation = system.simulation(integrator = "EulerBDIntegrator",
                               compute_forces = False,
...
)
simulation.run(10)
```

- [ ] suggestion: Allow geometry files as input for a box potential such that more complicated shapes can be realized with external tools
- [ ] implement CUDA kernel
    - meet up with Felix to discuss HALMD integration
- [ ] implement reactions with topologies
    - come up with convenient API to create / manipulate topologies
- [ ] improve reaction scheduler to gain more performance
    - filter particles out, that do not participate in any reaction
    - upon event creation, check if event is scheduled to happen in the current time interval
    - this introduces a bias on the probabilities of the remaining events (if there are intersections), try to balance that
- [ ] improve neighbor lists to gain more performance
    - verlet lists
- [ ] snapshotting
    - this point belongs together with the IO point
    - implement snapshotting using the observables framework
- [ ] implement IO (de-/serialization, dumping of trajectories into hdf5 files)
    - implement VMD plugin (see, e.g., lammps plugin on how to hide particles)
    - use and extend h5md?
    - use h5xx?
    - implement IO using the observables framework
- [ ] create benchmark (NCores x NParticles x rates)
    - maybe execute this benchmark automatically on some host
- [ ] domain decomposition (e.g., MPI)

### Topology reaction scheduling on GPUs

Let's assume we can do the following on the GPU
- Diffusion of normal particles and topologies
- Simple reactions, i.e. reactions between normal particles

Topology reactions can change the structure of topologies (e.g. polymerization, 
binding/unbinding to a complex). This cannot be done on the GPU. Instead those reactions
have to be performed on the CPU, which is in principle not a problem when those reactions
occur rarely. The actual problem is, that the __GPU cannot halt on its own__ when it find out that
a topology reaction should be performed. There are two ways of determining how long the GPU
should execute:
1. with a fixed time $\tau$
    - the GPU executes diffusion and normal reactions for a time $\tau$ which is much larger
    than the integration step and then returns
    - the CPU performs all possible topology reaction events based on its current state, 
    where reaction probabilities are $\mathrm{rate}\cdot \tau$. This could be done with the fixed timestep
    version of our reaction schedulers
2. with a time $\tau$ sampled from a Gillespie algorithm
    - given one system state with a number of possible topology reactions events, 
    choose __one__ event and a corresponding $\tau$
    - perform this reaction and then let the GPU run for $\tau$

### Compartments

Compartments are defined as regions of the simulation box. Within those compartments certain instantaneous
conversions are defined. Those are different from actual reactions implementation wise but they
basically do the same thing. For example:
- A compartment is defined via $r > 10$, i.e. if a particle is more than 10 length units away from the origin
it is considered to be in the compartment.
- Associated with this compartment is a conversion `A -> B`, i.e. if an A particle travels into this compartment
it will be converted to a B particle instantaneously.
- One could define a second complementary compartment $r < 10$ with a conversion `B -> A`.
 
Why is this useful? For example:
- If one is only interested in setting up an observable for particles close to a certain point. E.g. I want to
know the pair correlation radial distribution of `A` particles around some other static particle, but I only need the 
radial distribution in close proximity (because the static particle might induce some crowding effects), 
I can set up a compartment that converts the `A` particles to `A_close` particles when they come close to the 
static particle. Then my observable only records the pair correlation of `A_close` particles.
- Absorbing boundary conditions can easily be implemented. Imagine I have a non-periodic system in x-direction and
I want to construct an absorbing boundary in the halfspace defined by $x < 0$ for a certain particle type `A`. One could
define the compartment $x < 0$ with the conversion `A -> N`, where `N` is a species that rapidly decays 
in the next timestep (i.e. `N` has an _actual_ decay reaction with a very high rate). 
Note that there is no second complementary compartment with a reverse conversion, i.e. `N` particles cannot become
`A` particles again. 

The execution of those conversions can be put into a Program, which is executed on the kernels. To
have full flexibility and accessibility from the Python layer, it makes sense to construct those compartments
similar to potentials.
Another nice feature is, that the computational complexity of applying the conversions is $O(N)$ when $N$ is the
total number of particles. So it should not take longer than the evaluation of first order potentials.

What types of compartments are easily implemented?:

- Radial, $ \| r-r_0 \| > \text{or} < R $, with three parameters
    - Vec3 origin $r_0$
    - double radius $R$
    - bool largerOrLess
- Plane, $ a_0 x_0 + a_1 x_1 + a_2 x_2 > \text{or} < d $, in Hesse normal form with three parameters
    - Vec3 normalCoefficients $a$
    - double distanceFromOrigin $d$
    - bool largerOrLess

Issues can arise if compartments overlap. Then it is implementation-dependent, which conversions get executed first
and thus which particle-type you end up with. There would be no _efficient_ way of determining if compartments overlap.
But when a particle is found to be in two compartments during run-time, warnings can be printed.