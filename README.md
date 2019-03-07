# FabSC
This plugin runs the samples from an ![EasyVVUQ](https://github.com/UCL-CCS/EasyVVUQ) Stochastic Collocation (SC) campaign using ![FabSim3](https://github.com/djgroen/FabSim3) via the `campaign2ensemble` subroutine.

## Installation
Simply type `fab localhost install_plugin:FabSC` anywhere inside your FabSim3 install directory.

## Explanation of files
+ `FabSC/FabSC.py`: contains the *sc* subroutine in which the job properties are specified, e.g. number of cores, memory, wall-time limit etc
+ `FabSC/templates/sc`: contains the command-line execution command for a single EasyVVUQ SC sample.
+ `FabSC/examples/advection_diffusion/`: an example script, see below.

## Detailed Examples

### Executing an ensemble job on localhost
In the examples folder there is a script which runs an EasyVVUQ SC campaign using FabSim3 for a simple advection-diffusion equation (ade) finite-element solver on the localhost. The governing equations are:

![equation](https://latex.codecogs.com/gif.latex?%5Cfrac%7Bdu%7D%7Bdx%7D%20&plus;%20%5Cfrac%7B1%7D%7BPe%7D%5Cfrac%7Bd%5E2u%7D%7Bdx%7D%20%3D%20f),

where the Peclet Number (Pe) and forcing term (f) are the uncertain SC parameters, and u is the velocity subject to Dirichlet boundary conditions u(0)=u(1)=0. The script executes the ensemble using Fabsim, computes the first two moments of the output, generates some random sample of the SC surrogate and computes the first-order Sobol indices of Pe and f.

The file `run_SC_Fab_campaign.py` contains the main script. The first 4 steps are the same as for an EasyVVUQ campaign that does not use FabSim to execute the runs:
 1. Create an EasyVVUQ campaign object, with `ade_input.json` as argument, which defines the UQ campaign:
 `my_campaign = uq.Campaign(state_filename=input_json)`
 2. Per uncertain parameter, select the input distribution and polynomial order, e.g. `my_campaign.vary_param("Pe", dist=uq.distributions.legendre(6))`
 3. Select the SC_Sampler which creates a tensor grid from the 1D rules selected in step 2: `sc_sampler = uq.elements.sampling.SCSampler(my_campaign)`, and add the runs via `my_campaign.add_runs(sc_sampler, max_num=number_of_samples)`. The `number_of_samples` variable is simply the number of points in the tensor grid.
 4. Create the ensemble run directories which will be used in Fabsim's `campaign2ensemble` subroutine: `my_campaign.populate_runs_dir()`
 
The fifth step is specific to FabSim. For now, several variables need to be hardcoded, i.e.: 
 + A simulation identifier (`$sim_ID`)
 + Your FabSim home directory (`$fab_home`)
 + The `FabSC/template/sc` file contains the command line instruction to run a single sample, in this case: `python3 $ade_exec ade_in.json`. Here, `ade_in.json` is just the input file with the parameter values generated by EasyVVUQ. Furthermore, `$ade_exec` is the full path to the Python script which runs the advection diffusion equation at the parameters of `ade_in.json`. It is defined in `deploy/machines_user.yml`, which in this case looks like
 
`localhost:`

 &nbsp;&nbsp;&nbsp;&nbsp;`ade_exec: "$fab_home/plugins/FabSC/examples/advection_diffusion/run_ADE.py"`
 
 The following two commands execute the ensemble run:
 
 1. `cd $fab_home && fab localhost campaign2ensemble:$sim_ID, campaign_dir=$campaign_dir`
 2. `cd $fab_home && fab localhost sc_ensemble:$sim_ID`
 
The run directory `$campaign_dir` is available from the EasyVVUQ object. The FabSim results directory (`~/FanSim3/results`) has the same structure as the EasyVVUQ run directory, so the results can simply be copied back, in this case via

`cp -r ~/FabSim3/results/ade_example1_localhost_16/RUNS/Run_* $campaign_dir/runs`

Afterwards, post-processing tasks in EasyVVUQ can be undertaken, by creating a `SCAnalysis` object: `sc_analysis = uq.elements.analysis.SCAnalysis(my_campaign, value_cols=output_columns)`. Here, `output_columns` is the name of the column in the output CSV file containing the simulation results (u(x) in this case).

+ To compute the mean and variance of the output use: `sc_analysis.get_moments()`.  

+ To generate a random output sample from the SC surrogate, use `sc_analysis.surrogate(xi)`, where `xi` is a random Monte Carlo input sample.

+ To compute the Sobol indices from the SC samples, use: `sc_analysis.get_Sobol_indices(typ)`, where `typ` can be `'first_order'` or `'all'`.

### Executing an ensemble job on a remote host

To run the example script on a remote host, every instance of `localhost` must replaced by the `machine_name` of the remote host. Ensure the host is defined in `machines.yml`, and the user login information and `$ade_exec` in `deploy/machines_user.yml`.

