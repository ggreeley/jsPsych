# Advanced Options for Trials

The parameters available for a trial depend primarily on what plugin is used for the trial. However, there are several options that do not depend on the particular plugin; they are available for all trials.

## The data parameter

The `data` parameter enables tagging the trial with additional properties. This can be useful for storing properties of the trial that are not directly apparent from the values that the plugin records. The `data` parameter value should be an object that contains key-value pairs.

A simple example is the [Flanker Task](https://en.wikipedia.org/wiki/Eriksen_flanker_task). In this experiment, participants respond to the direction of an arrow, pressing a key to the left for a left-pointing arrow (<) and a key to the right for a right-pointing arrow (>). The arrow appears in the center of *flankers*, or arrows that the participant should ignore. Those flankers can be congruent (>>>>>) or incongruent (<<><<).

A trial for the Flanker Task written with jsPsych might look like this:

```javascript
var trial = {
  type: 'html-keyboard-response',
  stimulus: '<<<<<',
  choices: ['f','j'],
  data: {
    stimulus_type: 'congruent',
    target_direction: 'left'
  }
}
```

Note the use of the data parameter to add a property `stimulus_type` with the value `congruent` and a property `target_direction` with the value `left`. Having these properties recorded directly in the data simplifies data analysis, making it easy to aggregate data by `stimulus_type` and/or `target_direction`.

## Inter-trial interval

The default inter-trial interval (ITI) in jsPsych is 0 ms. This can be adjusted at the experiment-wide level by changing the `default_iti` parameter in `jsPsych.init()`.

The ITI can also be controlled at the trial level through the `post_trial_gap` parameter. Setting this parameter to a positive integer *x* will cause a blank screen to display after the trial for *x* milliseconds.

```javascript
var trial = {
  type: 'html-keyboard-response',
  stimulus: 'There will be a 1.5 second blank screen after this trial.',
  post_trial_gap: 1500
}
```

## The on_start event

Immediately before a trial runs, there is an opportunity to run an arbitrary function through the `on_start` event handler. This event handler is passed a single argument containing an *editable* copy of the trial parameters. This event handler can therefore be used to alter the trial based on the state of the experiment, among other uses.

```javascript
var trial = {
  type: 'html-keyboard-response',
  stimulus: '<<<<<',
  choices: ['f','j'],
  data: {
    stimulus_type: 'congruent',
    target_direction: 'left'
  },
  on_start: function(trial){
    trial.stimulus = '<<><<';
    trial.data.stimulus_type = 'incongruent';
  }
}
```

## The on_finish event

After a trial is completed, there is an opportunity to run an arbitrary function through the `on_finish` event handler. This event handler is passed a single argument containing an *editable* copy of the data recorded for that trial. This event handler can therefore be used to update the state of the experiment based on the data collected or modify the data collected.

This can be useful to calculate new data properties that were unknowable at the start of the trial. For example, with the Flanker Task example above, the `on_finish` event could add a new property `correct`.

```javascript
var trial = {
  type: 'html-keyboard-response',
  stimulus: '<<<<<',
  choices: ['f','j'],
  data: {
    stimulus_type: 'congruent',
    target_direction: 'left'
  },
  on_finish: function(data){
    if(data.key_press == "f"){
      data.correct = true; // can add property correct by modify data object directly
    } else {
      data.correct = false;
    }
  }
}
```

## The on_load event

The `on_load` callback can be added to any trial. The callback will trigger once the trial has completed loading. For most plugins, this will occur once the display has been initially updated but before any user interactions or timed events (e.g., animations) have occurred.

#### Sample use
```javascript
var trial = {
  type: 'image-keyboard-response',
  stimulus: 'imgA.png',
  on_load: function() {
    console.log('The trial just finished loading.');
  }
};
```

## Dynamic parameters

Most trial parameters can be functions. In a typical declaration of a jsPsych trial, parameters have to be known at the start of the experiment. This makes it impossible to alter the content of the trial based on the outcome of previous trials. When functions are used as the parameter value, the function is evaluated at the start of the trial, and the return value of the function is used as the parameter for that trial. This enables dynamic updating of the parameter based on data that a subject has generated or any other information that you may not know in advance.

Here is a sketch of how this functionality could be used to display feedback to a subject in the Flanker Task.

```javascript

var timeline = [];

var trial = {
  type: 'html-keyboard-response',
  stimulus: '<<<<<',
  choices: ['f','j'],
  data: {
    stimulus_type: 'congruent',
    target_direction: 'left'
  },
  on_finish: function(data){
    if(data.key_press == "f"){
      data.correct = true; // can add the property "correct" to the trial data by modifying the data object directly
    } else {
      data.correct = false;
    }
  }
}

var feedback = {
  type: 'html-keyboard-response',
  stimulus: function(){
    // determine what the stimulus should be for this trial based on the accuracy of the last response
    var last_trial_correct = jsPsych.data.get().last(1).values()[0].correct;
    if(last_trial_correct){
      return "<p>Correct!</p>";
    } else {
      return "<p>Wrong.</p>"
    }
  }
}

timeline.push(trial, feedback);

```

The trial's `data` parameter can be also function, which is useful for when you want to save information to the data that can change during the experiment. For example, if you have a global variable called `current_difficulty` that tracks the difficulty level in an adaptive task, you can save the current value of this variable to the trial data like this:

```js
var current_difficulty; // value changes during the experiment

var trial = {
  type: 'survey-text',
  questions: [{prompt: "Please enter your response."}]
  data: function() { 
    return {difficulty: current_difficulty}; 
  }
}
```

It's also possible to use a function for just a single parameter in the trial's `data` object, for instance if you want to combine static and dynamic information in the data:

```js
var trial = {
  type: 'survey-text',
  questions: [{prompt: "Please enter your response."}]
  data: {
    difficulty: function() { 
      return current_difficulty; // this value changes during the experiment
    },
    task_part: 'recall', // this part of the trial data is always the same
    block_number: 1
  }
}
```

Dyanmic parameters work the same way with nested parameters, which are parameters that contain one or more sets of other parameters. For instance, many survey-* plugins have a `questions` parameter that is a nested parameter: it is an array that contains the parameters for one or more questions on the page. To make the `questions` parameter dynamic, you can use a function that returns the array with all of the parameters for each question:

```js
var subject_id; // value is set during the experiment

var trial = {
  type: 'survey-text',
  questions: function(){
    return [ {prompt: "Hi "+subject_id+"! What's your favorite city?", required: true, name: 'fav_city'} ];
  }
}
```

You can also use a function for any of the individual parameters inside of a nested parameter.

```js
var trial = {
  type: 'survey-text',
  questions: [
    { 
      prompt: function() {  
        // this question prompt is dynamic - 
        // the text that is shown will change based on the participant's earlier response
        var favorite_city = JSON.parse(jsPsych.data.getLastTrialData().values()[0].responses).fav_city;
        var text = "Earlier you said your favorite city is "+favorite_city+". What do you like most about "+favorite_city+"?"
        return text;
      }, 
      required: true,
      rows: 40,
      columns: 10
    },
    {
      prompt: 'This is the second question the page. This one is not dynamic.' 
    }
  ]
}
```

Note that if the plugin *expects* the value of a given parameter to be a function, then this function *will not* be evaluated at the start of the trial. This is because some plugins allow the researcher to specify functions that should be called at some point during the trial. Some examples of this include the `stimulus` parameter in the canvas-* plugins, the `mistake_fn` parameter in the cloze plugin, and the `stim_function` parameter in the reconstruction plugin. If you want to check whether this is the case for a particular plugin and parameter, then the parameter's `type` in the `plugin.info` section of the plugin file. If the parameter type is `jsPsych.plugins.parameterType.FUNCTION`, then this parameter must be a function and it will not be executed before the trial starts. 