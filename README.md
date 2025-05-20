# SiennaOps-Training

1. Navigate to a new folder and start a Julia REPL, by either clicking on the Julia icon in Windows or by typing `julia` and pressing enter in a Mac or Windows Terminal:
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

2. Setup an environment and output folder:
```
using Pkg 
Pkg.activate(".") 
Pkg.instantiate()

results_dir = "outputs"
mkdir(results_dir)
```

3. Install Sienna\Data, Sienna\Ops, and an optimization problem solver -- we'll use HiGHS, which is open-source:
```
Pkg.add([
    "PowerSystems",
    "PowerNetworkMatrices",
    "PowerSystemCaseBuilder",
    "PowerSimulations",
    "StorageSystemsSimulations",
    "HydroPowerSimulations",
    "PowerFlows",
    "PowerGraphics",
    "PowerAnalytics",
    "HiGHS",
    "DataFrames",
    "CSV",
]);
```

4. Now load the packages we'll need today:
```
using PowerSystems
using PowerSystemCaseBuilder
using PowerSimulations
using PowerGraphics
using PowerAnalytics
using DataFrames
using Dates
using CSV
using HiGHS
```

## Load systems from PowerSystemCaseBuilder.jl

5. We can start by listing all the available test system categories:
```
show_categories()
```

6. What PowerSimulations.jl test systems are available?
```
show_systems(PSISystems)
```

7. Load in a version of the Reliability Test System with time-series for day-ahead modeling:
```
system = build_system(PSISystems, "modified_RTS_GMLC_DA_sys");
```

8. Print the system to take a look around:
```
system
```

What are all these Types? Take a look at the Type Tree:
https://nrel-sienna.github.io/PowerSystems.jl/stable/api/type_tree/

## Let's explore and retrieve data

9. Take a look at what renewable generators are in this power system:
```
show_components(system, RenewableDispatch)
```

10. Let's look at more information about a particular renewable generator, retrieving by name:
```
g = get_component(RenewableDispatch, system, "101_PV_4")
```

## Accessing data in a component
Each field has a getter and setter function of the form:
get_
set_

11. Get the rating:
```
get_rating(g)
```

12. 0.516.... what?
```
get_units_base(system)
```

Getters and setters handle per-unitization for you, but different per-unitization options trip up new users!
When in doubt, switch your system units base and spotcheck your data with different bases to make sure you're confident. See:
https://nrel-sienna.github.io/PowerSystems.jl/stable/tutorials/creating_system/#Changing-System-Per-Unit-Settings

13. Change the settings to device base (per-unitized by `base_power` value):
```
set_units_base_system!(system, "DEVICE_BASE")
```

14. Now get the rating again:
```
get_rating(g)
```

15. Change the settings to natural units (MW, MVAR, etc.)
```
set_units_base_system!(system, "NATURAL_UNITS")
```

16. Now get the rating again:
```
get_rating(g)
```

17. Now let's set a new rating -- still in natural units! (MVAR)
```
set_rating!(g, 25.8)
```

18. And we can always just print the component again to review its data:
```
g
```

19. Some fields link to other objects in the `System`:
```
get_bus(g)
```

## Manipulating components

20. Get many components by `Type`:
```
res = get_components(RenewableDispatch, system)
```

21. What exactly did we just retrieve?
```
typeof(res)
```

22. Iterators get us the data access we need, without really large memory allocations
Use an iterator to easy update data:
```
for g in get_components(RenewableDispatch, system)
    set_available!(g, false)
end
```

## Filtering to get particular components

23. Be mindful of those units when filtering!
```
get_name.(get_components(x -> get_prime_mover_type(x) == PrimeMovers.PVe && get_rating(x) >= 100.0, RenewableDispatch, system))
```

24. Just update combustion turbine ramp limits:
```
for comp in get_components(x -> get_prime_mover_type(x) == PrimeMovers.CT, ThermalStandard, system)
    pmax = get_active_power_limits(comp).max
    set_ramp_limits!(comp, (up = (pmax*0.2)/60, down = (pmax*0.2)/60)) # Ramp rate is expected to be in MW/min
end
```

25. We can review all the data we just updated in a batch:
```
show_components(system, ThermalStandard, [:base_power, :active_power_limits, :ramp_limits])
```

## Time-series data

26. See a summary:
```
show_time_series(system)
```

Attention!: the function is sensitive to which Units Base settings you have defined

27. Get the entire year-long time series:
```
get_time_series_array(SingleTimeSeries, g, "max_active_power")
```

28. Get one forecast window:
```
get_time_series_array(Deterministic, g, "max_active_power")
```

## Save our work!

29. We have turned off the renewable generators and updated CT ramp rates. Export the system to json and .h5 files for the time series:
```
to_json(system, "my_new_RTS_DA_system.json")
```

## Takeaways:
- Use show_components and get_components to filter for the data or access that you need, without large memory allocations
- Use getters (get_) and setters (set_) to access and change data, with per-unitization handling built it

# Moving to Production Cost Modeling

## Set up the optimizer

30. Before you run the simulation, you need to define the solver parameters:
Xpress Parameter Documentation: https://www.fico.com/fico-xpress-optimization/docs/latest/solver/optimizer/HTML/GUID-3BEAAE64-B07F-302C-B880-A11C2C4AF4F6.html
HiGHS Parameter Documentation: https://ergo-code.github.io/HiGHS/dev/
Gurobi Parameter Documentation: https://www.gurobi.com/documentation/current/refman/parameters.html
```
solver = optimizer_with_attributes(
        HiGHS.Optimizer,
        "time_limit" => 150.0, 
        "threads" => 12,
        "log_to_console" => true,
        "mip_abs_gap" => 1e-5
    )
```

31. Set up a default unit commitment model
```
template = template_unit_commitment()
```

How to learn more about the different formulations? The Formulation Library: https://nrel-sienna.github.io/PowerSimulations.jl/stable/formulation_library/Introduction/

32. Apply that template to our data to make a complete model, with both data and equations:
```
models = SimulationModels(
    decision_models=[
        DecisionModel(template, system, optimizer=solver, name="UC"),
    ],
)
```
We just have one model -- the unit commitment model.


33. In addition to defining the formulation template, sequential simulations require
definitions for how information flows between problems to set the initial conditions
of the next problem:
```
DA_sequence = SimulationSequence(
    models=models,
    ini_cond_chronology=InterProblemChronology(),
)
```

34. Finally, put together the models and the flow of information between them to simulate
3 days of a unit commitment:
```
sim = Simulation(
    name = "default_UC",
    steps = 3,
    models=models,
    sequence=DA_sequence,
    simulation_folder=results_dir,
)
```

35. Build the simulation, which does all the equation preparation before executing:
```
build!(sim, recorders = [:simulation])
```

36. Simulate!
```
execute!(sim)
```

37. Now, load in the simulation results from the output files:
```
results = SimulationResults(sim)
uc_results = get_decision_problem_results(results, "UC")
```

38. Let's visualize the results
```
plot_fuel(uc_results);
```

39. We can also look at how much it cost to operate the thermal generators at each time step:
```
costs = read_realized_expressions(uc_results, list_expression_names(uc_results))["ProductionCostExpression__ThermalStandard"]
```
Wind, solar, and hydro have 0 operating cost and do not contribute to total cost in this model

40. We can sum over the set of generators and time-steps to get total production cost for this window
```
sum(sum(eachcol(costs[!, 2:end])))
```

## Now, let's change some things in the system:

41. Now, let's change how the thermal generators behave by adding in ramping and minimum up
and down time constraints:
```
set_device_model!(template, ThermalStandard, ThermalStandardUnitCommitment)
```
You can see the formulations options in the documentation: 
https://nrel-sienna.github.io/PowerSimulations.jl/stable/formulation_library/ThermalGen/

42. We've changed our template, so we need to re-build the DecisionModels within our simulation, which
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
    name = "UC_with_ramping",
    steps = 3,
    models=models,
    sequence=DA_sequence,
    simulation_folder=results_dir,
)
```

43. Now, let's re-build, and execute our simulation:
```
build!(sim, recorders = [:simulation]);
execute!(sim)
```

44. Let's re-extract the results and take a look at the impact on the dispatch stack:
```
results = SimulationResults(sim)
uc_results = get_decision_problem_results(results, "UC")
plot_fuel(uc_results);
```

45. Finally, let's extract the operating costs again and see the impact on total cost:
```
costs = read_realized_expressions(uc_results, list_expression_names(uc_results))["ProductionCostExpression__ThermalStandard"]
sum(sum(eachcol(costs[!, 2:end])))
```

## Adding in renewables

46. Now, let's add in those large-scale wind and solar plants by making them
available again:
```
for g in get_components(RenewableDispatch, system)
    set_available!(g, true)
end
```

47. We've changed the data, not the template formulations, so this time we could just re-build and simulate,
but let's redefine our simulation name:
```
sim = Simulation(
    name = "UC_with_renewables",
    steps = 3,
    models=models,
    sequence=DA_sequence,
    simulation_folder=results_dir,
)

build!(sim, recorders = [:simulation]);
execute!(sim)
```

48. Let's re-extract the results and take a look at the impact on the dispatch stack:
```
results = SimulationResults(sim)
uc_results = get_decision_problem_results(results, "UC")
plot_fuel(uc_results);
```

49. Finally, let's extract the operating costs again and see the impact on total cost:
```
costs = read_realized_expressions(uc_results, list_expression_names(uc_results))["ProductionCostExpression__ThermalStandard"]
sum(sum(eachcol(costs[!, 2:end])))
```

## Complete our scenario analysis with PowerAnalytics

50. Map to all our results scenarios:
```
results_all = create_problem_results_dict(results_dir, "UC"; populate_system=true)
```

51. Define which time-independent metrics we're interested in: 
```
timeless_computations = [PA.calc_sum_objective_value, PA.calc_sum_solve_time, PA.calc_sum_bytes_alloc]
timeless_names = ["Objective Value", "Solve Time", "Memory Allocated"];
```

For more, see PowerAnalytics' Built-In Metrics: https://nrel-sienna.github.io/PowerAnalytics.jl/previews/PR43/reference/public/#Built-in-Metrics

52. We'll also make selectors, which help us calculate our metrics across whichever dimensions and/or groupings
are of interest:
```
thermal_standard_selector_sys = make_selector(ThermalStandard; groupby=:all)
renewable_dispatch_selector_sys = make_selector(RenewableDispatch; groupby=:all)
```

53.  Let's also calculate some time dependent metrics of performance:
```
time_computations = [
    (PA.calc_total_cost, thermal_standard_selector_sys, "Total Cost (\$)"),
    (PA.calc_active_power, thermal_standard_selector_sys, "Thermal Generation (MWh)"),
    (PA.calc_curtailment, renewable_dispatch_selector_sys, "Renewables Curtailment (MWh)"),
]
```

54. A few utility functions help us calculate across metrics and scenarios:
```
function analyze_one(results)
    time_series_analytics = compute_all(results, time_computations...)
    aggregated_time = aggregate_time(time_series_analytics)
    computed_all = compute_all(results, timeless_computations, nothing, timeless_names)
    all_time_analytics = hcat(aggregated_time, computed_all)
    return time_series_analytics, all_time_analytics
end
function save_one(output_dir, time_series_analytics, all_time_analytics)
    CSV.write(joinpath(output_dir, "summary_dataframe_new.csv"), time_series_analytics)
    CSV.write(joinpath(output_dir, "summary_stats_new.csv"), all_time_analytics)
end
function post_processing(all_results) 
    summaries = []
    for (scenario_name, results) in pairs(all_results)
        println("Computing for scenario: ", scenario_name)
        (time_series_analytics, all_time_analytics) = analyze_one(results)
        save_one(results.results_output_folder, time_series_analytics, all_time_analytics)
        push!(summaries, hcat(DataFrame("Scenario" => scenario_name), all_time_analytics))
    end

    summaries_df = vcat(summaries...)
    CSV.write("all_scenarios_summary_new.csv", summaries_df)
    return summaries_df
end
```

55. Post-process all scenarios together:
```
post_processing(results_all)
```