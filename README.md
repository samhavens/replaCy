# replaCy

We found that in multiple projects we had duplicate code for using spaCy’s blazing fast matcher to do the same thing: Match-Replace-Grammaticalize. So we wrote replaCy!

* Match - spaCy’s matcher is great, and lets you match on text, shape, POS, dependency parse, and other features. We extended this with “match hooks”,  predicates that get used in the callback function to further refine a match.
* Replace - Not built into spaCy’s matcher syntax, but easily added. You often want to replace a matched word with some other term.
* Grammaticalize - If you match on ”LEMMA”: “dance”, and replace with suggestions: ["sing"], but the actual match is danced, you need to conjugate “sing” appropriately. This is the “killer feature” of replaCy

## Requirements

* `spacy >= 2.0` (not installed by default, but replaCy needs to be instantiated with an `nlp` object)

## Installation

`pip install replacy`

## Quick start

```python
from replacy import ReplaceMatcher
from replacy.db import load_json
import spacy


match_dict = load_json('/path/to/your/match/dict.json')
# load nlp spacy model of your choice
nlp = spacy.load("en_core_web_sm")

rmatcher = ReplaceMatcher(nlp)

# get inflected suggestions
# look up the first suggestion
span = rmatcher("She extracts revenge.")[0]
span._.suggestions
# >>> ['exacts']
```

## match_dict.json format

Here is a minimal `match_dict.json`:

```json
{
    "extract-revenge": {
        "patterns": [
            {
                "LEMMA": "extract",
                "TEMPLATE_ID": 1
            }
        ],
        "suggestions": [
            [
                {
                    "TEXT": "exact",
                    "FROM_TEMPLATE_ID": 1
                }
            ]
        ],
        "match_hook": [
            {
                "name": "succeeded_by_phrase",
                "args": "revenge",
                "match_if_predicate_is": true
            }
        ],
        "test": {
            "positive": [
                "And at the same time extract revenge on the whites he so despises?",
                "Watch as Tampa Bay extracts revenge against his former Los Angeles Rams team."
            ],
            "negative": [
                "Mother flavours her custards with lemon extract."
            ]
        }
    }
}
```

* The top-level key, `extract-revenge` must be unique (as must any dictionary key). The name is used as a unique identifier, but never shown.

* The inner keys are as follows
  * `patterns` - A list of [spaCy Matcher patterns](https://spacy.io/usage/rule-based-matching#matcher) (actually, a superset of a spaCy matcher pattern), which may look like e.g. `[{"LOWER": "hello"}, {"IS_PUNCT": True}, {"LOWER": "world"}]`. The added syntax which makes it a superset is being able to add `"TEMPLATE_ID": int` to some of the dicts. This labels that part of the match as a template to be inflected, such as a verb to conjugate or a noun to pluralize. In the above example, we label the lemma `extract` as having `TEMPLATE_ID` of `1`.
  * `suggestions` - a list of lists of dicts. The dicts have 1-2 keys: either just `TEXT`, which will be used in the suggestion, or `"TEXT": "sometext"` and `"FROM_TEMPLATE_ID": int`, which will apply the conjugation/pluralization of the `TEMPLATE_ID` with value `int` to `"TEXT"`. In the above example, suggestions is `[[{"TEXT":"exact","FROM_TEMPLATE_ID":1}]]`, which means we will match the conjugation of `exact` to the conjugation of `extracts`, from the step above.
  * `match_hook` - (despite the singular name) A list of "match hooks". These are Python functions which refine matches. See the following section.
  * `test` - has `positive` and `negative` keys. `positive` is a list of strings which this rule SHOULD match against, `negative` is a list of strings which SHOULD NOT match. Used for testing now, but we have plans to infer rules from this section.
  * (optional) `comment` - a string for other humans to read; ignored by replaCy
  * (optional) `anything` - you can add any extra structure here, and replaCy will attempt to tag matching spans with this information using the spaCy custom namespace `span._`. For example, you can add the key `oogly` with value `"boogly"` for the match `"LOWER": "secret password"`. Then if you call `span = rmatcher("This is the secret password.")[0]`, then `span._.oogly == "boogly"`.

Between match hooks and custom properties, replaCy is incredibly powerful, and allows you to control your NLP application's behavior from a single JSON file.

### Match hooks

Match hooks are powerful and somewhat confusing. replaCy provides a starting kit of hooks, but since they are just Python functions, you can supply your own. To see all the built in hooks, see [custom_patterns.py](https://github.com/Qordobacode/replaCy/blob/master/replacy/custom_patterns.py). An example is `preceded_by_pos`, which is copied here in full. Notice the signature of the function; if this interests you, see the next subsection, "Hooks Return Predicates".

```python
SpacyMatchPredicate = Callable[[Doc, int, int], bool]

def preceded_by_pos(pos) -> SpacyMatchPredicate:
    if isinstance(pos, list):
        pos_list = pos

        def _preceded_by_pos(doc, start, end):
            bools = [doc[start - 1].pos_ == p for p in pos_list]
            return any(bools)

        return _preceded_by_pos
    elif isinstance(pos, str):
        return lambda doc, start, end: doc[start - 1].pos_ == pos
    else:
        raise ValueError(
            "args of preceded_by_pos should be a string or list of strings"
        )
```

This allows us to put in our `match_dict.json` a hook that effectively says "only do this spaCy match is the preceding POS tag is `pos`, where `pos` is either a string, like `"NOUN"`, or a list such as `["NOUN", "PROPN"]`. Here is the most complicated replaCy match I have written, which demonstrates the use of many hooks:

```json
{
    "require": {
        "patterns": [
            {
                "LEMMA": "require",
                "POS": "VERB",
                "DEP": {
                    "NOT_IN": [
                        "amod"
                    ]
                },
                "TEMPLATE_ID": 1
            }
        ],
        "suggestions": [
            [
                {
                    "TEXT": "need",
                    "FROM_TEMPLATE_ID": 1
                }
            ]
        ],
        "match_hook": [
            {
                "name": "succeeded_by_phrase",
                "args": "that",
                "match_if_predicate_is": false
            },
            {
                "name": "succeeded_by_phrase",
                "args": "of",
                "match_if_predicate_is": false
            },
            {
                "name": "preceded_by_dep",
                "args": "auxpass",
                "match_if_predicate_is": false
            },
            {
                "name": "relative_x_is_y",
                "args": [
                    "children",
                    "dep",
                    "csubj"
                ],
                "match_if_predicate_is": false
            }
        ],
        "test": {
            "positive": [
                "Those require more consideration.",
                "Your condition is serious and requires surgery.",
                "I require stimulants to function."
            ],
            "negative": [
                "My pride requires of me that I tell you to piss off.",
                "Is there any required reading?",
                "I am required to tell you that I am a registered Mex offender - I make horrible nachos.",
                "Deciphering the code requires an expert.",
                "Making small models requires manual skill."
            ]
        },
        "comment": "The pattern includes DEP NOT_IN amod because of expresssions like 'required reading' and the relative_x_is_y hook is because this doesn't work for clausal subjects"
    }
}
```

#### Hooks Return Predicates

To be a match hook, a Python function must take 1 or 0 arguments, and return a predicate (function which returns a boolean) with inputs `(doc, start, end)`. If you read about the spaCy Matcher, you will understand why the arguments are `doc, start, end`. The reason match hooks RETURN a predicate, rather than BEING a predicate is for flexibility.

The structure of a match hook is:

* `name`, the name of the Python function
* (optional) `args` - the argument of the function. Yes, argument, singular - match hooks take one or zero arguments. If you need more than one argument, have the hook accept a dict or list.
* `match_if_predicate_is` - a boolean which flips the behavior from "if this predicate is true, then match" or "if this predicate is false, then match". This is just to make naming functions easier. For example, we have `preceded_by_pos` as a hook, with `arg: "NOUN"`, and `match_if_predicate_is` set to `true`. This hook is much more sensible than `not_preceded_by_pos`, with args `[every, pos, but, NOUN]`.

To use your own match hooks, instantiate the replace matcher with a module containing them, e.g.

```python
    from replacy import ReplaceMatcher
    from replacy.db import load_json
    import spacy

    # import the module with your hooks
    import my.custom_hooks as ch


    nlp = spacy.load("en_core_web_sm")
    rmatch_dict = load_json("./resources/match_dict.json")
    # pass replaCy your custom hooks here, and then they are usable in your match_dict.json
    rmatcher = ReplaceMatcher(nlp, rmatch_dict, custom_match_hooks=ch)
    span = rmatcher("She excepts her fate.")[0]
    span._.suggestions
    # >>> ['acccepts']
```

## Testing match_dict (JSON schema validation)

```python
from replacy import ReplaceMatcher
from replacy.db import load_json

match_dict = load_json('/path/to/your/match/dict')
ReplaceMatcher.validate_match_dict(match_dict)
```
