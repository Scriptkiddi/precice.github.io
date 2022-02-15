---
title: Waveform iteration for time interpolation of coupling data
permalink: couple-your-code-waveform.html
keywords: api, adapter, time, waveform, subcycling, multirate
summary: "With waveform iteration, you can interpolate coupling data in time for higher-order time stepping and more stable subcycling."
---

{% experimental %}
These API functions are work in progress, experimental, and are not yet released. The API might change during the ongoing development process. Use with care."
{% endexperimental %}
{% note %}
This feature is only available for implicit coupling. Without loss of generality, we moreover only discuss the API functions `readBlockVectorData` and `writeBlockVectorData` in the examples.
{% endnote %}

preCICE allows the participants to use subcycling - meaning: to work with individual time step sizes smaller than the time window size. Note that participants always have to synchronize at the end of each *time window*. If you are not sure about the difference between a time window and a time step or you want to know how subcycling works in detail, see ["Step 5 - Non-matching time step sizes" of the step-by-step guide](couple-your-code-timestep-sizes.html). In the following section, we take a closer look at the exchange of coupling data when subcycling, and advanced techniques for interpolation of coupling data inside of a time window.

## Exchange of coupling data with subcycling

preCICE only exchanges data at the end of the last timestep in each time window – the end of the time window. By default, preCICE only exchanges data that was written at the very end of the time window. This approach automatically leads to discontinuities or "jumps" when going from one time windows to the next. Coupling data has a constant value (in time) within one coupling window. This leads to lower accuracy of the overall simulation (for details, see [^1]).

### Example for subcycling without waveform iteration

The figure below visualizes this situation for a single coupling window ranging from $$t_\text{ini}$$ to $$t_\text{ini}+\Delta t$$:

![Coupling data exchange without interpolation](images/docs/couple-your-code/couple-your-code-waveform/WaveformConstant.png)

The two participants Dirichlet $$\mathcal{D}$$ and Neumann $$\mathcal{N}$$ use their respective timestep sizes $$\delta t$$ and produce coupling data $$c$$ at the end of each timestep. But only the very last samples $$c_{\mathcal{N}\text{end}}$$ and $$c_{\mathcal{D}\text{end}}$$ are exchanged. If the Dirichlet participant $$\mathcal{D}$$ calls `readBlockVectorData`, it always receives the same value $$c_{\mathcal{N}\text{end}}$$ from the Neumann participant $$\mathcal{N}$$, independent from the current timestep.

## Linear interpolation in a time window

A simple solution to reach higher accuracy is to apply linear interpolation inside of a time window to get smoother coupling boundary conditions. With this approach time-dependent functions (so-called *waveforms*) are exchanged between the participants. Since these waveforms are exchanged iteratively in implicit coupling, we call this procedure *waveform iteration*. Exchanging waveforms leads to a more robust subcycling and allows us to support higher order time stepping (for details, see [^1]).

### Example for waveform iteration with linear interpolation

Linear interpolation between coupling boundary conditions of the previous and the current time window is illustrated below:

![Coupling data exchange with linear interpolation](images/docs/couple-your-code/couple-your-code-waveform/WaveformLinear.png)

If the Dirichlet participant $$\mathcal{D}$$ calls `readBlockVectorData`, it samples the data from a time-dependent function $$c_\mathcal{D}(t)$$. This function is created from linear interpolation of the first and the last sample $$c_{\mathcal{D}0}$$ and $$c_{\mathcal{D}5}$$ created by the Neumann participant $$\mathcal{N}$$ in the current time window. This allows $$\mathcal{D}$$ to sample the coupling condition at arbitrary times $$t$$ inside the current time window.

## Experimental API for waveform iteration

If we want to improve the accuracy by using waveforms, this requires an extension of the existing API, because we need a way to tell preCICE where we want to sample the waveform. For this purpose, preCICE offers an experimental API, which is currently only supporting a single linear interpolation along the complete time window. Here, `readBlockVectorData` accepts an additional argument `relativeReadTime`. This allows us to choose the time where the interpolant should be sampled:

```cpp
// stable API with constant data in time window
void readBlockVectorData(int dataID, int size, const int* valueIndices, double* values) const;

// experimental API for waveform iteration
void readBlockVectorData(int dataID, int size, const int* valueIndices, double relativeReadTime, double* values) const;
```

`relativeReadTime` describes the time relatively to the beginning of the current timestep. This means that `relativeReadTime = 0` gives us access to data at the beginning of the timestep. Since we will call `advance(dt)` at a later point in time to finalize the timestep, `relativeReadTime = dt` gives us access to data at the end of the timestep.

{% note %}
The functionality of `writeBlockVectorData` remains unchanged, because the data at the beginning and at the end of the window are sufficient to create a linear interpolant over the window. Therefore, all samples but the one from the very last timestep in the time window are ignored. This might, however, change in the future (see [precice/#1171](https://github.com/precice/precice/issues/1171)).
{% endnote %}

The experimental API has to be activated in the configuration file via the `experimental` attribute. This allows us to define the order of the interpolant in the `read-data` tag of the corresponding `participant`. The default is constant interpolation (`waveform-order="0"`). The following example uses `waveform-order="1"` and, therefore, linear interpolation:

```xml
<solver-interface experimental="true" ... >
...
    <participant name="FluidSolver">
        <use-mesh name="FluidMesh" provide="yes"/>
        <write-data name="Forces" mesh="MyMesh"/>
        <read-data name="Displacements" mesh="FluidMesh" waveform-order="1"/>
    </participant>
...
</solver-interface>
```

## Usage example

We are now ready to extend the example from ["Step 6 - Implicit coupling"](couple-your-code-implicit-coupling.html) to use waveforms. Only few changes are necessary to sample the `Displacements` at the middle of the time window, which might be necessary for our specific application:

```cpp
...
precice_dt = precice.initialize();
while (not simulationDone()){ // time loop
  // write checkpoint
  ...
  dt = beginTimeStep(); // e.g. compute adaptive dt 
  dt = min(precice_dt, dt);
  if (precice.isReadDataAvailable()){ // always true, because we can sample at arbitrary points
    // sampling in the middle of the timestep
    precice.readBlockVectorData(displID, vertexSize, vertexIDs, 0.5 * dt, displacements);
    setDisplacements(displacements); // displacement at the middle of the timestep
  }
  solveTimeStep(dt); // might be using midpoint rule for time-stepping
  if (precice.isWriteDataRequired(dt)){ // only true at the end of the time window
    computeForces(forces);
    precice.writeBlockVectorData(forceID, vertexSize, vertexIDs, forces);
  }
  precice_dt = precice.advance(dt);
  // read checkpoint & endTimeStep  
  ...
}
...
```


As described in ["Step 7 - Data initialization"](couple-your-code-initializing-coupling-data), we can also use `initializeData` to provide initial data for the interpolation. If `initializeData` is not called, preCICE uses zero initial data for constructing the interpolant.

## Literature

[^1]: Rüth, B, Uekermann, B, Mehl, M, Birken, P, Monge, A, Bungartz, H-J. Quasi-Newton waveform iteration for partitioned surface-coupled multiphysics applications. Int J Numer Methods Eng. 2021; 122: 5236– 5257. [https://doi.org/10.1002/nme.6443](https://doi.org/10.1002/nme.6443)