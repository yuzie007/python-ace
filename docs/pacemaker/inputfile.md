# Input file

The required settings are provided by input YAML file. This file consists of several sections devoted to setting up 
particular settings of `pacemaker`. The sections are listed below.

## Cutoff and (optional) metadata

* Global cutoff for the neighborlist constructor is setup as:

```YAML
cutoff: 10.0
```

* Metadata (optional)

This is arbitrary key (string)-value (string) pairs that would be added to the potential YAML file: 
```YAML
metadata:
  info: some info
  comment: some comment
  purpose: some purpose
```
Moreover, `starttime` and `user` fields would be added automatically

## Dataset specification section

This section is denoted by the key

```yaml
data:
  ...
```
Dataset could be saved into file as a pickled `pandas` dataframe with special names for [columns](#fitting-dataset-preparation).

```YAML
data: 
  filename: some_stored_dataset.pckl.gzip
  # cache_ref_df: False             # whether to store the queried or modified dataset into file, default - True
  # ignore_weights: False          # whether to ignore energy and force weighting columns in dataframe
  # datapath: ../data              # path to folder with cache files with pickled dataframes 
```
`data:datapath` option, if not provided, could be replaced with *environment variable* **$PACEMAKERDATAPATH**

Example of creating the **subselection of fitting dataframe** and saving it is given in `examples/data_selection/data_selection.ipynb`

Example of generating **custom energy/forces weights** is given in `examples/custom-weights/data_custom_weights.ipynb`


### Test set

You could provide test set either as a fraction or certain number of samples from the train set (option `test_size`) or
as a separate pckl.gzip file (option `test_filename`)

```yaml
data:
  test_filename: my_test_dataset.pckl.gzip
```

or

```yaml
data:
  test_size: 100 # would take 100 samples randomly from train/fit set
  # test_size: 0.1 #  if <1 - would take given fraction of samples randomly from train/fit set
```

## Interatomic potential (or B-basis) configuration


### Basis configuration
In order to specify the B-basis potential, you have to provide four main components (aka **basis shape**): `elements`, `embeddings` for each element,
`bonds` for each possible pairs of elements and `functions` for each possible combination of elements (unary, binary, ternary, etc.) 
as follows:
```YAML
potential:
  deltaSplineBins: 0.001
  elements: [Al, Ni]  # list of all element

  # Embeddings are specified for each individual elements,
  # all parameters could be distinct for different species
  embeddings: ## possible keywords: ALL, UNARY, elements: Al, Ni
    Al: {
      npot: 'FinnisSinclairShiftedScaled',
      fs_parameters: [1, 1, 1, 0.5], ## non-linear embedding function: 1*rho_1^1 + 1*rho_2^0.5
      ndensity: 2,
      
      # core repulsion parameters
      rho_core_cut: 200000,
      drho_core_cut: 250
    }

    Ni: {
      npot: 'FinnisSinclairShiftedScaled', ## linear embedding function: 1*rho_1^1
      fs_parameters: [1, 1],
      ndensity: 1,

      # core repulsion parameters
      rho_core_cut: 3000,
      drho_core_cut: 150
    }

  ## Bonds are specified for each possible pairs of elements
  ## One could use keywords: ALL (Al,Ni, AlNi, NiAl)
  bonds: ## possible keywords: ALL, UNARY, BINARY, elements pairs as AlAl, AlNi, NiAl, etc...  
    ALL: {
        radbase: ChebExpCos,
        radparameters: [5.25],
        
        ## outer cutoff
        rcut: 5,
        dcut: 0.01,

        ## inner cutoff  [r_in - delta_in, r_in] - transition region from ACE to core-repulsion
        # at r < r_in-delta_in - no ACE interaction, only core-repulsion
        r_in: 1.0,
        delta_in: 0.5,
        
        
        ## core-repulsion parameters `prefactor` and `lambda` in
        ## prefactor*exp(-lambda*r^2)/r, > 0 only if r < r_in - delta_in
        core-repulsion: [100.0, 5.0],
    }

    ## BINARY overwrites ALL settings when they are repeated
    BINARY: {
        radbase: ChebPow,
        radparameters: [6.25],

        ## cutoff may vary for different bonds
        rcut: 5.5,
        dcut: 0.01,

        ## inner cutoff, applied in a range [r_in - delta_in, r_in]
        r_in: 1.0,
        delta_in: 0.5,

        ## core-repulsion parameters `prefactor` and `lambda` in
        ## prefactor*exp(-lambda*r^2)/r,> 0 only if r < r_in - delta_in
        core-repulsion: [10.0, 5.0],
    }

  ## possible keywords: ALL, UNARY, BINARY, TERNARY, QUATERNARY, QUINARY,
  ##  element combinations as (Al,Al), (Al, Ni), (Al, Ni, Zn), etc...
  functions:
    # number_of_functions_per_element: 700  # specify the total number of functions per element to keep
    UNARY: {
      nradmax_by_orders: [15, 3, 2, 2, 1],
      lmax_by_orders: [ 0, 2, 2, 1, 1],
      # coefs_init: zero # initialization of functions coefficients: zero (default) or random
    }

    BINARY: {
      nradmax_by_orders: [15, 2, 2, 2],
      lmax_by_orders: [ 0, 2, 2, 1],
      # coefs_init: zero # initialization of functions coefficients: zero (default) or random
    }
```

In sections `embeddings`,  `bonds` and `functions` one could use keywords (ALL, UNARY, BINARY, TERNARY, QUATERNARY, QUINARY).
The settings provided by more specific keyword will override those from less specific keyword, 
i.e. ALL < UNARY < BINARY < ('Al','Ni') 

### Upfitting
If you want to continue the fitting of the existing potential from `potential.yaml` file, then specify
```YAML
potential: potential yaml
```
alternatively, one could use `pacemaker ... -p potential.yaml ` option.

For specifying both initial and target potential from the file one could provide:
```YAML
potential: 
  filename: potential.yaml
  
  ## in "ladder" fitting scheme, potential from with to start fit
  # initial_potential: initial_potential.yaml

  ## reset potential from potential.yaml, i.e. set radial coefficients to delta_nk and func coeffs=[0..]
  # reset: true 
```
or alternatively, one could use  `pacemaker ... -p potential.yaml -ip initial_potential.yaml ` options.

## Fitting settings
Example of `fit` section is:
```YAML
fit:
    ## LOSS FUNCTION OPTIONS ##
    loss: {
      ## [0..1] or auto, relative force weight,
      ## kappa = 0 - energies-only fit,
      ## kappa = 1 - forces-only fit
      ## auto - determined from dataset based on variance of energies and forces
      kappa: 0,
      ## L1-regularization coefficient
      L1_coeffs: 0,
      ## L2-regularization coefficient
      L2_coeffs: 0,
      ## w0 radial smoothness regularization coefficient
      w0_rad: 0,
      ## w1 radial smoothness regularization coefficient
      w1_rad: 0,
      ## w2 radial smoothness regularization coefficient
      w2_rad: 0  
    }

    ## DATA WEIGHTING OPTIONS ##
    weighting: {
        ## weights for the structures energies/forces are associated according to the distance to E_min:
        ## convex hull ( energy: convex_hull) or minimal energy per atom (energy: cohesive)
        type: EnergyBasedWeightingPolicy,
        ## number of structures to randomly select from the initial dataset
        nfit: 10000,         
        ## only the structures with energy up to E_min + DEup will be selected
        DEup: 10.0,  ## eV, upper energy range (E_min + DElow, E_min + DEup)        
        ## only the structures with maximal force on atom  up to DFup will be selected
        DFup: 50.0, ## eV/A
        ## lower energy range (E_min, E_min + DElow)
        DElow: 1.0,  ## eV
        ## delta_E  shift for weights, see paper
        DE: 1.0,
        ## delta_F  shift for weights, see paper
        DF: 1.0,
        ## 0<wlow<1 or None: if provided, the renormalization weights of the structures on lower energy range (see DElow)
        wlow: 0.75,        
        ##  "convex_hull" or "cohesive" : method to compute the E_min
        energy: convex_hull,        
        ## structures types: all (default), bulk or cluster
        reftype: all,        
        ## random number seed
        seed: 42 
    }
    
    ## Custom weights:  corresponding to main dataset index and `w_energy` and `w_forces` columns should
    ## be provided in pckl.gzip file
    #weighting: {type: ExternalWeightingPolicy, filename: custom_weights_only.pckl.gzip}
    
    ## OPTIMIZATION OPTIONS ##
    optimizer: BFGS # BFGS, L-BFGS-B, Nelder-Mead, etc. : scipy minimization algorithm
    ## additional options for scipy.minimize(..., options={...}, ...)
    #options: {maxcor: 100}    
    maxiter: 1000 # maximum number of iteration for EACH scipy minimization round
    
    ## EXTRA OPTIONS ##
    repulsion: auto            # set inner cutoff based on the minimal distance in the dataset
      
    #trainable_parameters: ALL  # ALL, UNARY, BINARY, ..., radial, func, {"AlNi": "func"}, {"AlNi": {"func","radial"}}, ...

    ##(optional) number of consequentive runs of fitting algorithm (for each ladder step), that helps convergence
    #fit_cycles: 1   
    
    ## starting from second fit_cycle:
    
    ## applies Gaussian noise with specified relative sigma/mean ratio to all potential trainable coefficients
    #noise_relative_sigma: 1e-3
    
    ## applies Gaussian noise with specified absolute sigma to all potential trainable coefficients
    #noise_absolute_sigma: 1e-3  
    
    # reset the function coefficients according to Gaussian distribution with given sigma; enable ensemble fitting mode
    #randomize_func_coeffs: 1e-3  

    ## LADDER SCHEME (i.e. hierarchical fitting) ##      
    ## enables hierarchical fitting (LADDER SCHEME), that sequentially add specified number of B-functions (LADDER STEP)    
    #ladder_step: [10, 0.02]  
    ##      - integer >= 1 - number of basis functions to add in ladder scheme,
    ##      - float between 0 and 1 - relative ladder step size wrt. current basis step
    ##      - list of both above values - select maximum between two possibilities on each iteration 
    ##     see. Ladder scheme fitting for more info


    ## Possible values:
    ## body_order  -  new basis functions are added according to the body-order, i.e., a function with higher body-order
    ##                will not be added until the list of functions of the previous body-order is exhausted
    ## power_order -  the order of adding new basis functions is defined by the "power rank" p of a function.
    ##                p = len(ns) + sum(ns) + sum(ls). Functions with the smallest p are added first      
    #ladder_type: body_order    
                                

    ## callbacks during the fitting. Module quick_validation.py should be available for import
    ## see example/pacemaker_with_callback for more details and examples
    #callbacks:
    #  - quick_validation.test_fcc_potential_callback
```

If not specified, then *uniform weight* and *energy-only* fit (kappa=0),
*fit_cycles*=1, *noise_relative_sigma* = 0 settings will be used.

If ladder fitting scheme is used, then intermediate version of the potential after each ladder step will be saved
into `interim_potential_ladder_step_{LADDER_STEP}.yaml`.

## Backend specification

```YAML
backend:
  evaluator: tensorpot  

  ## for `tensorpot` evaluator, following options are available:
  # batch_size: 10            # batch size for loss function evaluation, default is 10
  # batch_size_reduction: True # automatic batch_size reduction if not enough memory (default - True) 
  # batch_size_reduction_factor: 1.618  # batch size reduction factor
  # display_step: 20          # frequency of detailed metric calculation and printing
```
Alternatively, backend could be selected as `pacemaker ... -b tensorpot` 

## Ladder (hiererchical) basis extension
In a ladder scheme potential extension happens by adding new portion of basis functions step-by-step,
to form a "ladder" from *initial potential* to *final potential*. Following settings should be added to
the input YAML file:

* Specify *final potential* shape by providing `potential` section:
```yaml
potential:
  deltaSplineBins: 0.001
  element: Al
  ...
```

* Specify *initial potential* by providing `initial_potential` option in `potential` section: 
```yaml
potential:
    ...
    initial_potential: some_start_or_interim_potential.yaml    # potential to start fit from
```

If *initial potential* is not specified, then the fit will start from empty potential. 
Alternatively, you can specify *initial potential* by command-line option

`pacemaker ... -ip some_start_or_interim_potential.yaml `

* Specify `ladder_step` in `fit` section:
```yaml
fit:
    ...
    
    ladder_step: [10, 0.02]       
## Possible values:
##  - integer >= 1 - number of basis functions to add in ladder scheme,
##  - float between 0 and 1 - relative ladder step size wrt. current basis step
##  - list of both above values - select maximum between two possibilities on each iteration  
```
