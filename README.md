# Getting started

Run the container with:

```
docker run --env-file license.env -v ./your-config.json:/home/config.json --net host -it mdrogalis/shadowtraffic:latest --config /home/config.json
```

Useful switches for development:

- `--sample`: only produce N events and stop
- `--watch`: restart every time the configuration file has changed
- `--stdout`: ignore connections and send all events to standard out


## Configuration file

The file supplied with `--config` needs two keys: `generators` and `connections`. You can also supply `globalConfigs`.

The error messages should mostly be good enough to figure out what these data structures should look like.

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

Generates decimal numbers, presumably from a statistical generator.

```json
{
    "_gen": "number",
    "n": { "_gen": "uniformDistribution", "bounds": [1, 5] }
}
```

### Integer generator

Generates whole integers, presumably from a statistical generator.

```json
{
    "_gen": "number",
    "n": { "_gen": "normalDistribution", "mean": 20, "sd": 2 }
}
```

### SequentialInteger generator

Generates integers sequentially, starting by default from 0.

```json
{
    "_gen": "sequentialInteger",
}
```

#### Variants

```json
{
    "_gen": "sequentialInteger",
    "startingFrom": 42
}
```

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

### Date today generator

```json
{
    "_gen_": "dateToday"
}
```

- All dates are expressed in `yyyy-MM-dd` format.

### Date Between generator

```json
{
    "_gen_": "dateBetween",
    "between": ["2023-06-01", "2023-12-25"]
}
```

- Generated date is inclusive of both range rounds.

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

- Histogram for selecting a value over a continuous interval, useful for controlling lookup frequency.
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

- Explicitly specify the connection to look data up from. Useful if you're generating data for Kafka, but looking up data from Postgres.

```json
{
    "_gen": "lookup",
    "connection": "postgres",
    "table": "users",
    "column": "id"
}
```

- Use a `histogram` to control how the element is selected from the population.

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

- Sometimes make a new key, sometimes use a previously generated one.

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

### Var generator

Look up a variable defined in the top-level vars section.

```json
{
    "topic": "users",
    "vars": {
        "id": { "_gen" "uuid" }
    },
    "value": {
        "userId": {
            "_gen": "var",
            "var": "id"
        }
    }
}
```

## Transformation generators

### Merge generator

Merge a set of objects into one. Most useful combined with the `previousEvent` function to model an old event with some new attributes.

```json
{
    "_gen": "merge",
    "objects": [
        {
            "_gen": "previousEvent",
            "path": [ "value", "payload", "after" ]
        },
        {
            "_gen": "someKeys",
            "object": {
                "firstName": { "_gen": "string", "expr": "#{Name.first_name}" },
                "lastName": { "_gen": "string", "expr": "#{Name.last_name}" },
            }
        }
    ]
}
```

### SomeKeys generator

Selects one or more keys from an object at random. Most useful combined with `previousEvent` to model an old event with some randomly added changes.

```json
{
    "_gen": "merge",
    "objects": [
        {
            "_gen": "previousEvent",
            "path": [ "value", "payload", "after" ]
        },
        {
            "_gen": "someKeys",
            "object": {
                "firstName": { "_gen": "string", "expr": "#{Name.first_name}" },
                "lastName": { "_gen": "string", "expr": "#{Name.last_name}" },
            }
        }
    ]
}
```

## Transition generators

### State Machine generator

Generates data according to a set of states and transitions. Runs a state machine instance per unique `for` value, which is a path to value in the generated object (most usefully some identifier).

```json
{
    "_gen": "stateMachine",
    "initial": "a",
    "for": [ "key" ]
    "transitions": {
        "a": "b",
        "b": "c",
        "c": "a"
    },
    "states": {
        "a": 1,
        "b": 2,
        "c": 3
    }
}
```

#### Variants

An optional top-level state machine may be defined for the generator. When present, the value of it's state is merged into the _entire_ generator output. This is useful when modeling change that affects many attributes, even including the output collection.

```json
{
    "topic": "customers",
    "key": {
        "_gen": "number",
        "n": {
            "_gen": "uniformDistribution",
            "bounds": [1, 10]
        }
    },
    "value": {},
    "stateMachine": {
        "_gen": "stateMachine",
        "for": [ "key" ],
        "initial": "a",
        "transitions": {
            "a": "b",
            "b": "c",
            "c": "a"
        },
        "states": {
            "a": {
                "value": { "action": "start" }
            },
            "b": {
                "value": { "action": "running" }
            },
            "c": {
                "value": { "action": "stopped" }
            },
        }
    }
}
```

### PreviousEvent generator

Obtains the last event generated for this state machine's `for` attribute. Can optionally drill into a subvalue using `path`.

This generator may only be used within a state machine.

```json
{
    "_gen": "previousEvent",
    "path": [ "value", "payload", "after" ]
}
```

# Configuration

## Connection configs

Connections are name -> connection details. Use the name in your generators when explicitly specifying a connection. This is useful so you can connect to multiple clusters of Postgres, or refer to a Postgres table name that is the same as a Kafka topic name.

Example:
```
"connections": {
        "pg": {
            "kind": "postgres",
            "connectionConfigs": {
                "host": "localhost",
                "port": 5432,
                "username": "postgres",
                "password": "postgres",
                "db": "mydb"
            }
        }
    }
```

### Postgres

Required:
- `connectionConfigs`, with `host`, `port`, `db`.
- Optionally also with `username` and `password`

### Kafka

Required:
- `producerConfigs`, with `bootstrap.servers`.
- Optionally also with `key.serializer`, `value.serializer`

### Webhook

Required:
- `httpConfigs`, with `url`
- `dataShape`, one of `kafka`, `postgres`, to indicate whether you're day is defined in key/value or row.

## Scalar configs

### Null rates

Applied to a generator.

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

Kafka-specific. Replace an entire value, usually a composite, with null. 

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
