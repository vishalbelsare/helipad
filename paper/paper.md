---
title: 'Helipad: A Framework for Agent-Based Modeling in Python'
tags:
	- Agent-based modeling
	- Computational modeling
	- Python
	- Heterogeneous agent modeling
	- Computational economics
	- Computational biology
authors:
	- name: Cameron Harwick
	  orcid: 0000-0003-2703-1627
	  affiliation: 1
affiliations:
	- name: Assistant Professor of Economics, SUNY Brockport
	  index: 1
date: 29 June 2023
bibliography: helipad.bib
---

# Summary

Agent-based modeling tools commonly trade off usability against power and vice versa. On the one hand, full development environments like NetLogo feature a shallow learning curve, but have a relatively limited proprietary language. Others written in Python or Matlab, for example, have the advantage of a full-featured language with a robust community of third-party libraries, but are typically more skeletal and require more setup and boilerplate in order to write a model. Helipad is introduced to fill this gap. Helipad is a new agent-based modeling framework for Python with the goal of a shallow learning curve, extensive flexibility, minimal boilerplate, and powerful yet easy to set up visualization, in a full Python environment. We summarize Helipad’s general architecture and capabilities, and briefly preview a variety of models from a variety of disciplines, including multilevel models, matching models, network models, spatial models, and others.

# Statement of Need

Agent-based modeling is an alternative to traditional analytical modeling that simulates interactions among agents algorithmically [see @Bonabeau:2002; @EpsteinAxtell:1996 for overviews]. It is particularly valuable for modeling dynamic systems that are difficult to describe with a closed-form analytical solution. In an agent-based model, discrete agents are programmed to interact under conditions that simulate the environment in question, and carry their state with them. Agent behavior can be directly specified in an open-ended way, allowing models to be interpreted much more easily than highly stylized analytical models where agent state can be specified only at an aggregate level. In addition, equilibrium can emerge from such a model – or not – without building the equilibrium into the assumptions of the model [@Arthur:2015 ch. 1].

Agent-based modeling necessarily involves programming, and there are numerous frameworks available in a variety of languages. NetLogo is one popular integrated development environment (IDE) that includes a code editor, a proprietary language, and extensive visualization tools, especially for spatial models [@BanosEtAl:2015; @WilenskyRand:2015; @RailsbackGrimm:2012]. Its popularity is due to its shallow learning curve and its integrated environment: very little setup is necessary, models can be easily packaged, and visualizations are simple to set up.

Nevertheless, NetLogo is limited in important ways. First, its language is only object-oriented in a very restricted sense, limiting some of the advantages of agent-based models that involve the states of individual agents.[^1] And second, while its self-containedness is an advantage in some respects, it also limits the ability to interact with outside libraries and to use code written for more traditional object-oriented languages.

Agent-based modeling frameworks in other languages on the other hand – for example, in Python [@KazilEtAl:2020], Java [@LukeEtAl:2018], MatLab, or Mathematica – have the potential to be far more powerful with the ability to draw on general language features, outside libraries, and wider communities of users. However, they are not integrated IDEs: they tend to be skeletal, to provide a basic structure with some visualization capabilities, but generally require a great deal more setup and boilerplate – especially for visualization – than an IDE like NetLogo.

This paper introduces a new agent-based modeling framework for Python, Helipad, to fill this gap. Helipad is not a full-fledged IDE (though it approaches one when used in a Jupyter notebook), but it has the goal of reducing boilerplate to a minimum and allowing models to be built, tested, and visualized in incremental steps, an important trait for rapid debugging [GilbertTroitzch:2005, 21]. In the following section we introduce Helipad’s general architecture and its array of modeling capabilities.[^2] Section 2 provides an overview of various sample models that have been written to demonstrate Helipad’s capabilities. The paper concludes with suggestions for future applications.

# Helipad’s Architecture

## Prerequisites

Helipad runs cross-platform on Python 3.8 or higher. It has minimal dependencies, requiring only Matplotlib [@Hunter:2007] and NetworkX [HagbergEtAl:2008] for visualizations, and Pandas [@McKinney:2010] for data collection, both of which in turn rely on Numpy [@HarrisEtAl:2020]. Shapely is optional at install time, but necessary for geospatial models.

Helipad can be run “headlessly” or with a GUI, which consists of a control panel and/or a visualization window. The GUI can be run in two different environments. First, a Helipad model can be run directly as a .py file, using Tkinter to provide a cross-platform windowed application interface (\autoref{fig:tkinter}).

![An Axelrod tournament model [@Axelrod:1980] running in Helipad’s Tkinter frontend.\label{fig:tkinter}](tkinter-frontend.png)

Helipad can also be run in a Jupyter notebook (\autoref{fig:jupyter}), a format that allows code and exposition to be mixed together and run in-browser [@KluyverEtAl:2016] in an environment very nearly approaching an IDE. Model code and features are identical in both frontends. Doing so requires, in addition to Jupyter Lab, the Ipywidgets and Ipympl libraries.

Helipad is available on PyPi.org, and is most easily installed using `pip install helipad` from the command line. It can also be installed with Conda using `conda install -c charwick helipad`.

## Hooks

There are two distinct strategies that can govern the relationship between user code and the code of an agent-based modeling framework. The two are not entirely mutually exclusive, but in practice, frameworks will hew toward one or the other.

1. _An imperative strategy._ Many frameworks are simply a collection of functions and classes that must be called or subclassed explicitly from a user-specified loop. The advantage of this strategy is that it provides explicit and precise control over every aspect of a model’s runtime. The disadvantage is that a great deal of boilerplate must be written in each model.
1. _A hook strategy._ Helipad, by contrast, incorporates the boilerplate and takes care of the looping, allowing user code to be inserted in specific places in the model’s runtime through hooks. The advantage of this strategy is that it allows a logical organization of code by topic and minimal boilerplate code. The disadvantage is that the framework makes certain assumptions about model structure, though there are ways to mitigate this disadvantage.

![A representation of the relationship between user code and hook code under imperative and hook strategies.\label{fig:hooks}](hook-diagram.svg)

Helipad uses a hook strategy, and minimizes the disadvantages by providing direct access to the model class in most hooks, allowing as much fine-grained control over the model’s runtime as would be possible with an imperative framework.

Helipad provides a loop structure for a model, into the elements of which user code can be inserted via hooks. Some of these will be critical to a model’s functioning (e.g. the agents’ step function); others offer low-level control over various aspects of the model (e.g. the `cpanel` hooks).

The `@heli.hook` decorator inserts a user function into these specified points in the model’s runtime. For example, the following agent step function instructs agents to work in the first stage of each period, and consume their product in the second stage.

```
from helipad import *
heli = Helipad()

@heli.hook
def agentStep(agent, model, stage):
	if stage==1: agent.wealth += produce(agent.id, agent.laborSupply)
	elif stage==2:
		agent.utility += sqrt(agent.wealth)
		agent.wealth = 0
		agent.laborSupply = random.normal(100, 10)
```

To tell Helipad to run this code for every agent every stage of every period, the `@heli.hook` decorator is added to the top of the function, which is named so that Helipad runs it during the `agentStep` hook. The decorator can also take the hook name as an argument (e.g. `@heli.hook('agentStep')`) and be placed above a function with any name.

There are a few things to notice about this example.

1. Helipad passes three arguments to any function used in the `agentStep` hook: `agent`, `model`, and `stage`. The documentation specifies the exact function signature to be used for each hook.
1. The `agent` and `model` objects are passed as arguments, allowing access to their properties and methods during each agent step.
1. Multiple functions can be added to a single hook, in which case they will be executed in the order they were registered (though the `prioritize` argument can be used to move a hook to the front of the queue).

## Agents, Breeds, Goods, and Primitives

Helipad provides for agent heterogeneity through agent _breeds_. A breed is registered during model setup, and assigned to an agent when it is initialized, whether at the beginning of the model run or through reproduction. An agent’s breed can be accessed with the `agent.breed` property.

Goods are items that agents can hold stocks of, which are kept track of in the `agent.stocks` property. Goods are also registered during model setup, along with user-defined per-agent properties (for example, agents might have a reservation price for each good), and can be exchanged using `agent.trade()`. One good may optionally be used as a medium of exchange, which allows the `agent.buy()` and `agent.pay()` functions to be used. Agents can optionally take a variety of utility functions over these goods, including Cobb Douglass, Leonteif, and CES.

Agent primitives are a way to specify deeper heterogeneity than breeds. Primitives are registered using a separate agent class, subclassed from `baseAgent`, and their behavior is specified using separate hooks. For example, a model might use primitives to distinguish between permanently distinct `'buyer'` and `'seller'` agents that share no common code, while using breeds within each agent to distinguish separate buying and selling strategies. An agent cannot switch primitives once instantiated. Agents of a given primitive can be accessed using `model.agents[primitive]`, and by breed within that primitive with `model.agents[primitive][breed]`. The default primitive is `'agent'`, which is registered automatically at the initialization of a model. Agents of all primitives can be hooked with the `baseAgent` set of hooks.

One included primitive is the `MultiLevel` class, allowing for multi-level agent-based models where the agents at one level are themselves full models [@MathieuEtAl:2018]. `MultiLevel` therefore inherits from both the `baseAgent` and the `Helipad` classes.

Helipad’s `agent` class is well-suited to evolutionary models and genetic algorithms  [@Holland:1992; @Holland:1998]. Agents can reproduce both haploid and polyploid through a powerful `agent.reproduce()` method that allows child traits to be inherited in a variety of ways from one or multiple parents, along with mutations to those traits. Ancestry is tracked with a directed network (see below on networks). Agents keep track of their age in periods in the `agent.age` property, and can be killed with `agent.die()`.

## Parameters

Helipad constructs its control panel GUI primarily with user parameters that allow model variables to be adjusted before and, optionally, during model runtime. Parameters can be registered with `model.params.add()`. Helipad supports the following parameter types, depending on the format of the variable in question:

1. _Sliders,_ for numerical variables over a range. Sliders can also be set to slide over a discrete set of values, for example on a logarithmic scale.
1. _Checkboxes,_ for boolean variables.
1. _Menus,_ for categorical choices.
1. _Checkentries,_ for a variable equal to the value of a text box if a checkbox is checked, or False otherwise. The text box can be numeric or a string.
1. _Checkgrids,_ for a series of related booleans.
1. _Hidden_ parameters can be retrieved and set in user code, but do not display in the control panel.

\autoref{fig:tkinter} above shows six slider parameters and two checkgrids, along with two checkentries and a slider in the top configuration section.

Helipad also allows parameters to be specified on a per-breed and per-good basis, with the parameter taking separate values for each registered breed or good. Current parameter values can be accessed at any point in model code using `model.param()`. Parameters can also be set in model code, in which case they are also reflected in the control panel GUI.

Helipad provides a shocks API for numeric parameters. A shock consists of a parameter, a timer function that takes the current model time and outputs a boolean, and a value function that takes the current parameter value and outputs a new value. The value function will then update the parameter value whenever the timer function returns `True`. This can be used for one-time, regular, or stochastic shocks, possibly generating data for impulse response functions as in @Harwick:2018. Registered shocks can be toggled on and off before and during the model’s runtime in the control panel.

## Data and Visualization

Ease of visualization is the key advantage of Helipad over other Python-based agent-based modeling frameworks. Unlike some others which have the user create plots directly in (e.g.) Matplotlib, or require the launch of a webserver, Helipad includes an extensible visualization API to manage a full-featured and interactive visualization window. It can also be easily extended with custom visualizations written in Python, without a frontend/backend division; thus Helipad models – even with custom visualizations – can generally be self-contained in one .py file without becoming unwieldy. Visualizations of various types can be registered in only a few lines of code. For example, in the Price Discovery model (described in the following section) where the `lastPrice` property is set in the `agentStep` hook, the following five lines are all that is necessary to collect trade price data and register a live-updating plot of the geometric mean with percentile bars as the model runs.

```
from helipad.visualize import TimeSeries
viz = heli.useVisual(TimeSeries)

heli.data.addReporter('ssprice', heli.data.agentReporter('lastPrice', 'agent', stat='gmean', percentiles=[0,100]))
pricePlot = viz.addPlot('price', 'Price', logscale=True, selected=True)
pricePlot.addSeries('ssprice', 'Soma/Shmoo Price', '#119900')
```

Model data is collected each period into _reporters,_ corresponding to data columns, that return a value each period and record column data. The data as a whole can be accessed during model runtime, exported to a Pandas dataframe, analyzed after the model’s run using the `terminate` hook, or exported to a CSV through the control panel.

Parameter values are automatically registered as reporters. Helipad also includes functions to generate reporter functions for summary statistics (including arithmetic and geometric means, sum, maximum, minimum, standard deviation, and percentiles) of agent properties, as in the code block above, but reporter functions can be entirely user-defined as well. For reporters generated as summary statistics of agent properties, percentile marks and ± multiples of the standard deviation can be automatically plotted above and/or below the mean, as additional dotted lines above and below a time series line, or as error bars on a bar chart. Reporters can also be registered as a decaying average of any strength with the `smooth` parameter.

There are two included overarching visualizers: `TimeSeries` and `Charts`, for diachronic (over time) and synchronic (a particular point in time) data, respectively. Custom visualizations can also be written using the extensible `BaseVisualization` or `MPLVisualization` classes, the latter of which provides low-level access to the Matplotlib API while also maintaining important two-way links between the visualizer and the underlying model, such as automatic updating and interactivity through event handling. Both `TimeSeries` and `Charts` divide the visualization window into _Plots,_ areas in the graph view onto which data series can be plotted as the model runs.

`TimeSeries` stacks its plots vertically with a separate vertical axis for each plot, and model time as the shared horizontal axis. The visibility of plots can be toggled in the control panel prior to the model’s run. Plots can be displayed on a logarithmic or linear scale. Once a plot area is registered, reporters can be drawn to it by registering a _series_ that connects a reporter function to a plot area. \autoref{fig:tkinter}, for example, shows only one active plot with multiple series displayed on it. When this is done, the plot area will update live with reporter data as the model runs. Series can be drawn as independent lines, or stacked on top of one another within a plot with the `stack` parameter, for example if the sum of several series is important.

The `Charts` visualizer is divided into a grid of plots, where slice-in-time data of any form can be plotted and updated live as the model runs. The `Charts` visualizer also features a slider to scrub through time and display the state on each plot at any point in the model history. Bundled plot types include bar charts, network graph diagrams (including networks laid on top of a spatial coordinate grid), spatial maps, scatterplots of agent properties, and bar charts, all of which can be displayed alongside one another in the visualization window. Spatial models can be set up with either rectangular or polar coordinate systems (which are useful in certain ontogenetic models; see e.g. @Kauffman:1993, 556), the former of which can wrap on either or both dimensions for cylindrical or toroidal geometries.  Arbitrary polygonal patches can also be imported from GIS files and used in geospatial models.

Custom plot types can be registered and displayed within the `Charts` visualizer by extending the `ChartPlot` class, and a tutorial notebook for building a custom 3D bar chart visualizer is included. Finally, the appearance of spatial and network plots can be extensively customized, with color, size, and text all set to convey meaningful per-agent information such as breed, wealth, age, ID, and so on.

![The `Charts` visualizer running in Jupyter Lab, displaying Network, Bar Chart, and Spatial plots.\label{fig:jupyter}](jupyter-frontend.png)

## Other Capabilities

In addition to linear activation and random activation style models, Helipad also supports matching models through a `match` hook that activates agents _n_ at a time (with _n_=2 for pairwise matching). Pairwise matching is an important feature of models in both monetary theory and game theory. Periods can be divided into multiple stages, with activation style (linear, random, or matching) customizable on a per-stage basis, for example with random activation in the first stage, order preserved in the second stage, and matching in the third stage.

Agents can be linked together with multiple named networks, directed or undirected, and weighted or unweighted. Agents can be explicitly connected with with an `agent.addEdge()` method, or a network of a given density can be generated automatically. This network structure can be exported for further analysis to dedicated network packages like NetworkX [@HagbergEtAl:2008]. Ancestry relationships, as mentioned earlier, are kept track of using a directed `'lineage'` network. Spatial models, too, are created by initializing a `Patch` primitive and creating an undirected grid network among the patches, with optional diagonal connections. Networks among agents can be displayed on top of a spatial map, and various network layouts can be rotated through in the visualizer. Agents in a spatial model acquire `position` and `orientation` properties, along with methods for motion appropriate to the coordinate system.

Helipad also includes tools for batch processing model runs, most importantly, a parameter sweep function that runs a model up to a given termination period through every permutation of one or more parameters.[^3] The resulting data from each run can be exported to CSV files or passed to further user code for analysis.

Finally, Helipad supports _events,_ which trigger when the model satisfies a user-defined criterion, registered by placing an `@event` decorator over a criterion function. For example, an event might be “population stabilizes” or “death rate exceeds 5% over 10 periods”. An event may repeat or not. On the firing of an event, Helipad records the current model data into the Event object and notifies the user in the visualization window: `TimeSeries` draws a line at the event mark, and `Charts` flashes the window. Events can be used to stop a model after it reaches a certain state, or to automatically move the model into a different phase.

# Example Models

This section describes the various sample models that have been written in Helipad, and the features they exemplify, to give a sense of the variety of models possible. All of the following models can be downloaded from Helipad’s Github page.[^4] This list, of course, is by no means exhaustive.

## Helicopter drops

@Harwick:2018, Helipad’s origin and namesake, is a model of relative price responses to monetary shocks in the presence or absence of a banking system, i.e. depending on whether money is injected through helicopter drops or open-market operations. The model features two agent breeds – Hobbits and Dwarves – who consume one of two goods, jam and axes, respectively, and who have differing demand for money set with a per-breed parameter. The relative demand for money balances and goods is determined by a CES utility function. The model also features a store and (optionally) a bank, both registered as separate primitives. The control panel uses the `callback` argument of `model.params.add()` to enforce relationships between certain parameters, and post-model analysis using `statsmodels` is run using the `'terminate'` hook.

## Matching models

A price discovery model with random matching is described in @Caton:2020, ch. 9. In this model – significantly, written in under 50 lines of code – agents are randomly endowed with two goods, and repeatedly randomly paired to trade along the contract curve of a standard Edgeworth Box setup with Cobb-Douglas utility. The model runs until the per-period trade volume falls below a certain threshold, leading to convergence on a uniform equilibrium price. 

Another matching model is the @Axelrod:1980 tournament displayed in \autoref{fig:tkinter}. In the Axelrod tournament, strategies are assigned to breeds, and paired randomly against other strategies in 200-round repeated prisoner’s dilemmas. As it turns out, the much-celebrated dominance of tit-for-tat is not robust to the collection of strategies it plays against; indeed, in \autoref{fig:tkinter}, Grudger comes out ahead by a substantial margin.

## Evolutionary models

An evolutionary model is described in @Harwick:2021, where an agent’s reproductive fitness depends positively on a partially-heritable human capital parameter, but negatively on local population density. The fact that population density increases the economic returns to human capital leads to cyclical human capital dynamics. Not only the mean, but also the variance in human capital  over time can be seen with the plotted error bands.

Similarly, the @BowlesGintis:2011, ch. 7.1 model of the evolution of altruism through deme selection is included as a multi-level model, with the agents in the top-level model representing competing demes, and the agents of each deme representing individuals. Cooperation is selected against within demes, but demes with a higher proportion of cooperative members are more likely to prevail against and colonize demes with fewer cooperative members. Altruism can survive as a stable strategy provided the benefits to cooperation are high enough, and the likelihood of inter-deme conflict is sufficiently strong. Relative proportions of selfish and altruistic agents are plotted on a stacked plot adding up to 100%.

## Spatial models

Conway’s Game of Life [@Gardner:1970], a cellular automaton where a grid evolves according to simple rules but whose results cannot be predicted from the initial state without stepping through algorithmically, can be implemented as a spatial model in Helipad in just 27 lines of code, including 5 for interactivity in the visualizer, i.e. the ability to toggle cells on and off.

Finally, a standard spatial population model of equilibrium predator-prey relationships (in this case, sheep and grass) is included. The productivity of grass places a hard limit on the sheep population, especially when the latter reproduce sexually. The model can be easily extended to other coordinate systems, for example polar or geospatial models.

## Conclusion

Helipad is a powerful and extensible agent-based modeling framework for Python that ensures a shallow learning curve with a hook-based architecture. It has specialized tools for economic, biological, game-theoretic, and network models, but can be easily extended and combined with other Python libraries as well.

Since its initial public release, Helipad has added a number of significant modeling and architectural features, and is now API stable, ensuring backward-compatibility for future versions. The source code is open, and as agent-based simulations gain traction in social-scientific work, Helipad has the potential to aid the packagability and legibility model code – especially in notebook format – as well as to lower the barriers to creating new agent-based simulations in a variety of fields.

# Appendix. Glossary

**Agent.** The basic unit of an agent-based model. In Helipad, an agent is a self-contained object with code and functions for interacting with other agents and with the model environment.

**Breed.** An _agent_ type that allows heterogeneity within a _primitive_. Agents of different breeds share code, but conditional statements can make them behave differently. For example, the model might be split between agents playing a 'hawk' strategy and those playing a 'dove' strategy.

**Control panel.** The window (in the Tkinter frontend) or cell (in a Jupyter notebook) where model settings can be adjusted prior to and during a model’s run.

**Good.** Something which _agents_ can hold stocks of and receive utility from. Helipad registers goods and provides functions allowing them to be traded, bought, and so on.

**Edge.** A connection between two _agents_. An edge may have a weight, and may have a direction or no direction. The structure of edges in a model constitutes a _network_.

**Event.** A user-defined condition used to mark phases of a model. An event is triggered when its criterion function first returns `True`. It then records the model time and marks it in the visualization area, and records all the data from that period.

**Hook.** A predetermined place in the model’s runtime where user code can be inserted.

**Matching Model.** A model where a _period_ consists in _agents_ being matched in groups, rather than being stepped individually.

**Money.** A _good_ serving as a medium of exchange, allowing the use of monetary functions such as `agent.buy()`, `agent.pay()`, and so on.

**Multi-level Model.** A model in which _agents_ are themselves full models with sub-agents. The `MultiLevel` _primitive_ in this case might represent firms, demes, or other kinds of groups.

**Network.** A structure of connections (_edges_) among agents. There are dedicated Python tools for analyzing networks, such as NetworkX, which can interface with Helipad. Networks may be visualized in a variety of layouts, including on top of a _spatial model_.

**Parameter.** A variable whose value can be adjusted from the _control panel_. Parameters can be global, or split out on a per-_breed_ or per-_good_ basis (e.g. if the productivity of each good were to be set separately).

**Patch.** A fixed agent _primitive_ used in _spatial models,_ representing the coordinate grid upon which other agent primitives are placed. Patches are placed in a grid _network_ with connections to their  immediate neighbors, and – optionally – their diagonal neighbors as well. Patch shape will depend on the spatial geometry: a rectangle (in a rectilinear coordinate system), an annular sector (in a polar coordinate system), or an arbitrary polygon (in a geospatial model).

**Period.** An agent-based model repeatedly runs the _step_ functions of all the agents. Each time this happens constitutes one period, and model time is kept by the number of periods elapsed since initialization.

**Plot.** A discrete area in the visualization window. The `Charts` visualizer, for example, can take a variety of plot types including network, spatial, and bar plots, and `TimeSeries` takes a time series plot. Some plots, like the bar chart and time series plot, can take individual _series_.

**Primitive.** An agent type more basic than a _breed_. For example, _agents_ could be divided by breed to employ different strategies, but a different type of agent entirely – for example, firms – could be registered as a separate primitive, with its own set of breeds. By default all agents are registered with the `'agent'` primitive. Agents cannot change primitives once initialized.

**Reporter.** A function that gathers model data and outputs a numerical value. Registered reporters are run each period to collect data. Helipad provides functions to generate reporter functions for certain common data types, but custom reporter functions can also be written.

**Series.** A line on a time series _plot_ or a bar on a bar chart plot, drawing model data from a _reporter_.

**Shock.** An exogenous shift in a _parameter_ value. Shocks may be timed automatically with a timer function, or initiated by the user via buttons in the _control panel_.

**Spatial model.** A model where _agents_ have a spatial location on a grid of _patches_. Instantiating a spatial model provides agents with methods and properties for orientation and motion, depending on the geometry.

**Stage.** The division of a _period_ into multiple parts, for example if _agents_ must all run some code before any of them run other code.

**Step.** A function that runs each _period_. Each _agent_ has a step function, as well as the model as a whole.

# References

[^1]: Specifically, while agents are themselves objects, there is no way to create user-defined objects.
[^2]: A complete and up-to-date API reference can be found at [helipad.dev](https://helipad.dev).
[^3]: The checkentry is the only parameter type with an open-ended value range, and thus cannot be swept. All other parameter types have finite value ranges.
[^4]: See [github.com/charwick/helipad/tree/master/sample-models](https://github.com/charwick/helipad/tree/master/sample-models). Some are also available as [Jupyter notebooks](https://github.com/charwick/helipad/tree/master/sample-notebooks).