Symbion: Interleaving symbolic and concrete execution
=================================

Let's suppose you want to symbolically analyze a specific function of a program, but there is an huge initialization step that you want to skip because it is not necessary to your analysis. Moreover, maybe your program is running on an embedded system and you have access to a debug interface, but you can't really extract it from the device.

This is the perfect scenario for `Symbion`!

We implemented a built-in system that let users define a so called `ConcreteTarget` that is used to "import" a concrete state of the target program inside `angr`.
Once the state is imported you can symbolically execute the code, implement your analyses, modify the concrete process memory and eventually resume the concrete process execution. By iterating this process it is possible to implement *run-time and interactive* advanced symbolic analyses that are backed up by the real program execution! 

Isn't that cool?


## How to install
To immediately start to use this technique you need an implementation of a `ConcreteTarget` (effectively, an object that is going to be the "glue" between `angr` and the concrete process.)
We ship a default one (the `AvatarGDBConcreteTarget`) in the following repo https://github.com/angr/angr-targets.

Assuming you installed [angr-dev](https://github.com/angr/angr-dev), activate the virtualenv and run:

```bash
git clone https://github.com/angr/angr-targets.git
cd angr-targets
pip install .
```

Now you are ready to go!

##Gists
Once you have create an entry state, instantiated a *SimulationManager*, and specified a list of *stop_points* using the `Symbion` interface:

```python
# Instantiating the ConcreteTarget
avatar_gdb = AvatarGDBConcreteTarget(avatar2.archs.x86.X86_64,
                                     GDB_SERVER_IP, GDB_SERVER_PORT)

# Creating the Project
p = angr.Project(binary_x64, concrete_target=avatar_gdb,
                             use_sim_procedures=True)

# Getting an entry_state
entry_state = p.factory.entry_state()

# Forget about these options as for now, will explain later.
entry_state.options.add(angr.options.SYMBION_SYNC_CLE)
entry_state.options.add(angr.options.SYMBION_KEEP_STUBS_ON_SYNC)      

# Use Symbion!                                
simgr.use_technique(angr.exploration_techniques.Symbion(find=[0x85b853])
```
 we are going to resume the concrete process execution. 
 
 When one of your *stop_points* (effectively a breakpoint) is hit, we give the control back to angr. A new plugin called *concrete* is in charge of synchronizing the concrete state of the program inside a new `SimState`. Shortly this is what's happening during the synchronization:

* All the registers' values (NOT marked with concrete=False in the respective arch file in [archinfo](https://github.com/angr/archinfo/tree/master/archinfo)) are copied inside the new SimState.
* The underlying memory backend is hooked in a way that all the further memory accesses triggered during symbolic execution are redirected to the concrete process.
* If the project is initialized with SimProc like this:

```python
p = angr.Project(binary_x64, concrete_target=avatar_gdb,
                             use_sim_procedures=True)
```
we are going to re-hook the external functions' addresses with a `SimProcedure` if we happen to have it, otherwise with a `SimProcedure` stub (you can control this decision by using the Options SYMBION_KEEP_STUBS_ON_SYNC). Conversely, the real code of the function is executed inside angr (Warning: do that at your own risk!)

Once this process is completed, you can play with your new `SimState` backed by the concrete process freezed at that particular *stop_point*.

## Options
The way we synchronize the concrete process inside angr is customizable by two state options:

* **SYMBION_SYNC_CLE**: this option controls the synchronization of the memory mapping of the program inside angr. When the project is created, the memory mapping inside angr is different from the one inside the concrete process (this will change as soon as `Symbion` will be fully compatible with [archr](https://github.com/angr/archr)). If you want the process mapping to be fully synchronized with the one of the concrete process, set this option to the `SimState` before initializing the `SimulationManager` (Note that this is going to happen at the first synchronization of the concrete process inside `angr`, *NOT* before)

```python
entry_state.options.add(angr.options.SYMBION_SYNC_CLE)
simgr = project.factory.simgr(state)
```

* **SYMBION_KEEP_STUBS_ON_SYNC**: this option controls how we re-hook external functions with SimProcedures. If the project has been initialized to use `SimProcedures` (use_sim_procedures=True), we are going to re-hook external functions with `SimProcedures` (if we have that particular implementation) or with a generic stub. If you want to execute `SimProcedures` just for functions for which we have an available implementation and the real code for the ones we have not, set this option to the `SimState` before initializing the `SimulationManager`.

```python
entry_state.options.add(angr.options.SYMBION_KEEP_STUBS_ON_SYNC)
simgr = project.factory.simgr(state)
```


## Example
You can find more information about this technique and a complete example in our blogpost: https://angr.io/blog/angr_symbion/.

For more technical detail a public paper will be available soon, or if you really need it urgently ping @degrigis on our `angr` Slack channel.