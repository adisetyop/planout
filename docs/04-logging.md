---
id: logging
title: Logging
layout: docs
permalink: /docs/logging.html
prev: random-operators.html
next: namespaces.html
---

You will usually want to log which units (e.g., users) are exposed to your experiment.

Logging this information enables monitoring your experiment and improving your analysis of the results. In particular, many experiments only change your service for the small portion of users that use a particular part of the service; keeping track of these users will make your analysis more precise.

The `SimpleExperiment` class providing functionality for logging data to a file using Python's `logger` module. Additionally, you can extend `Experiment` to log to one or more [other data stores instead](extending-logging.html), such as a database, scribe, or a real-time counter. 

## Anatomy of a log
Consider Experiment 1 from the [PlanOut paper](https://www.facebook.com/download/255785951270811/planout.pdf):

```python
class Exp1(SimpleExperiment):
  def assign(self, params, userid):
    params.group_size = UniformChoice(choices=[1, 10], unit=userid);
    params.specific_goal = BernoulliTrial(p=0.8, unit=userid);
    if params.specific_goal:
      params.ratings_per_user_goal = UniformChoice(
        choices=[8, 16, 32, 64], unit=userid)
      params.ratings_goal = params.group_size * params.ratings_per_user_goal
    return e
```

It takes a `userid` as input, and assigns three paramers, `group_size`, `specific_goal`, and `ratings goal`. It does not specify a [custom salt or experiment name](how-planout-works.html), so the experiment salt and parameter name are automatically set to the class name `Exp3`. The default logger in `SimpleExperiment` log all of these fields:

```json
{
  "inputs": {
    "userid": 9
  },
  "name": "Exp1",
  "checksum": "2b0c4cfb",
  "params": {
    "group_size": 1,
    "specific_goal": 0
  },
  "time": 1396513178,
  "salt": "Exp1",
  "event": "exposure"
}
```

In addition, a `checksum` field is attached. This is part of an MD5 checksum of
your code, so that analysts can keep track of when an experiments' assignments
have potentially changed: whenever the checksum changes, the assignment
procedure code is different, and whenever the salt changes, the assignments
will be completely different.

`SimpleExperiment` computes checksums from the code contained in your class'
`assign()` method.  Because of the way the Python interpreter works, this
checksum can only be computed when Python is running in non-interactive mode,
so if you are following along this tutorial via the Python shell, you won't
see a checksum in your log file.

## Types of logs

### Auto-exposure logging
By default, exposures are logged once per instance of an experiment object when you first get a parameter. This is called auto-exposure logging. It is recommended for most situations, since you will want to track whenever a unit is exposed to your experiment. Generally, any unit for which a parameter has been retrieved should be counted as exposed, unless you wish to make further assumptions.

### Manual exposure logging
In some cases, you might want to choose exactly when exposures are logged. You can disable auto-exposure logging  with the `set_auto_exposure_logging` method and instead choose to directly call `log_exposure` to keep track of exposures.

Why might you want to do this? You might be adding experimental assignment information to other existing logs of outcome data, but many of the users who have outcome observations may not actually have been exposed. Other cases occur when some advance preparation of some components (e.g., UI) or data are required, but you can assume that parameter values set at this stage do not yet affect the user.

### Logging other events
You want to see how the parameters you are manipulating affect outcomes, conversions, or other events.
In many cases, you may do this analysis by later combining your exposure logs and other behavioral logs by joining on the relevant unit ID (e.g., userid). Especially for more real-time analysis, it can be useful to log outcome data with assignment data.

It can also be convenient to log these events along with assignment information. You can do this by calling the `log_event()` method directly, specifying an event name and a dictionary of any additional data to log.

Alternatively, you may have existing logging procedures for many of these events that you don't want to be part of your experiment definition. In this case, you could add experimental assignment information to these logs by instantiating an experiment, turning off auto-exposure logging for that instance, and adding parameter information to your logs.