# Examples

## Farmer problem

The following defines the well-known "Farmer problem", first outlined in [Introduction to Stochastic Programming](https://link.springer.com/book/10.1007%2F978-1-4614-0237-4), in StochasticPrograms. The problem revolves around a farmer who needs to decide how to partition his land to sow three different crops. The uncertainty comes from not knowing what the future yield of each crop will be. Recourse decisions involve purchasing/selling crops at the market.

```@example farmer
using StochasticPrograms
using GLPK
```
An example implementation of the farmer problem is given by:
```@example farmer
farmer_model = @stochastic_model begin
    @stage 1 begin
        @parameters begin
            Crops = [:wheat, :corn, :beets]
            Cost = Dict(:wheat=>150, :corn=>230, :beets=>260)
            Budget = 500
        end
        @decision(model, x[c in Crops] >= 0)
        @objective(model, Min, sum(Cost[c]*x[c] for c in Crops))
        @constraint(model, sum(x[c] for c in Crops) <= Budget)
    end
    @stage 2 begin
        @parameters begin
            Crops = [:wheat, :corn, :beets]
            Required = Dict(:wheat=>200, :corn=>240, :beets=>0)
            PurchasePrice = Dict(:wheat=>238, :corn=>210)
            SellPrice = Dict(:wheat=>170, :corn=>150, :beets=>36, :extra_beets=>10)
        end
        @uncertain ξ[c in Crops]
        @variable(model, y[p in setdiff(Crops, [:beets])] >= 0)
        @variable(model, w[s in Crops ∪ [:extra_beets]] >= 0)
        @objective(model, Min, sum(PurchasePrice[p] * y[p] for p in setdiff(Crops, [:beets]))
                   - sum(SellPrice[s] * w[s] for s in Crops ∪ [:extra_beets]))
        @constraint(model, minimum_requirement[p in setdiff(Crops, [:beets])],
            ξ[p] * x[p] + y[p] - w[p] >= Required[p])
        @constraint(model, minimum_requirement_beets,
            ξ[:beets] * x[:beets] - w[:beets] - w[:extra_beets] >= Required[:beets])
        @constraint(model, beets_quota, w[:beets] <= 6000)
    end
end
```
The three yield scenarios can be defined through:
```@example farmer
ξ₁ = Scenario(wheat = 3.0, corn = 3.6, beets = 24.0, probability = 1/3)
ξ₂ = Scenario(wheat = 2.5, corn = 3.0, beets = 20.0, probability = 1/3)
ξ₃ = Scenario(wheat = 2.0, corn = 2.4, beets = 16.0, probability = 1/3)
```
We can now instantiate the farmer problem using the defined stochastic model and the three yield scenarios:
```@example farmer
farmer = instantiate(farmer_model, [ξ₁,ξ₂,ξ₃], optimizer = GLPK.Optimizer)
```
Printing:
```@example farmer
print(farmer)
```
We can now optimize the model:
```@example farmer
optimize!(farmer)
x = optimal_decision(farmer)
println("Wheat: $(x[1])")
println("Corn: $(x[2])")
println("Beets: $(x[3])")
println("Profit: $(objective_value(farmer))")
```
Finally, we calculate the stochastic performance of the model:
```@example farmer
println("EVPI: $(EVPI(farmer))")
println("VSS: $(VSS(farmer))")
```

## Continuous scenario distribution

As an example, consider the following generalized stochastic program:
```math
\DeclareMathOperator*{\minimize}{minimize}
\begin{aligned}
 \minimize_{x \in \mathbb{R}} & \quad \operatorname{\mathbb{E}}_{\omega} \left[(x - \xi(\omega))^2\right] \\
\end{aligned}
```
where ``\xi(\omega)`` is exponentially distributed. We will skip the mathematical details here and just take for granted that the optimizer to the above problem is the mean of the exponential distribution. We will try to approximately solve this problem using sample average approximation. First, lets try to introduce a custom discrete scenario type that models a stochastic variable with a continuous probability distribution. Consider the following implementation:
```julia
using StochasticPrograms
using Distributions

struct DistributionScenario{D <: UnivariateDistribution} <: AbstractScenario
    probability::Probability
    distribution::D
    ξ::Float64

    function DistributionScenario(distribution::UnivariateDistribution, val::AbstractFloat)
        return new{typeof(distribution)}(Probability(pdf(distribution, val)), distribution, Float64(val))
    end
end

function StochasticPrograms.expected(scenarios::Vector{<:DistributionScenario{D}}) where D <: UnivariateDistribution
    isempty(scenarios) && return DistributionScenario(D(), 0.0)
    distribution = scenarios[1].distribution
    return ExpectedScenario(DistributionScenario(distribution, mean(distribution)))
end
```
The fallback [`probability`](@ref) method is viable as long as the scenario type contains a [`Probability`](@ref) field named `probability`. The implementation of [`expected`](@ref) is somewhat unconventional as it returns the mean of the distribution regardless of how many scenarios are given.

We can implement a sampler that generates exponentially distributed scenarios as follows:
```julia
struct ExponentialSampler <: AbstractSampler{DistributionScenario{Exponential{Float64}}}
    distribution::Exponential

    ExponentialSampler(θ::AbstractFloat) = new(Exponential(θ))
end

function (sampler::ExponentialSampler)()
    ξ = rand(sampler.distribution)
    return DistributionScenario(sampler.distribution, ξ)
end
```
Now, lets attempt to define the generalized stochastic program using the available modeling tools:
```julia
using Ipopt

sm = @stochastic_model begin
    @stage 1 begin
        @decision(model, x)
    end
    @stage 2 begin
        @uncertain ξ from DistributionScenario
        @objective(model, Min, (x - ξ)^2)
    end
end
```
```julia
Two-Stage Stochastic Model

minimize f₀(x) + 𝔼[f(x,ξ)]
  x∈𝒳

where

f(x,ξ) = min  f(y; x, ξ)
              y ∈ 𝒴 (x, ξ)
```
The mean of the given exponential distribution is ``2.0``, which is the optimal solution to the general problem. Now, lets create a finite sampled model of 1000 exponentially distributed numbers:
```julia
sampler = ExponentialSampler(2.) # Create a sampler

sp = instantiate(sm, sampler, 1000, optimizer = Ipopt.Optimizer) # Sample 1000 exponentially distributed scenarios and create a sampled model
```
```julia
Stochastic program with:
 * 1 decision variable
 * 1 recourse variable
 * 1000 scenarios of type DistributionScenario
Solver is default solver
```
By the law of large numbers, we approach the generalized formulation with increasing sample size. Solving yields:
```julia
optimize!(sp)

println("Optimal decision: $(optimal_decision(sp))")
println("Optimal value: $(objective_value(sp))")
```
```julia
Optimal decision: [2.0397762891884894]
Optimal value: 4.00553678799426
```
Now, due to the special implementation of the [`expected`](@ref) function, it actually holds that the expected value solution solves the generalized problem. Consider:
```julia
println("Expected value decision: $(expected_value_decision(sp)")
println("VSS: $(VSS(sp))")
```
```julia
EVP decision: [2.0]
VSS: 0.00022773669794418083
```
Accordingly, the VSS is small.
