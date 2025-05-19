# SiennaOps-Training

1. Start a Julia REPL, by either clicking on the Julia icon in Windows or by typing `julia` and pressing enter in a Mac or Windows Terminal:
```
$ julia
               _
   _       _ _(_)_     |  Documentation: https://docs.julialang.org
  (_)     | (_) (_)    |
   _ _   _| |_  __ _   |  Type "?" for help, "]?" for Pkg help.
  | | | | | | |/ _` |  |
  | | |_| | | | (_| |  |  Version 1.10.4 (2024-06-04)
 _/ |\__'_|_|_|\__'_|  |  Official https://julialang.org/ release
|__/                   |

julia> 
```

2. Install Sienna\Data and Sienna\Ops:
```
using Pkg; Pkg.add(["PowerSystems", "PowerNetworkMatrices", "PowerFlows", "PowerSystemCaseBuilder", "PowerSimulations", "PowerGraphics", "PowerAnalytics"])
```

3. Now load the packages we just installed:
```
using PowerSystems
using PowerSystemCaseBuilder
using PowerSimulations
using PowerGraphics
using PowerAnalytics
```

4. We also need an optimization problem solver -- we'll use HiGHS, which is open-source
```
using Pkg; Pkg.add(["HiGHS"])
using HiGHS
solver = optimizer_with_attributes(HiGHS.Optimizer, "mip_rel_gap" => 0.5)
```

5. Load in a version of the Reliability Test System with time-series for day-ahead modeling:
```
system = build_system(PSISystems, "modified_RTS_GMLC_DA_sys");
```

6. Print the system to take a look around:
```
system
```

7. Use the help function `?` to look up information about the `RenewableDispatch` `Type`

8. Take a look at what renewable generators are in this power system:
```
show_components(system, RenewableDispatch)
```

9. Let's look at more information about the first renewable generator:
```
get_component(RenewableDispatch, system, "122_WIND_1")
```

10. Let's start by looking at the system without it's large-scale renewables by turning them
off:
```
for g in get_components(RenewableDispatch, system)
           set_available!(g, false)
end
```

11. Set up a default unit commitment model
```
template = template_unit_commitment()
```

12. Apply that template to our data to make a complete model, with both data and equations:
```
models = SimulationModels(
    decision_models=[
        DecisionModel(template, system, optimizer=solver, name="UC"),
    ],
)
```
We just have one model -- the unit commitment model.


13. In addition to defining the formulation template, sequential simulations require
definitions for how information flows between problems to set the initial conditions
of the next problem:
```
DA_sequence = SimulationSequence(
    models=models,
    ini_cond_chronology=InterProblemChronology(),
)
```

14. Finally, put together the models and the flow of information between them to simulate
3 days of a unit commitment:
```
sim = Simulation(
    name = "default_UC",
    steps = 3,
    models=models,
    sequence=DA_sequence,
    simulation_folder=mktempdir(cleanup=true),
)
```

15. Build the simulation, which does all the equation preparation before executing:
```
build!(sim, recorders = [:simulation])
```

16. Simulate!
```
execute!(sim)
```

17. Now, load in the simulation results from the output files:
```
results = SimulationResults(sim)
uc_results = get_decision_problem_results(results, "UC")
```

18. Let's visualize the results
```
plot_fuel(uc_results);
```

19. We can also look at how much it cost to operate the thermal generators at each time step:
```
costs = read_realized_expressions(uc_results, list_expression_names(uc_results))["ProductionCostExpression__ThermalStandard"]
```
Wind, solar, and hydro have 0 operating cost and do not contribute to total cost in this model

20. We can sum over the set of generators and time-steps to get total production cost for this window
```
sum(sum(eachcol(costs[!, 2:end])))
```

## Now, let's change some things in the system:

21. Now, let's change how the thermal generators behave by adding in ramping and minimum up
and down time constraints:
```
set_device_model!(template, ThermalStandard, ThermalStandardUnitCommitment)
```
You can see the formulations options in the documentation: 
https://nrel-sienna.github.io/PowerSimulations.jl/stable/formulation_library/ThermalGen/

22. We've changed our template, so we need to re-build the DecisionModels within our simulation, which
includes a copy of the template:
```
models = SimulationModels(
    decision_models=[
        DecisionModel(template, system, optimizer=solver, name="UC"),
    ],
)
DA_sequence = SimulationSequence(
    models=models,
    ini_cond_chronology=InterProblemChronology(),
)
sim = Simulation(
    name = "default_UC",
    steps = 3,
    models=models,
    sequence=DA_sequence,
    simulation_folder=mktempdir(cleanup=true),
)
```

22. Now, let's re-build, and execute our simulation:
```
build!(sim, recorders = [:simulation]);
execute!(sim)
```

23. Let's re-extract the results and take a look at the impact on the dispatch stack:
```
results = SimulationResults(sim)
uc_results = get_decision_problem_results(results, "UC")
plot_fuel(uc_results);
```

24. Finally, let's extract the operating costs again and see the impact on total cost:
```
costs = read_realized_expressions(uc_results, list_expression_names(uc_results))["ProductionCostExpression__ThermalStandard"]
sum(sum(eachcol(costs[!, 2:end])))
```

## Adding in renewables

25. Now, let's add in those large-scale wind and solar plants by making them
available again:
```
for g in get_components(RenewableDispatch, system)
    set_available!(g, true)
end
```

26. We've changed the data, not the template formulations, so this time we just need to re-build and simulate:
```
build!(sim, recorders = [:simulation]);
execute!(sim)
```

27. Let's re-extract the results and take a look at the impact on the dispatch stack:
```
results = SimulationResults(sim)
uc_results = get_decision_problem_results(results, "UC")
plot_fuel(uc_results);
```

28. Finally, let's extract the operating costs again and see the impact on total cost:
```
costs = read_realized_expressions(uc_results, list_expression_names(uc_results))["ProductionCostExpression__ThermalStandard"]
sum(sum(eachcol(costs[!, 2:end])))
```
