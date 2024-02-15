# Home Assistant Add-on: vosk

## Installation

Follow these steps to get the add-on installed on your system:

1. Navigate in your Home Assistant frontend to **Settings** -> **Add-ons** -> **Add-on store**.
2. Add the store https://github.com/rhasspy/hassio-addons
2. Find the "vosk" add-on and click it.
3. Click on the "INSTALL" button.

## How to use

After this add-on is installed and running, it will be automatically discovered
by the Wyoming integration in Home Assistant. To finish the setup,
click the following my button:

[![Open your Home Assistant instance and start setting up a new integration.](https://my.home-assistant.io/badges/config_flow_start.svg)](https://my.home-assistant.io/redirect/config_flow_start/?domain=wyoming)

Alternatively, you can install the Wyoming integration manually, see the
[Wyoming integration documentation](https://www.home-assistant.io/integrations/wyoming/)
for more information.

## Models

Models are automatically downloaded from [HuggingFace](https://huggingface.co/rhasspy/vosk-models), but they are originally from [Alpha Cephei](https://alphacephei.com/vosk/models). Please review the license of each model that you use ([model list](https://github.com/rhasspy/wyoming-vosk/blob/master/wyoming_vosk/download.py)).

## Modes

There are three operating modes:

1. Open-ended - any sentence can be spoken, but recognition is very poor compared to [Whisper](https://github.com/rhasspy/wyoming-faster-whisper)
2. Corrected - sentences similar to [templates](#sentence-templates) are forced to match
3. Limited -  only sentences from [templates](#sentence-templates) can be spoken


### Open-ended

This is the default mode: transcripts from [vosk](https://alphacephei.com/vosk) are used directly.

Recognition is very poor compared to [Whisper](https://github.com/rhasspy/wyoming-faster-whisper) unless you use one of the [larger models](https://alphacephei.com/vosk/models).
To use a specific model, such as `vosk-model-en-us-0.21` (1.6GB):

1. Create a directory in `/share/vosk/models` with the name of the model's language (e.g., `en`)
2. Download and extract the model
3. Copy the contents of the directory named after the model into `/share/vosk/models/<LANGUAGE>`

In the English example, all of the files **inside** the extracted `vosk-model-en-us-0.21` directory will be put into `/share/vosk/models/en`, so you would have a file named `/share/vosk/models/en/am/final.mdl`.


### Corrected

By specifying which sentences will be spoken ahead of time, transcripts from vosk can be corrected using [rapidfuzz](https://github.com/maxbachmann/RapidFuzz).

Create your [sentence templates](#sentence-templates) and save them to a file named `/share/vosk/sentences/<LANGUAGE>.yaml` where `<LANGUAGE>` is one of the [supported language codes](#supported-languages). For example, English sentences should be saved in `/share/vosk/sentences/en.yaml`.

You may adjust the `correct_sentences` config value to:

* 0 - force transcript to be one of the template sentences
* greater than 0 - allow more sentences that are not similar to templates to pass through

When `correct_sentences` is large, speech recognition is effectively open-ended again. Experiment with different values to find one that lets you speak sentences outside your templates without sacrificing accuracy too much.

If you have a set of sentences with a specific pattern that you'd like to skip correction, add them to your [no-correct patterns](#no-correct-patterns).


### Limited

Follow the instructions for [corrected mode](#corrected) to create your sentence templates, then enable the `limit_sentences` config option.

This will tell vosk that **only** the sentences from you templates can ever be spoken. Sentence correction is still needed (due to how vosk works internally), but it will ensure that sentences outside the templates cannot be sent.

This mode will get you the highest possible accuracy, with the trade-off being that you cannot speak sentences outside the templates.


## Sentence Templates

Each language may have a YAML file with [sentence templates](https://github.com/home-assistant/hassil#sentence-templates).
Most syntax is supported, including:

* Optional words, surrounded with `[square brackets]`
* Alternative words, `(surrounded|with|parens)`
* Lists of values, referenced by `{name}`
* Expansion rules, inserted by `<name>`

The general format of a language's YAML file is:

``` yaml
sentences:
  - this is a plain sentence
  - this is a sentence with a {list} and a <rule>
lists:
  list:
    values:
      - value 1
      - value 2
expansion_rules:
  rule: body of the rule
```

Sentences have a special `in/out` form as well, which lets you say one thing (`in`) but put something else in the transcript (`out`).

For example:

``` yaml
sentences:
  - in: lou mo ss  # lumos
    out: turn on all the lights
  - in: knocks   # nox
    out: turn off all the lights
```

lets you say "lumos" to send "turn on all the lights", and "nox" to send "turn off all the lights".
Notice that we used words that sound like "lumos" and "nox" because [the vocabulary](https://huggingface.co/rhasspy/vosk-models/tree/main/_vocab) of the default English model is limited (`vosk-model-small-en-us-0.15`).

The `in` key can also take a list of sentences, all of them outputting the same `out` string.

### Lists

Lists are useful when you many possible words/phrases in a sentence.

For example:

``` yaml
sentences:
  - set light to {color}
lists:
  color:
    values:
      - red
      - green
      - blue
      - orange
      - yellow
      - purple
```

lets you set a light to one of six colors.

This could also be written as `set light to (red|green|blue|orange|yellow|purple)`, but the list is more manageable and can be shared between sentences.

List values have a special `in/out` form that lets you say one thing (`in`) but put something else in the transcript (`out`).

For example:

``` yaml
sentences:
  - turn (on|off) {device}
lists:
  device:
    values:
      - in: tv
        out: living room tv
      - in: light
        out: bedroom room light
```

lets you say "turn on tv" to turn on the living room TV, and "turn off light" to turn off the bedroom light.

### Expansion Rules

Repeated parts of a sentence template can be abstracted into an expansion rule.

For example:

``` yaml
sentences:
  - turn on <the> light
  - turn off <the> light
expansion_rules:
  the: [the|my]
```

lets you say "turn on light" or "turn off my light" without having to repeat the optional part.

## No Correct Patterns

When you [correct sentences](#correct-sentences), you want to keep the score cutoff as low as possible to avoid letting invalid sentences though. But what if you just want *some* open-ended sentences, such as "draw me a picture of ..." which you can then forward to an image generator?

Add the following to your sentences YAML file:

``` yaml
sentences:
  ...
no_correct_patterns:
  - <regular expression>
  - <regular expression>
  ...
```

You can add as many regular expressions to `no_correct_patterns` as you'd like. If the transcript matches any of these patterns, it will be sent with no further corrections. This effectively lets you "punch holes" in the sentence templates to allow some sentences through.

## Allow Unknown

With `--allow-unknown`, you can enable the detection of "unknown" words/phrases outside of the model's vocabulary. Transcripts that are "unknown" will be set to empty strings, indicating that nothing was recognized. When combined with [limited sentences](#limited), this lets you differentiate between in and out of domain sentences.

## Configuration

### Option: `correct_sentences`

Strictness when correcting sentences, where 0 is the most strict and larger values get less strict. This is only used when a YAML file exists for the model's language at `/share/vosk/sentences/<LANGUAGE>.yaml`.

### Option: `limit_sentences`

When enabled, only sentences from the file `/share/vosk/sentences/<LANGUAGE>.yaml` can be spoken.

### Option: `allow_unknown`

When enabled with `limit_sentences`, sentences that are not part of the templates (or "no correct" patterns) will be returned as empty strings.

### Option: `preload_language`

Preloads the speech-to-text model for the selected language. Other models are loaded as requested.

### Option: `debug_logging`

Enable debug logging. Useful for seeing satellite connections and each wake word detection in the logs.

## Support

Got questions?

You have several options to get them answered:

- The [Home Assistant Discord Chat Server][discord].
- The Home Assistant [Community Forum][forum].
- Join the [Reddit subreddit][reddit] in [/r/homeassistant][reddit]

In case you've found an bug, please [open an issue on our GitHub][issue].

[discord]: https://discord.gg/c5DvZ4e
[forum]: https://community.home-assistant.io
[issue]: https://github.com/home-assistant/addons/issues
[reddit]: https://reddit.com/r/homeassistant
[repository]: https://github.com/hassio-addons/repository
