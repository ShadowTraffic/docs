# Getting started

Run the container with:

```
docker run --env-file license.env -v ./your-config.json:/home/config.json --net host -it mdrogalis/shadowtraffic:latest --config /home/config.json
```

Useful switches for development:

- `--sample`: only produce N events and stop
- `--watch`: restart every time the configuration file has changed
- `--stdout`: ignore connections and send all events to standard out

# Generators
## Scalar generators

### String generator

```json
{
    "_gen": "string",
    "expr": "#{Name.full_name}"
}
```

- Expr takes Java Faker commands through `#{}` templating

### Number generator

```json
{
    "_gen": "number",
    "between": [5, 200],
    "decimal": true
}
```

- `between` generates between a range, inclusive both ends
- `decimal` generates a floating point number
- `min` sets an inclusive floor
- `max` sets an inclusive ceiling
### Boolean generator

```json
{
    "_gen": "boolean"
}
```

- Equivalent to `oneOf` with choices `true` and `false`.

### UUID generator

```json
{
    "_gen": "uuid"
}
```

### Date generator

```json
{
    "_gen_": "date",
    "between": ["2023-06-01", "2023-12-25"]
}
```

- All dates are expressed in `yyyy-MM-dd` format.
- `between` generates a date between a range, inclusive.
#### Variants

```json
{
    "_gen_": "date",
    "after": "2023-06-01",
    "max": "2023-12-01"
}
```

- `max` sets an inclusive ceiling.
- `afterOrEqualTo` is also supported.

```json
{
    "_gen_": "date",
    "before": "2023-06-01",
    "min": "2023-01-01"
}
```

- `min` sets an inclusive ceiling.
- `beforeOrEqualTo` is also supported.

### Duration generator

```json
{
    "_gen_": "duration",
    "between": [1000, 4000]
}
```

All durations are expressed in milliseconds.

#### Variants
- Same as `date`, supporting `after`, `before`, `min`, `max`, plus stat dists.

### Datetime generator

?

Support for `now`.

## Statistical generators

### oneOf

```json
{
    "_gen": "oneOf",
    "choices": [
        "a",
        "b",
        "c"
    ]
}
```

- Randomly selects a choice with equal probability.

#### Variants

```json
{
    "_gen": "oneOf",
    "choices": [
        { "_gen": "string", "expr": "#{Name.full_name}" },
        { "_gen": "number", "between": [ 0, 5 ] },
        { "_gen": "boolean" }
    ]
}
```

- Each choice can be another invocation to a generator.

### weightedOneOf

```json
{
    "_gen": "weightedOneOf",
    "choices": [
        { "weight": 2, "value": "hello" },
        { "weight": 8, "value": "world" }
    ]
}
```

- Randomly selects a choice using the given probabilities.

#### Variants

```json
{
    "_gen": "weightedOneOf",
    "choices": [
	{ "weight": 2,
	  "value": {
	      "_gen": "string", "expr": "#{Name.full_name}"
	  }
	},
	{ "weight": 8, "value": "world" }
    ]
}
```

- Each choice can be another invocation to a generator.

### Normal distribution

```json
{
    "_gen": "normalDistribution",
    "mean": 100,
    "sd": 20
}
```

### Uniform distribution

```json
{
    "_gen": "uniformDistribution",
    "bounds": [1, 10]
}
```

- `bounds` are inclusive.

### Degenerate distribution

```json
{
    "_gen": "degenerateDistribution",
    "value": 42
}
```

- Always returns the value.
### Histogram generator

```json
{
    "_gen": "histogram",
    "bins": [
        { "bin": 0.2, "frequency": 8 },
        { "bin": 0.8, "frequency": 2 }
    ]
}
```

- Histogram for selecting a value over a continuous interval, most useful for controlling lookup frequency.
- 20% of the population do 80% of the action.
- Like a weighted choice, but for continuous data, not discrete.


## Query generators

### Lookup generator

```json
{
    "_gen": "lookup",
    "topic": "users",
    "path": ["key", "id"]
}
```

#### Variants

```json
{
    "_gen": "lookup",
    "connection": "postgres",
    "table": "users",
    "column": "id"
}
```

```json
{
    "_gen": "lookup",
    "topic": "users",
    "path": ["key", "id"],
    "histogram": {
	"_gen": "histogram",
        "bins": [
            { "bin": 0.2, "frequency": 80 },
	    { "bin": 0.8, "frequency": 20 }
        ]
    }
}
```

- Use a `histogram` to control how the element is selected from the population.

```json
{
    "topic": "users",
    "key": {
	"_gen": "oneOf",
	"choices": [
	    {
		"weight": 8,
		"value": {
		    "_gen": "string",
		    "expr": "#{Internet.uuid}"
		}
	    },
	    {
		"weight": 2,
		"value": {
		    "_gen": "lookup",
		    "topic": "users",
		    "path": [ "key" ]
		}
	    }
	]
    }
}
```

- Sometimes make a new key, sometimes use a previously generated one.

## Transition generators

### State machine generator

```json
{
    "topic": "userActions",
    "key": { "_gen": "string", "expr": "#{Internet.uuid}" },
    "value": {
	"action": {
	    "_gen": "stateMachine",
	    "for": [ "key" ],
	    "initial": "login",
	    "transitions": {
		"login": ["sendMessage", "sendEmoji"],
		"sendMessage": ["sendEmoji", "logout"],
		"sendEmoji": ["sendMessage", "logout"]
	    }
	}
    }
}
```

- Generates varied actions for each user ID according the state machine.
- By default, the transition names are the state names, so this example generates the strings `login`, `sendMessage`, etc.

#### Variants

```json
{
    "topic": "userActions",
    "key": { "_gen": "string", "expr": "#{Internet.uuid}" },
    "value": {
	"actionCode": {
	    "_gen": "stateMachine",
	    "for": [ "key" ],
	    "initial": "login",
	    "transitions": {
		"login": ["sendMessage", "sendEmoji"],
		"sendMessage": ["sendEmoji", "logout"],
		"sendEmoji": ["sendMessage", "logout"]
	    },
	    "states": {
		"login": { "_gen": "number", "between": [1, 3] },
		"sendMessage": { "_gen": "number", "between": [4, 6] },
		"sendEmoji": { "_gen": "number", "between": [7, 8] },
		"logout": { "_gen": "number", "between": [9, 10] }
	    }
	}
    }
}
```

- States can  be constants or generators.

```json
{
    "topic": "userActions",
    "key": { "_gen": "string", "expr": "#{Internet.uuid}" },
    "value": {
	"actionCode": {
	    "_gen": "stateMachine",
	    "for": [ "key" ],
	    "initial": "login",
	    "transitions": {
		"login": {
		    "_gen": "weightedOneOf",
		    "choices": [
			{ "weight": 2, "value": "sendMessage" },
			{ "weight": 8, "value": "sendEmoji" },
		    ]
		},
		"sendMessage": {
		    "_gen": "weightedOneOf",
		    "choices": [
			{ "weight": 4, "value": "sendEmoji" },
			{ "weight": 6, "value": "logout" },
		    ]
		},
		"sendEmoji": {
		    "_gen": "weightedOneOf",
		    "choices": [
			{ "weight": 7, "value": "sendMessage" },
			{ "weight": 3, "value": "logout" },
		    ]
		}
	    },
	    "states": {
		"login": { "_gen": "number", "between": [1, 3] },
		"sendMessage": { "_gen": "number", "between": [4, 6] },
		"sendEmoji": { "_gen": "number", "between": [7, 8] },
		"logout": { "_gen": "number", "between": [9, 10] }
	    }
	}
    }
}
```

- Transitions can be weighted choices.

```json
{
    "topic": "userActions",
    "key": {
	"_gen": "lookup",
	"topic": "users",
	"path": ["key", "id"]
    },
    "value": {
	"actionId": {
	    "_gen": "string",
	    "expr": "#{Internet.uuid}"
	}
    },
    "stateMachines": [
	{
	    "_gen": "stateMachine",
	    "for": [ "key" ],
	    "initial": "login",
	    "transitions": {
		"login": ["sendMessage", "sendEmoji"],
		"sendMessage": ["sendEmoji", "logout"],
		"sendEmoji": ["sendMessage", "logout"]
	    },
	    "states": {
		"login": {
		    "value": {
			"action": "login",
		    }
		},
		"sendMessage": {
		    "value": {
			"action": "sendMessage",
			"message": {
			    "_gen": "string",
			    "expr": "#{Team.name}"
			}
		    }
		},
		"sendEmoji": {
		    "value": {
			"action": "sendEmoji",
			"message": {
			    "_gen": "oneOf",
			    "choices": [
				"👍", "👋", "🙂"
			    ]
			}
		    }
		},
		"logout": {
		    "value": {
			"action": "logout"
		    }
		}
	    }
	}
    ]
}
```

- Top-level state machines merge their state values into the final event, in the order they are declared.

# Configuration

## Scalar configs

### Null rates

Applied to particular generator types.

```json
{
    "_gen": "string",
    "expr": "#{Name.full_name}",
    "null": {
	"rate": 0.5
    }
}
```

#### Variants

```json
{
    "_gen": "string",
    "expr": "#{Name.full_name}",
    "null": {
	"rate": {
	    "_gen": "histogram",
	    "bins": [
		{ "bin": 0.2, "frequency": 8 },
		{ "bin": 0.8, "frequency": 2 }
	    ]
	}
    }
}
```

## Local configs

Applied at the generator level.

### Throttling

```json
{
    "topic": "users",
    "key": { "_gen": "string", "expr": "#{Internet.uuid}" },
    "localConfigs": {
	"throttle": {
	    "ms": 200
	}
    }
}
```

#### Variants
```json
{
    "topic": "users",
    "key": { "_gen": "string", "expr": "#{Internet.uuid}" },
    "localConfigs": {
	"throttle": {
	    "ms": {
		"_gen": "normalDistribution",
		"mean": 100,
		"sd": 20
	    }
	}
    }
}
```


```json
{
    "topic": "users",
    "key": { "_gen": "string", "expr": "#{Internet.uuid}" },
    "localConfigs": {
	"throttle": {
	    "ms": {
		"_gen": "oneOf",
		"choices": [5, 1000]
	    }
	}
    }
}
```

### Null values

Replace an entire value, usually a composite, with null.

```json
{
    "topic": "users",
    "key": { "_gen": "string", "expr": "#{Internet.uuid}" },
    "value": {
	"name": { "_gen": "string", "expr": "#{Name.full_name}" }
    },
    "localConfigs": {
	"value": {
	    "null": {
		"rate": 0.05
	    }
	}
    }
}
```

- Equivalent to a Kafka tombstone 5% of the time.
#### Variants

```json
{
    "topic": "users",
    "key": { "_gen": "string", "expr": "#{Internet.uuid}" },
    "value": {
	"name": { "_gen": "string", "expr": "#{Name.full_name}" }
    },
    "localConfigs": {
	"value": {
	    "null": {
		"rate": {
		    "_gen": "normalDistribution",
		    "mean": 0.05,
		    "sd": 0.01
		}
	    }
	}
    }
}
```